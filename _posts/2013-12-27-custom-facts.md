---
layout: page
title:  "Appendix A - Custom Facts"
date:   2013-12-27 21:49:00
categories: puppet
---

This is the usual course of events during a Puppet Agent run:

1. Puppet Agent collects all relevant 'facts' about itself and submits
   this information to the Puppet Master.
2. The Puppet Master examines this information and builds a 'catalog'
   containing configuration details for the node.
3. The Puppet Agent receives the catalog and makes any necessary changes.

The component which collects all the facts is called 'facter'. The
default list of facts is quite comprehensive, but there will be times
when decisions need to be made based on other information. In these
cases, the best option is to add custom facts to facter's database
which will then be automatically provided to the Puppet Master on the
next agent run.

## Defining Custom Facts

There are a number of ways to define custom facts. Two methods we've used so far are:

1. Defining 'external' facts in a YAML file.
2. Defining a fact using the pluginsync functionality via the 'custom' module.

### External Facts

This is the simplest method of defining a custom fact. We create a new file called
/etc/puppetlabs/facter/facts.d/FACT_NAME.yaml and add a line like this:

{% highlight yaml %}
factname: factvalue
{% endhighlight %}

For example, when defining a new node's 'role':

{% highlight yaml %}
role: web
{% endhighlight %}

Since the server role needs to be defined before the first puppet
agent run, we can just create the necessary file and trust that facter
will pick up the information. The 'hatch' autoprovisioning script does
exactly that:

{% highlight text %}
echo "Setting server role to '$ROLE'"
ssh $TEMPIP "mkdir -p /etc/puppetlabs/facter/facts.d && echo \"role: $ROLE\" >
/etc/puppetlabs/facter/facts.d/role.yaml"
{% endhighlight %}

Note that External facts override Ruby facts.

### Ruby Facts

We can define facts using Ruby code, but first we need to know where
to put the code. The best way to illustrate this is with a real-life
example. For auto-provisioning purposes, it would really help if the
puppet agent was aware of the IP address it is supposed to have,
rather than the one currently configured. Since we have the intended
hostname for the system ($::fqdn), we can get this information from
DNS, so we'll call the new fact 'dns_ipaddress'.

Note: For this method to work, the agent must have 'pluginsync'
      enabled. This is on by default in Puppet Enterprise:

File: /etc/puppetlabs/puppet/puppet.conf

{% highlight text %}
[agent]
pluginsync = true
{% endhighlight %}

Once that's done, we need to create a new module for custom facts (and
any other custom properties we may need) in
/etc/puppetlabs/puppet/modules:

{% highlight text %}
pl_custom
  lib
    facter
      dns_ipaddress.rb
  manifests
    init.pp
{% endhighlight %}

The init.pp file must exist or this won't be recognised as a valid
module, but it only needs to define the class.

{% highlight text %}
# Custom module for distributing custom facts, types, etc.
class pl_custom { }
{% endhighlight %}

Then in pl_custom/lib/facter/dns_ipaddress.rb we have our ruby code:

{% highlight ruby %}
Facter.add('dns_ipaddress') do
  setcode do
    Facter::Util::Resolution.exec("/usr/bin/host -t a $(/usr/local/bin/facter fqdn) | /bin/awk '{print $NF}'")
  end
end
{% endhighlight %}

As you can see, all it does is run a system command and assign the
output to a facter fact. The next time the Puppet Agent runs, it will
start by syncing plugins, and then evaluate any new code before
sending the fact list to the master.

And that's it!

## Further Reading

* [Custom Facts][custom-facts]
* [Plugins in Modules][plugins-in-modules]

[custom-facts]: http://docs.puppetlabs.com/guides/custom_facts.html
[plugins-in-modules]: http://docs.puppetlabs.com/guides/plugins_in_modules.html

