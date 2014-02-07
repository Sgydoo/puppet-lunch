---
layout: page
title:  "Chapter 4 - Auto-provisioning With Cloud Provisioner"
date:   2013-12-27 18:21:00
categories: puppet
---

Provisioning a new server can be time-consuming. Auto-provisioning is
the ability to build a standard, fully configured system with very
little input. Puppet Enterprise provides tools which can help with
this, but the process still requires some manual intervention. The aim
of this exercise is to produce a single script which handles all the
fiddly bits, resulting in a fully functional system.

We're running Cloud Provisioner from the Puppet Console host. Puppet
terminology for this host is the "CP control node".

## Auto-provisioning - Fundamental Problems

The subject of auto-provisioning raises some questions which must be
answered before we can proceed. There are also a few different ways we
can answer these questions. We'll try and summarise the main issues in
the following table.

#### Q: Do we clone an existing VM image, or install the OS from scratch?

To automatically install from scratch requires the use of DHCP and
PXE, with new nodes downloading a microkernel, then running a custom
unattended installation procedure using a downloaded OS installation
ISO image. It's a rather involved process, but it's the only one which
works for new *physical* machines.

A: We will use VMware 'templates', which are essentially virtual
machine images, ready-installed with a basic version of our chosen OS.

Cloud Provisioner command: puppet node_vmware create

#### Q: How do we determine the IP address of the new node?

If using DHCP, this can be obtained by parsing the output of 'puppet
node_vmware find'.

Alternatively, all newly-provisioned nodes could be initiated with a
known static IP address. We will take this approach, but only because
it's the easiest thing to do in *our* environment.

#### Q: How do we ensure the Puppet Agent is configured with the correct certificate name?

Once the IP address of the new node is known, we can install the agent
across the network using 'puppet node install', providing the agent
certificate name with the --certname option.

SSH must be configured appropriately on the new node to allow access
from the Cloud Provisioner host, so this must be a property of the
VMware template we're using (more on this later).

Cloud Provisioner command: puppet node install

#### Q: How do we automatically configure the network?

Puppet can manage the IP address of a host, but ideally the interface
must be configured before any other network-dependent services are
installed. There may be a way to specify this dependency in Puppet,
but I haven't found this yet.

The easiest solution we've found so far is to manipulate the network
configuration files directly using sed, then rebooting the new VM
before the puppet agent starts to configure the machine. It's not
ideal, and it's not elegant, but it will work - and it solves the
dependency problem.

This will require a custom script (more on this script later).

## Pre-requisites

### VMware Template Requirements

We'll be using a basic CentOS 6 VMware template, which needs a bit of preparation. Namely:

* The SSH public key of the Cloud Provisioner host's root account listed in /root/.ssh/authorized_keys
* VMware-tools installed
* A known static IP address configured \*
* No entries in /etc/udev/rules.d/70-persistent-net.rules
* Appropriate entries in /etc/resolv.conf
* NetworkManager disabled

\* Of course it's also possible to use DHCP rather than a pre-determined
IP address, as mentioned previously.

### Credentials for the VMware Cluster

These are required to let the Cloud Provisioner talk to VMware to
create, start, stop VMs and to query the running machine list. We've
set up a new vCenter user called 'auto_provisioner'. The credentals are
added to the root user's ~/.fog file, which looks like this:

{% highlight yaml %}
:default:
  :vsphere_server: vc5.puppetlunch.com
  :vsphere_username: DOMAIN\auto_provisioner
  :vsphere_password: xxxxxxx
  :vsphere_expected_pubkey_hash: 177bf9cc5a80b7bf74d301ec6a7d51d5...
{% endhighlight %}

Note: We have to specify the Windows domain name in the username field.
This was not mentioned in any of the official documentation.

The vsphere_expected_pubkey_hash is returned by vCenter and allows us
to confirm that we're talking to the correct host.

## Method

Before creating a new VM, we'll need a corresponding Address record
added to DNS. Once that's done, we can run the shell script which will
be responsible for provisioning a new host. This complete method for
provisioning a new machine will only require two simple steps:

