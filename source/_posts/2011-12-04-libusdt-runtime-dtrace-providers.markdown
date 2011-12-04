---
layout: post
title: "libusdt - creating DTrace providers at runtime"
date: 2011-12-04
comments: false
categories: 
---

With this post I'd like to introduce <tt>libusdt</tt>, a library for
creating DTrace USDT providers at runtime, intended as the basis for
implementing custom DTrace providers in dynamic languages, but
applicable wherever the provider definition isn't known at compile
time.

A bit of the history of this library: in 2007 I'd been tinkering with
a Ruby extension to allow access to libdtrace as a consumer of DTrace
data, with the hope that this might make it easier to write complex
analysis scripts - like [this port to ruby](https://github.com/chrisa/ruby-dtrace/blob/master/examples/scsi.rb) of Chris Gerhard's
<tt>[scsi.d](http://blogs.oracle.com/chrisg/entry/scsi_d_script)</tt>
script.

At this point, the Ruby interpreter had been [patched by Joyent](http://joyeur.com/2007/05/07/dtrace-for-ruby-is-available/)
to include its
own USDT provider, which was accessible using the consumer
library. What wasn't possible though was to define providers in Ruby, to
create probes meaningful to the application rather than just to the
interpreter. My first attempt at this, autogenerating and building the
C source and D script required to specify a provider, was never going
to be anything other than an experiment.

[This post](http://blogs.oracle.com/kamg/entry/adding_user_defined_dtrace_probes)
by Keith McGuigan, though, showed that the JVM was capable of creating
providers at runtime without invoking the usual compile-time
machinery. It seemed that to do this using the existing USDT provider
in Solaris, it'd have to be doing a number of things:

 * Dynamically creating tracepoints in the running process
 * Dynamically creating DOF to describe the provider to the kernel
 * Loading the DOF to enable the provider.

Was the JVM doing all this? Keith's blog post didn't discuss the
low-level implementation, and there didn't seem to be a way to do it
with libdtrace...

This was just before dtrace.conf(08). I was fortunate enough to attend
and was able to speak to Keith about the JVM implementation. It turns
out that this is pretty much how it works - the JVM with its
JIT-compiler already has the capability of creating new native code at
runtime, and creating the appropriate DOF document is simply a matter
of following the extensive documentation in <tt><sys/dtrace.h></tt>.

I got this working in Ruby, and so now
[ruby-dtrace](https://github.com/chrisa/ruby-dtrace/) could both
produce and consume DTrace data, and eventually do it on all the
platforms supporting USDT providers - Solaris and Mac OS X, SPARC,
Intel and PowerPC, 32 and 64 bit (it's entirely responsible for the
obsolete computers I still keep around...). This supported providers
like [Rack::Probe](https://github.com/ecin/rack-probe).

[Devel::DTrace::Provider](https://github.com/chrisa/perl-dtrace/tree/master/Devel-DTrace-Provider)
followed for Perl, like ruby-dtrace mostly in Perl - it was clear that
the core of these extensions needed to be a standalone library. One
more similar port,
[node-dtrace-provider](https://github.com/chrisa/node-dtrace-provider),
provoked a start on <tt>libusdt</tt>.

Incidentally the Perl port was written to add probes to my employer's
large Perl web-app, for an evaluation on Solaris - we immediately
discovered unexpected behaviours, with the additional context provided
by the application's probes going some way to making up for the lack
of a Perl stack helper.

<tt>[libusdt](https://github.com/chrisa/libusdt/)</tt> is currently
very limited - right now it only supports the Mac, and then only for
64 bit processes - but my plan is to bring over all the platform
support from ruby-dtrace and then to port all the language bindings
to use it as the core of their implementation.

Once that's done, I'd like to find ways to reduce the disabled-probe
overhead - while is-enabled probes are supported and used to allow
costly argument-gathering work to be avoided where possible, the use
of generated stub functions means there's still some overhead. 

Some of this work will undoubtedly be language-specific, tying the
generated probes in more closely with the runtimes, but some may also
come from improvements in the library itself.
