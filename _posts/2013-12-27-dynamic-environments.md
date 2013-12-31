---
layout: post
title:  "Appendix B - Dynamic Environments"
date:   2013-12-27 22:04:00
categories: puppet
---

One really powerful feature of Puppet is the ability to create an
arbitrary 'environment' on the fly. The feature is known as "dynamic
environments". It's a side-effect of allowing global variables in
Puppet's configuration file, and was not intentionally implemented by
the Puppet developers. The ability was first documented in [an October
2010 blog post][hh-blog] by Hunter Haugen.

The concept was further developed by [Adrien Thebo in November 2011][finch-blog],
whose article is widely referred to by many other blogs and fora
(Adrien went on to develop r10k, an automated module deployment tool
which supports dynamic environments).

However, there's a problem with ridiculously powerful features - they
give us too many options, especially if we start throwing other
powerful features like Hiera into the mix. So now it's time to try and
sift through the many possibilities and arrive at a simple, elegant
solution...

## Discussion

Going back to basics, these are the primary building blocks for a flexible Puppet installation:

* Modules - for removing all site-specific config from the site manifest file. Two main types of module:
    * Puppet Forge modules
    * Locally developed modules, which can be further divided into:
        * Basic package management
        * Simple "wrapper" classes for Puppet Forge modules
        * Standalone modules (e.g. for bespoke application management)
* Hiera - for removing configuration data from modules.
* Puppet configuration in Version Control (e.g. Git) - for simplified audit, rollback and development.
* Dynamic Environments - for development and safe experimentation.

When considering Dynamic Environments, one problem that immediately presents itself is the fact that we've
already defined our environment in Hiera's hierarchy, like this:

{% highlight yaml %}
---
:backends:
- yaml
:hierarchy:
- node/%{::fqdn}
- "%{::environment}/%{::role}"
- role/%{::role}
- "%{::environment}"
- global
:yaml:
:datadir: /etc/puppetlabs/puppet/hiera
{% endhighlight %}

So Hiera was already providing a flexible mechanism for classifying
nodes on a per-environment basis. If we were to switch to dynamic
environments using this hierarchy definition, we would have to clone
the existing hierarchy, adding a new directory to the datadir for each
possible environment. Before too long, Hiera's datadir would become
unmaintainable.

However, in our temporary test environments, we still want different
server configurations for different real environm ents. For example,
say I want to create a new environment for testing, but I want the
hosts to be configured in a Production-like way. My test environment
could be called anything at all, but if I want the machines in this
environment to be Production-like, I'll somehow need to let Puppet
know this so it can configure them correctly. This shows that we need
to reconsider what is meant by 'environment'. In fact, it shows that
we need two different concepts; a "model" environment (e.g. prod) and
a "temporary" environment (e.g. sytest1).

### Model (or Hiera) Environments

These are the traditional, usually permanent environments like
Development, QA, Staging or Production.

To keep things as simple as possible, we'll start out with two basic
models: 'prod' and 'non-prod'.

We'll define these in Hiera and classify nodes using a custom fact
(provisionally called $::model_environment). So our role-based Hiera
configuration would become:

{% highlight yaml %}
---
:backends:
- yaml
:hierarchy:
- node/%{::fqdn}
- "%{::model_environment}/%{::role}"
- role/%{::role}
- "%{::model_environment}"
- global
:yaml:
:datadir: /etc/puppetlabs/puppet/hiera
{% endhighlight %}

OK. It's starting to look good, but what about that datadir? If it's a
constant, we'll still need to use the same configuration data for all
nodes, whereas we actually want to be able to make changes which only
affect our temporary environment. So we'll discuss temporary
environments, then see what the Hiera config should be...

### Temporary (Puppet) Environments

These are environments used for testing purposes. For example, I want
to provision a production-like system for testing a new feature, but I
don't want to call the environment 'production', so we need to be able
to create an arbitrarily-named environment without affecting the
intended Production-like nature of the environment.

Of course environments created like this don't have to be temporary at
all. They could easily last for months or years, or be permanent
fixtures. That's one of the reasons creating environments from Git
branches is so flexible.

Clearly, Hiera needs to be aware of both types of environment. Let's tie them together...

### Hiera Configuration

To do this, Hiera's datadir must now be *dynamically* defined:

{% highlight yaml %}
---
:backends:
- yaml
:hierarchy:
- node/%{::fqdn}
- "%{::model_environment}/%{::role}"
- role/%{::role}
- "%{::model_environment}"
- global
:yaml:
:datadir: /etc/puppetlabs/puppet/environments/%{::environment}/hiera
{% endhighlight %}

This gives us a copy of the existing Hiera data in each temporary
environment, which is exactly what we're looking for.

### Puppet Modules

But what about modules? Some of these are developed in-house and
stored in a Git repository, while others come from the outside world
via Puppet Forge, so they would have to live in separate directories.

