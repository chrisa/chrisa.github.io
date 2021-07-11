---
layout: post
title: "Custom chef_gem resources"
date: 2012-07-05
comments: false
categories: ["chef"]
---

Since Chef 0.10.10, there's been a new core resource for installing
Ruby gems to support Chef cookbooks: chef_gem. This installs
gems into the current Ruby at compile time, and handles making the
newly-installed gems available to the running Chef client -- wrapping
up an ugly piece of code found in many cookbooks:

```ruby
gem_package "ruby-shadow" do
  action :nothing
end.run_action :install

Gem.clear_paths
```

into a single resource declaration:

```ruby
chef_gem "ruby-shadow"
```

We hadn't considered switching to this resource though, despite a
compatibility cookbook being <a
href="http://community.opscode.com/cookbooks/chef_gem">available</a>
for older Chef versions, because our local Ruby environment is based
on RPMs. We build a single Ruby RPM for all our management tools, with
individual gems also built into RPMs, so our gem dependency resources
specify <tt>yum_package</tt> rather than <tt>gem_package</tt>.

This hasn't given us any problems beyond a need to patch cookbooks to
install our RPMs, but now we're open sourcing some of our cookbooks,
it means that our dependencies are specific to our local environment. 

Using <tt>chef_gem</tt> as an abstraction though, we can specify
dependencies in a way that's appropriate for an open source cookbook
but have a local override which actually installs our RPMs. We also
avoid having to patch cookbooks just to make sure our RPMs are
installed for dependencies.

This is the resource definition, which is a shim between the
<tt>chef_gem</tt> resource and the <tt>yum_package</tt> provider,
adjusting the requested package name as we need for our RPMs.

```ruby
class Chef
  class Resource
    begin
      send(:remove_const, 'ChefGem')
    rescue
      nil
    end

    class ChefGem < Chef::Resource::YumPackage

      def initialize(name, run_context=nil)
        # our gem name -> rpm name mapping
        name = "rubygem19-#{name}"
        super
        @resource_name = :chef_gem
        @provider = Chef::Provider::Package::Yum
        after_created
      end

      def after_created
        Array(@action).flatten.compact.each do |action|
          self.run_action(action)
        end
        Gem.clear_paths
      end
    end
  end
end
```

This code goes in a local-enviroment cookbook as a library. Once we
upgrade to Chef 0.10.10 or later, it will override the core
chef_gem resource, and continue to install our RPMs.

