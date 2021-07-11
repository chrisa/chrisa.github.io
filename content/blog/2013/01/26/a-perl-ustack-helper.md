---
layout: post
title: "A Perl ustack helper?"
date: 2013-01-26
comments: false
categories: ["dtrace"]
---

I've done some work with DTrace and Perl, using my
[Devel::DTrace::Provider](https://metacpan.org/module/Devel::DTrace::Provider)
and Andy Armstrong's
[Devel::DTrace](https://metacpan.org/module/Devel::DTrace), but
there's never been a ustack helper: because of the structure of the
Perl interpreter, it's
[impossible](https://blogs.oracle.com/levon/entry/python_and_dtrace_in_build#comment-1180088715000)
to write one -- at least, in the intended manner...


## Perl and ustack helpers - the big problem ##

The primary problem with a Perl ustack helper is the lack of
correspondence between the C stack and the Perl stack: unlike Python,
V8 Javascript etc, Perl's stack is handled entirely separately, and
this is why the "standard" ustack helper will never be possible: there
just aren't the C stack frames there to annotate.

What we can try though, is to annotate a single C frame with as much
of the Perl stack as we can find by chasing pointers from that frame.

This post explains my experiments with that idea. To give the game
away early, it doesn't entirely work, but there are some techniques
here that might be useful elsewhere.

A lot of what I've done here is based directly on the existing ustack
helpers for other languages, specifically V8 and Python.


## Dynamically loading the ustack helper ##

If at all possible, I want the helper to work on an unmodified Perl
binary - if the thing works at all, I want to be able to use it like a
CPAN-style module rather than having to patch Perl. The first problem
is to get the the helper loaded. 

Given the way [libusdt](https://github.com/chrisa/libusdt) works, it
seems likely we can load a helper just like a provider, by
`ioctl()`ing the DOF down to the kernel from a Perl XS
extension. Of course, that's all DTrace's DOF-loading `_init`
routine does anyway, just we'll be doing it slightly later on in the
process's life.

Unfortunately this facility [isn't part of libusdt's public API yet](https://github.com/chrisa/libusdt/issues/6), but it's really not
that much code, especially if we're only supporting Illumos-based
systems.

Actually building the helper DOF is trivial: compile the script with
`dtrace_program_fcompile()`, and wrap it in DOF with
`dtrace_dof_create()`.

Loading the DOF containing the helper program works, and means we can
initialise the helper from an extension module, rather needing to
patch it into Perl's build process.


## Finding - or rather creating - a frame to annotate ##

Ideally we need a stack frame which is always found in the
interpreter's C stack, for which we can easily find the address of the
function, and where the stack is passed as one of the
arguments. There's no such frame in the standard Perl interpreter, but
we can manufacture one. Perl lets us replace the "runops loop" with
one of our own, and we can use this to meet all of our requirements.

The runops loop function is the core of the interpreter, responsible
for invoking the current "op", which returns the new current op. 

The usual, non-debug, runops loop looks like this (in Perl 5.16.2):

```c
int
Perl_runops_standard(pTHX)
{
    dVAR;
    register OP *op = PL_op;
    while ((PL_op = op = op->op_ppaddr(aTHX))) {
    }

    TAINT_NOT;
    return 0;
}
```

The top-level loop is always visible during execution, and we can
replace the usual function with one of our own, fulfilling our first
two requirements.

If we make this loop execute ops through another function, and pass
that function a pointer to the Perl stack, we fulfill the final
requirement. These functions are dtrace_runops and dtrace_call_op:

```c
STATIC OP *
dtrace_call_op(pTHX_ PERL_CONTEXT *stack)
{
        return CALL_FPTR(PL_op->op_ppaddr)(aTHX);
}

STATIC int
dtrace_runops(pTHX)
{
        while ( PL_op ) {
                if ( PL_op = dtrace_call_op(aTHX_ cxstack), PL_op ) {
                        PERL_ASYNC_CHECK(  );
                }
        }

        TAINT_NOT;
        return 0;
}
```	

We'll target the annotation at dtrace_call_op(), and attempt to walk
the stack starting from the PERL_CONTEXT pointer we're given.

Actually installing the alternative runops loop is a standard Perl
extension technique, and we just need to make sure it happens early
enough that the top-level loop is ours rather than the standard one.



## Frame annotations ##

The primary purpose of the ustack helper is to provide a descriptive
string for a frame where there's no corresponding C symbol - for
JITted code, say. If there is such a symbol, the ustack helper's
string will be ignored - and in this case, there is,
`dtrace_call_op`.

Fortunately there's a mechanism for adding annotations to these
frames, and that's what we'll use here: a string beginning with an
`@` will be used as an annotation. In the [Python
helper](https://blogs.oracle.com/levon/entry/python_and_dtrace_in_build),
it looks like this:

```
       libpython2.4.so.1.0`PyEval_EvalFrame+0xbdf
           [ build/proto/lib/python/mercurial/localrepo.py:1849 (addchangegroup) ]
```



## Targetting a specific frame ##

In a helper action, `arg0` is the program counter in that stack
frame. If we make the address of our inserted `dtrace_call_op`
function available to the helper, and of the preceding function, we
can compare the pc to these two addresses to determine when we're
annotating that function.

Here, `this->start` and `this->end` have been initialised to the
addresses of `dtrace_call_op` and the preceding function:

```
dtrace:helper:ustack:
/this->pc >= this->start/
{
        this->go++;
}

dtrace:helper:ustack:
/this->pc < this->end/
{
        this->go++;
}
```

For a reason I'm not entirely sure of, combining these predicates into
one doesn't work. 



## Passing values into the helper ##

With the extra control over the helper initialisation we get from
loading it "by hand", it turns out that macros work fine! We can use
this to pass values into the helper: symbol addresses and structure
member offsets. 

It doesn't seem to be possible to simply `#include <perl.h>` - the D
compiler barfs badly on Perl's headers, which are
.. involved. Fortunately we can do the necessary sizeof and offsetof
work in C and pass the results into D with macros. This should buy at
least some ability to cope with changes to Perl's data structures,
though more sweeping changes will still break things entirely.

Macros are strings, so all the values passed need to be formatted with
`sprintf`; at least this is just setup code.



## Copying a C string ##

Unless I've missed something, this is awkward. Our final stacktrace
string that the helper will return as the frame's annotation is
allocated out of D scratch space, so we need to copy C strings from
userspace into it. If we have the string's length available this is
easily done with `copyinto()`, but if we've just got a `char *`,
it's not.

Ideally we could take the string's length with `strlen()` and do a
copy -- but `strlen` [isn't available to helpers](http://src.illumos.org/source/xref/illumos-gate/usr/src/uts/common/dtrace/dtrace.c#8595).
It doesn't seem to be possible to use `strchr()` either, since it
returns `string` and not `char *`, so we can't find the length
that way.

I'm not sure if the lack of `strlen` is an oversight, or if there's
some reason that it's unsafe in arbitrary context: it seems that if
something like `strchr` is safe, `strlen` also ought to be.

We can't just copy a fixed length of data, so a character-by-character
"strncpy" is needed:

```
/* Copy a string into this->buf, at the location indicated by this->off */

#define APPEND_CHR_IF(offset, str) \
dtrace:helper:ustack:                                                      \
/this->go == 2 && !this->strdone/                                          \
{                                                                          \
    copyinto((uintptr_t)((char *)str + offset), 1, this->buf + this->off); \
    this->off++;                                                           \
}                                                                          \
dtrace:helper:ustack:                                                      \
/this->go == 2 && !this->strdone && this->buf[this->off - 1] == '\0'/      \
{                                                                          \
    this->strdone = 1;                                                     \
    this->off--;                                                           \
}

#define APPEND_CSTR(str) \
dtrace:helper:ustack:    \
/this->go == 2/          \
{                        \
    this->strdone = 0;   \
}                        \
APPEND_CHR_IF(0, str) \
APPEND_CHR_IF(1, str) \
APPEND_CHR_IF(2, str) \
...
[ up to the length of string required]
```


## Walking the stack ##

After all that, actually walking the stack from the pointer we've been
passed is relatively simple. Using the information in [Perlguts Illustrated](http://cpansearch.perl.org/src/RURBAN/illguts-0.42/index-14.html),
we walk the context stack, appending frame annotations to our string buffer. 

Obviously it's only possible to walk a limited number of frames, and
with the default size limit on helper size and the ops required for
string copies, quite a limited number of frames. 


## The output! ##

Here's an incredibly simple example of the output:

```
  0     20                      write:entry 
              libc.so.1`__write+0x15
              libperl.so`PerlIOUnix_write+0x46
              libperl.so`Perl_PerlIO_write+0x47
              libperl.so`PerlIOBuf_flush+0x50
              libperl.so`Perl_PerlIO_flush+0x45
              libperl.so`PerlIOBuf_write+0x11d
              libperl.so`Perl_PerlIO_write+0x47
              libperl.so`Perl_do_print+0xa7
              libperl.so`Perl_pp_print+0x195
              Helper.so`dtrace_call_op+0x3f
                [ 
                  t/helper/01-helper.t:
                  t/helper/01-helper.t:24
                  t/helper/01-helper.t:25
                  t/helper/01-helper.t:21
                  t/helper/01-helper.t:17
                  t/helper/01-helper.t:13
                ]
              Helper.so`dtrace_runops+0x56
              libperl.so`perl_run+0x380
              perl`main+0x15b
              perl`_start+0x83
```

This shows file:lineno pairs for each stack frame representing a
subroutine call that was found walking the context stack. 

Here's a (slightly) less trivial example, taken during a run of the
CPAN shell program:

```
  0     20                      write:entry 
              libc.so.1`__write+0x15
              libperl.so`PerlIOUnix_write+0x46
              libperl.so`Perl_PerlIO_write+0x47
              libperl.so`PerlIOBuf_flush+0x50
              libperl.so`Perl_PerlIO_flush+0x45
              libperl.so`PerlIOBuf_write+0x11d
              libperl.so`Perl_PerlIO_write+0x47
              libperl.so`Perl_do_print+0xa7
              libperl.so`Perl_pp_print+0x195
              Helper.so`dtrace_call_op+0x3f
                [ 
                  -e:
                  -e:1
                  /opt/local/lib/perl5/5.14.0/CPAN.pm:325
                  /opt/local/lib/perl5/5.14.0/CPAN.pm:325
                  /opt/local/lib/perl5/5.14.0/CPAN.pm:345
                  /opt/local/lib/perl5/5.14.0/CPAN.pm:421
                  /opt/local/lib/perl5/5.14.0/CPAN/Shell.pm:1494
                  /opt/local/lib/perl5/5.14.0/CPAN/Shell.pm:1461
                ]
              Helper.so`dtrace_runops+0x56
              libperl.so`perl_run+0x246
              perl`main+0x15b
              perl`_start+0x83
```

## The code, and its limitations ##

The code is [available on Github](https://github.com/chrisa/perl-Devel-DTrace-Helper).
I don't plan to release this module to CPAN any time soon!

For anything but the most trivial examples this code probably won't
provide useful Perl stacktraces, and it's only been tried on Perl
5.14.2 built with threads, on an Illumos-derived system.

It certainly won't work on the Mac, since [ustack helpers are disabled
there](http://mail.opensolaris.org/pipermail/dtrace-discuss/2011-March/009088.html),
and won't work without threads enabled in Perl because of an
implementation detail of Perl OPs we're exploiting that's different
without threads.

Hopefully though, this post sheds a bit of light on ustack helpers,
and maybe there are some interesting techniques here for other
situations.