1. Create new DNS entries for the hosts to be provisioned.
2. Run the provisioning script once per node.

The script we're going to write will do a lot of things for us. Here's
what it has to do...

1. Startup checks:

    * Check we're running as the correct user.
    * Validate command line parameters.
    * Check the new host's DNS entry is in place (not strictly
      necessary, but it ensures this step is completed).
    * Check that nothing responds to the temporary provisioning IP address.
    * Check that nothing responds to the target IP address.
    * Check whether an agent certificate for the new host already exists (it shouldn't).
2. Clone the specified template and wait for the new VM to boot.
3. Install puppet agent, which will automatically send a certificate signing request to the master.
4. Sign the certificate for the new node.
5. Update the 'environment' setting in the agent's puppet.conf.
6. Set the 'role' fact.
7. Reconfigure the network (scripted manipulation of config files)
8. Reboot

One the machine has rebooted, and assuming that the network
reconfiguration was successful, the puppet agent should run and
configure everything the way we want it.

But first, we'll run through all the steps manually to test the
concept...

## Manual Implementation

### Creating the new VM from a Template

{% highlight text %}
# time puppet node_vmware create \
> --template /Datacenters/Timbuktu/vm/Test/Automation/templates/centos6-basic \
> --vmname autoprov-temp1.puppetlunch.com --wait-for-boot

Notice: Connecting ...
Notice: Connected to vc5.puppetlunch.com as DOMAIN\auto_provisioner (API version 4.1)
Notice: Locating VM at /Datacenters/Timbuktu/vm/Test/Automation/templates/centos6-basic (Started at 12:24:54 PM)
Notice: Control will be returned to you in 10 minutes at 12:34 PM if locating (1/3) is unfinished.
Locating (1/3): 100%
|ooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo| Time:
00:00:00

Notice: Starting the clone process (Started at 12:24:55 PM)
Notice: Control will be returned to you in 10 minutes at 12:34 PM if starting (2/3) is unfinished.
Starting (2/3): 100%
|ooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo| Time:
00:02:42

Notice: Waiting for the machine to boot and obtain an IP address ... (Started at 12:27:38 PM)
Notice: Control will be returned to you in 10 minutes at 12:37 PM if booting (3/3) is unfinished.
Booting (3/3): 100%
|ooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo| Time:
00:01:21
---
  status: success
  id: "500661dd-cc72-d00d-35fc-33f13e4b5615"
  name: autoprov-temp1.puppetlunch.com
  uuid: "4206a374-d06e-0ac6-c0b1-29b5eef07c04"
  instance_uuid: "500661dd-cc72-d00d-35fc-33f13e4b5615"
  hostname: autoprovisioner-np.puppetlunch.com
  operatingsystem: "CentOS 4/5/6 (64-bit)"
  ipaddress: "10.40.3.167"
  power_state: poweredOn
  connection_state: connected
  hypervisor: hp1000-2.puppetlunch.com
  tools_state: toolsOk
  tools_version: guestToolsCurrent
  is_a_template: false
  memory_mb: 4096
  cpus: 2
  mo_ref: vm-32082
  mac_addresses:
    "Network adapter 1": "00:50:56:86:4d:9c"
  path: /Datacenters/Timbuktu/vm/Test/Automation/templates

real 4m10.839s
user 0m4.103s
sys 0m0.205s
{% endhighlight %}

### Installing the Puppet Agent

This is the command we want to run to install the Puppet agent:

{% highlight text %}
# puppet node install --login=root --keyfile=/root/.ssh/id_rsa \
  --install-script=puppet-enterprise \
  --installer-payload=/path/to/puppet-enterprise-3.1.0-el-6-x86_64.tar.gz \
  --installer-answers=/path/to/answers_file \
  --puppetagent-certname=autoprov-temp1.puppetlunch.com 10.40.3.167
{% endhighlight %}

Here's the output:

{% highlight text %}
Notice: Waiting for SSH response ...
Notice: Waiting for SSH response ... Done
Notice: Uploading Puppet Enterprise tarball ...
Notice: Uploading Puppet Enterprise tarball ... Done
Notice: Installing Puppet ...
puppetagent_certname: autoprov-temp1.puppetlunch.com
  status: success
  stdout: autoprov-temp1.puppetlunch.com
{% endhighlight %}

If we then check the Puppet Console, we see there's a new node
certificate waiting to be signed! Alternatively, we can check for
pending certificate signing requests from the command line on the
Puppet Master:

{% highlight text %}
[root@puppetmaster-np ~]# puppet cert list
"autoprov-temp1.puppetlunch.com" (SHA256)
C0:F6:D7:89:53:8F:C0:66:E6:E9:C6:DD:C0:F2:86:9C:67:49:D4:CA:77:D5:90:...
{% endhighlight %}

### Signing the Certificate

Since we're running everything from the Puppet Console machine, we'll
want to sign the new agent's certificate from this machine too. It's
possible to do remote certificate signing, but we have to configure a
few things first...

#### Remote Certificate Signing

Before we can remotely sign certificate requests, we need to:

* Generate a new certificate on the Puppet Console host.
* Sign the new certificate on the Puppet Master.
* Configure ACLs on the master to allow certificate signing from this host.

On the CP control node:

{% highlight text %}
# puppet certificate generate cpcontrol --ca-location remote
true
{% endhighlight %}

On the master, we can list and then sign the new certificate:

{% highlight text %}
[root@puppetmaster-np puppet]# puppet cert list
"autoprov-temp1.puppetlunch.com" (SHA256)
C0:F6:D7:89:53:8F:C0:66:E6:E9:C6:DD:C0:F2:86:9C:67:49:D4:CA:77:D5:90:...
"cpcontrol" (SHA256)
AC:F2:50:FF:78:74:E1:B0:41:D8:40:E6:06:38:ED:2C:34:09:CA:A2:74:1F:67:...

[root@puppetmaster-np puppet]# puppet cert sign cpcontrol
Notice: Signed certificate request for cpcontrol
Notice: Removing file Puppet::SSL::CertificateRequest cpcontrol at
'/etc/puppetlabs/puppet/ssl/ca/requests/cpcontrol.pem'
{% endhighlight %}

Again on the master, add an 'allow' line to the relevant ACL in
Puppet's $confdir/auth.conf file:

{% highlight text %}
path /certificate_status
method find, search, save, destroy
auth yes
allow pe-internal-dashboard
allow cpcontrol
{% endhighlight %}

So now, on the CP control node, we can request a list of certificate
signing requests by specifying the name of the newly-signed
certificate:

{% highlight text %}
[root@puppetconsole-np ~]# puppet certificate list --ca-location remote --certname cpcontrol
#<Puppet::SSL::Host:0x000000016d9e98 @name="autoprov-temp1.puppetlunch.com",
@certificate_request=nil, @certificate=nil, @key=nil, @ca=false,
@expiration=2013-10-17 14:17:53 +0100>
{% endhighlight %}

Brilliant! So to sign the certificate, we can do this:

{% highlight text %}
puppet certificate sign autoprov-temp1.puppetlunch.com --ca-location remote --certname cpcontrol
{% endhighlight %}

To remove this certificate between test runs, we need to revoke it on the master:

{% highlight text %}
[root@puppetmaster-np puppet]# puppet cert clean autoprov-temp1.puppetlunch.com
Notice: Revoked certificate with serial 11
Notice: Removing file Puppet::SSL::Certificate autoprov-temp1.puppetlunch.com at
'/etc/puppetlabs/puppet/ssl/ca/signed/autoprov-temp1.puppetlunch.com.pem'
Notice: Removing file Puppet::SSL::Certificate autoprov-temp1.puppetlunch.com at
'/etc/puppetlabs/puppet/ssl/certs/autoprov-temp1.puppetlunch.com.pem'
{% endhighlight %}

More info on these commands can be found here:

* [HTTP Access Control][http-access-control]
* [Troubleshooting Cloud Provisioner - Certificate Signing Issues][cert-signing-issues]

### Configuring the Network

So the quick and dirty way of configuring the network is to use sed to
edit the network files directly.

As with most things there's more than one way to do it, and I expect
there's a cleverer way than this, but it works fine for now.

### Rebooting

Once the network is all configured, it makes sense to reboot before
going any further. To do this, we call 'puppet node_vmware stop' and
then 'puppet node_vmware start'.

When the node comes back up and the Puppet agent is running, it will
immediately contact the puppet master, collect a catalog and start
making the requested changes to the system.

## Automatic Implementation

I've written a bash shell script which ties all this together in a
single command. The script is called 'hatch', and lives on the Puppet
Console host. It's inspired by [a really useful script by Ger Apeldoorn][ger-script].

Another script called 'mkbrood' is capable of using the hatch script
to provision multiple hosts according to instructions in a
configuration file (a bit like Amazon's Cloud Formation facility).

Both scripts can be found on [GitHub][auto-scripts]!

Now, here's how to use them...

{% highlight text %}
# hatch --help

This is the 'hatch' auto-provisioning script. Version: 2.1

Script Actions:
  * Clone a VMWare template
  * Install the Puppet-client on the new machine
  * Configure the network
  * Reboot the machine

Usage:
  hatch -n hostname -r role [ -p puppet_environment ] [ -h hiera_environment ] [ -u ] [ -t template ] [ -C numCPUs ] [-M memory ]
  hatch --help

Options:
  -n      :  Hostname of the new node
  -r      :  Server Role
  -p      :  Puppet Environment (Optional. Defaults to 'production')
  -h      :  Hiera Environment (Also optional. Defaults to 'prod')
  -u      :  Unattended. If set, the script will run without prompting
  -t      :  Template name. Optional. Defaults to 'centos6-static'
  -C      :  Number of virtual CPUs. Optional (Passed to VMSPEC script)
  -M      :  Amount of system memory, in MiB. Optional (Passed to VMSPEC script)
  --help  :  Print this message
{% endhighlight %}

### Provisioning Multiple Nodes

The hatch script has a wrapper script called 'mkbrood', written in Perl. A YAML configuration file describes the hosts
to be built. The script reads the file and runs 'hatch' for each node described.

{% highlight text %}
# mkbrood --help

This is the 'mkbrood' multiple host provisioner. Version 1.3

mkbrood is a wrapper script for the 'hatch' auto-provisioning script.
It allows a number of new nodes to be produced (hatched) at one time,
which is useful when provisioning a complete application platform.

The brood script expects to find its configuration files here:
 /etc/mkbrood/<platform>.yaml

It relies on the 'hatch' script, which it expects to find here:
 /usr/local/bin/hatch

Logs are written to:
 /var/log/mkbrood/<platform>.log

Usage:

 mkbrood <platform>
 mkbrood --help

Note that any environments referred to in the configuration must exist
in the puppet configuration *before* running this script.
{% endhighlight %}

Note that a valid environment with the same name must exist in the
puppet configuration *before* running this script.

A typical platform configuration file might look like this:

{% highlight yaml %}
---
model_environment: non-prod
node1:
  hostname: autoprov-temp1.puppetlunch.com
  role: web
node2:
  hostname: autoprov-temp2.puppetlunch.com
  role: tomcat
{% endhighlight %}

## Next...

* <a href="{{ site.baseurl }}/puppet/version-control.html">Chapter 5 - Managing Puppet Configuration With Git</a>
* <a href="{{ site.baseurl }}/contents">Table of Contents</a>


[http-access-control]: http://docs.puppetlabs.com/guides/rest_auth_conf.html
[cert-signing-issues]: http://docs.puppetlabs.com/pe/latest/trouble_cloudprovisioner.html#certificate-signing-issues
[ger-script]: http://puppetspecialist.nl/2012/12/single-command-server-deployment/
[auto-scripts]: https://github.com/Sgydoo/autoprovision