To make things even more interesting, we may want to use different Forge modules in different environments. For
example, if we decided we wanted to switch to a different apache module, but needed to test everything safely first.
And of course we would also like to implement Dynamic Environments in Puppet. So our requirements become:

1. Configure Puppet for dynamic environments.
2. Keep our locally-developed modules separate from Puppet Forge
   downloads (so we don't have to keep Puppet Forge modules in git).
3. Manage Puppet Forge modules separately for each environment.
4. Retain a directory for 'common' modules (to avoid repeating standard components).

To satisfy these criteria, we would need two module directories per
environment (local and forge). We would also have to adjust the module
search path via the modulepath directive. This is what the modulepath
in our puppet.conf file would look like:

{% highlight text %}
modulepath = /etc/puppetlabs/puppet/environments/$environment/forge_modules:/etc/puppetlabs/puppet/environments/$environment/local_modules:/etc/puppetlabs/puppet/modules:/opt/puppet/share/puppet/modules
{% endhighlight %}

In our git repo, the forge_modules directory would be empty except for
a file called 'Puppetfile' which tells r10k to fetch appropriate
modules (and versions thereof) from the Puppet Forge.

### Example Workflow

#### System at Rest

We should have a default value for both these variables. Assuming
we've only defined two values for $::model_environment ('prod' and
'non-prod'), and if we make the default puppet $::environment
'production', then we get the following directory layout if no other
environment branches exist:

{% highlight text %}
puppet
  environments
    production
      Puppetfile
      modules
        apache
        firewall
        tomcat
      hiera
        global.yaml
        node
          myserver.yaml
        non-prod
        non-prod.yaml
        prod
          web.yaml
        prod.yaml
        role
          tomcat.yaml
          web.yaml
      local_modules
        pl_apache
        pl_firewall
        pl_tomcat
hiera.yaml
manifests
  site.pp
modules
puppet.conf
{% endhighlight %}

The environments/production directory is stored in Git as our
master branch. To create a new environment, we just have to create
a new branch.

#### A New Test Environment

So we want to create a new temporary Production-like environment
called 'sytest1', to test a new feature or module without disturbing
production. This is what we do:

* Pull the master (standard production) branch from Git.
* Create a new branch called 'sytest1'.
* Make the necessary changes to the new branch and commit locally.
* Push the new branch to the Git repository.
* Run 'r10k deploy' on the puppet master to fetch the new branch from
  git and download any required modules.

Once finished, the directory tree will look like this:

{% highlight text %}
puppet
  environments
    production
      Puppetfile
      modules
        apache
        firewall
        tomcat
      hiera
        global.yaml
        node
          myserver.yaml
        non-prod
        non-prod.yaml
        prod
          web.yaml
        prod.yaml
        role
          tomcat.yaml
          web.yaml
      local_modules
        pl_apache
        pl_firewall
        pl_tomcat
    sytest1
      Puppetfile
      modules
        apache
        firewall
        tomcat
      hiera
        global.yaml
        node
          myserver.yaml
        non-prod
        non-prod.yaml
        prod
          web.yaml
        prod.yaml
        role
          tomcat.yaml
          web.yaml
      local_modules
        pl_apache
        pl_firewall
        pl_tomcat
hiera.yaml
manifests
  site.pp
modules
puppet.conf
{% endhighlight %}

* We then provision a new host (or set of hosts) with the puppet
  agent's $::environment variable set to 'sytest1' and
  $::model_environment set to 'prod'.
* When the new development's done and tested successfully, we can
  merge the 'sytest1' branch into 'production' and delete the
  sytest1 branch.

This should result in an updated environments/production directory
with the new fully-tested changes in place. The sytest1 directory
will be automatically removed from the puppet master following the
branch merge back into production and subsequent r10k run.

Simple!

## Dynamic Environments and Auto-provisioning

We have a script called 'mkbrood' which is described in
<a href="{{ site.baseurl }}/puppet/autoprovisioning.html">04 - Auto-provisioning
With Cloud Provisioner</a>. This takes a 'platform name' as an argument. If
we replace 'platform name' with 'transient environment' (or vice
versa) then we can create arbitrary platforms for our applications.

So we'd be able to say: "Build me a Production-like platform called
'stg-webservices'. It should have two web servers, two tomcat
application servers and a database server. It will run the LHCwidgets
application."

This can be easily defined with environments and roles. For example:

{% highlight text %}
environment:         stg-webservices
model_environment:   prod
server roles:        web-lhcwidgets, tomcat-lhcwidgets, postgres
{% endhighlight %}

We then:

* Create a new git branch from production called 'stg-webservices'.
* Plug the above information into the mkbrood configuration.
* Run the script.
* Allow Puppet to configure everything.

Interesting?

## Summary

We now have a very flexible solution which appears to satisfy all the
criteria without being overly complicated. The next step is to try it
out and report our findings...


[hh-blog]: http://hunnur.com/blog/2010/10/dynamic-git-branch-puppet-environments/
[finch-blog]: http://puppetlabs.com/blog/git-workflow-and-puppet-environments
