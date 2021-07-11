---
layout: post
title: "Ruby 1.9.2 and YAML"
date: 2011-12-03
comments: false
categories: ["chef"]
---

I found another gotcha when embedding MCollective in a larger
application: Ruby's YAML library situation. Historically Ruby has used
the Syck library, but since 1.9.2 libyaml is used if available,
through a set of bindings called Psych. 

While Syck is still available and may be selected at runtime, the
default is chosen at compile time, based on the presence or not of
libyaml - if it's there when you build Ruby, you'll get Psych by
default. 

To find out which binding your Ruby uses, inspect `YAML::ENGINE.yamler`:

```
ruby-1.9.2-p290 :004 > YAML::ENGINE.yamler
 => "syck" 
```

This wouldn't be a problem unless there were differences, and although
it's fair to say that Psych is more compliant with the YAML spec, it's
less tolerant of slight deviations - some of which Syck is guilty of.

Coming back to MCollective, this means that an MCollective running on
Ruby 1.9.2 with Psych won't necessarily be able to communicate with
other MCollective installations running on Ruby 1.8 with Syck. Mostly
this seems to be down to handling of whitespace in multiline YAML
values, which MCollective makes heavy use of since it nests one YAML
document in another.

The initial fix in my environment was to rebuild Ruby 1.9.2 without
libyaml, so Syck is used everywhere. Beyond that, upgrading everything
to the same Ruby 1.9.2 build appears to be the best long term answer. 

