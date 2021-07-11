---
layout: post
title: "Embedding MCollective"
date: 2011-10-25T09:32:00
comments: false
categories: ["chef"]
---

MCollective is "a framework to build server orchestration or parallel
job execution systems". It's very easy to create and deploy agents to
your hosts, and then interrogate or instruct them with simple
command-line tools. For ad hoc tasks, this is ideal.

As part of our platform engineering work though, we're integrating
MCollective with existing tools, and with our new orchestration
systems -- using MCollective as the glue between a centralised
controller and its agents on individual hosts.

In this situation, our "client" isn't a command-line tool but a larger
Rails application with a number of other dependencies. Bringing the
mcollective tools into this environment isn't straightforward, since
a number of the default options need to be changed. 

Here's my first attempt to embed an MCollective client in my
application:

```ruby
require 'mcollective'

class EmbeddedClient
  include MCollective::RPC
   
  def self.client(agent)
    client = rpcclient(agent)
  end
end
```

This has a number of problems.

I need to make the "mcollective" library code available. There's
currently no gem available, so it's not possible to just `gem
"mcollective", "1.2.0"` in the Gemfile. There's also the
MCollective plugins, which need to be in sync with the core library
version.

The configuration also needs to be available, and the default location
is /etc/mcollective/client.cfg - which is naturally outside of any
deployed application's root, and so we'd need to deploy that
separately.

Having made the code available, the next problem to deal with is
command-line option parsing. By default, MCollective will parse and
validate the process's ARGV, and raise an exception if it finds an
option it doesn't recognise. This isn't appropriate in an embedded
client, so we need to bypass option parsing. 

Here's how to create an embedded MCollective client, taking these
points into account:

```ruby
require 'mcollective'

class EmbeddedClient
  include MCollective::RPC
   
  def self.client(agent)
    options =  MCollective::Util.default_options
    options[:config] = 'config/mcollective.cfg'
    client = rpcclient(agent, {:options => options})
    client.discovery_timeout = 10
    client.timeout = 120
    client
  end
end
```

and here's the configuration that points to:

```
topicprefix = /topic/
main_collective = mcollective
collectives = mcollective
libdir = vendor/mcollective/plugins
logfile = stdout
loglevel = debug

# Plugins
securityprovider = psk
plugin.psk = yeah

connector = stomp
plugin.stomp.host = 10.101.1.16
plugin.stomp.port = 61613
plugin.stomp.user = guest
plugin.stomp.password = guest

# Facts
factsource = yaml
plugin.yaml = config/mcollective_facts.yaml
```

This works, but there's a problem - in rpcclient(), there's a call to
`exit!` on any exception during the client setup process,
including exceptions thrown by the stomp library. We need to avoid
that and handle exceptions ourselves, so:

```ruby
require 'mcollective'

class EmbeddedClient
  include MCollective::RPC

  def self.client(agent)
    options =  MCollective::Util.default_options
    options[:config] = 'config/mcollective.cfg'
    client = MCollective::RPC::Client.new(agent, :options => options)
    client.discovery_timeout = 10
    client.timeout = 120
    client
  end
end
```

Now we can handle exceptions ourselves, but there's still an issue -
the stomp library writes directly to $stderr and we should be
collecting that output and logging it in whatever way is appropriate
for the application:

```ruby
def log_stderr &block
  begin
    real_stderr, $stderr = $stderr, StringIO.new
    yield
  ensure
    $stderr, stderr = real_stderr, $stderr
    stderr.string.each_line do |line|
      log(:error, "stderr: #{line}")
    end
  end
end

# and then:
log_stderr do
  run_mcollective_processes
end
```

At this point, we've got the following problems solved:

 * MCollective's library code and plugins are within our application tree
 * The configuration file is also within the application
 * MCollective won't try to reinterpret our application's ARGV
 * We can handle client-setup exceptions ourselves
 * Stomp errors are logged, rather than being output to stderr.

which should let the application interact with MCollective in a
reliable and maintainable way.

