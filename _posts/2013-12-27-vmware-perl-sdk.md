---
layout: post
title:  "Appendix D - Perl SDK for VMware vSphere"
date:   2013-12-27 22:56:00
categories: puppet
---

There are some things that Puppet's Cloud Provisioner can't do - one
of these is editing the specifications of the machine it's building
(i.e. CPU count, RAM etc.). The Cloud Provisioner approach expects
templates to be created with the necessary specifications, but this
would require maintenance of multiple 'standard build' templates,
which is not terribly practical.

Instead, we would like to be able to change various settings
automatically from within our provisioning scripts. For this, we'll
need to use the vSphere API.

Note: This document assumes the reader is familiar with using CPAN to
      install Perl modules.

## Installing the Perl SDK

We install this on the Puppet Console server, as it's the Cloud
Provisioner control node. Here are the [installation instructions][sdk-install].

### Download

The SDK can be [downloaded here][sdk-download].

Since we're using vCenter version 5.1.0, we've chosen the 5.1 version
of the SDK. The filename is VMware-vSphere-Perl-SDK-5.1.0-780721.x86_64.tar.gz

### Prerequisites

Some packages are required before the SDK will even consider installing:

{% highlight text %}
yum install openssl-devel libxml2-devel e2fsprogs-devel
{% endhighlight %}

We also need to set this environment variable to allow Perl's https
client to not worry about verifying certificates:

{% highlight text %}
export PERL_LWP_SSL_VERIFY_HOSTNAME=0
{% endhighlight %}

Without this, we'll get the following error when trying to connect
to vCenter:

{% highlight text %}
Server version unavailable at 'https://vc5.puppetlunch.com:443/sdk/vimService.wsdl' at /usr/share/perl5/VMware/VICommon.pm line 545.
{% endhighlight %}

It's useful to set this variable in our .bashrc file.

### Install

We move the tarball to the /tmp directory and unpack it.

{% highlight text %}
# cp VMware-vSphere-Perl-SDK-5.1.0-780721.x86_64.tar.gz /tmp
# cd /tmp
# tar zxf VMware-vSphere-Perl-SDK-5.1.0-780721.x86_64.tar.gz
# ./vmware-vsphere-cli-distrib/vmware-install.pl
{% endhighlight %}

We also had to set the http_proxy and ftp_proxy environment
variables, even though the installation instructions say this is an
optional step:

{% highlight text %}
# export http_proxy=''
# export ftp_proxy=''
{% endhighlight %}

The installer used CPAN to download any pre-requisite Perl modules, but it couldn't get all of them:

{% highlight text %}
CPAN not able to install following Perl modules on the system. These must be
installed manually for use by vSphere CLI:
  Class::MethodMaker 2.10 or newer
  UUID 0.03 or newer
  XML::LibXML::Common 0.13 or newer
  XML::LibXML 1.63 or newer
{% endhighlight %}

It couldn't install these due to some missing RPM packages:

{% highlight text %}
gcc
libuuid-devel
{% endhighlight %}

It's not clear whether the vSphere SDK installer was successful, so
we'll try running it again...

After accepting the licence agreement, we see this:

{% highlight text %}
Please wait while configuring CPAN ...
Please wait while configuring perl modules using CPAN ...
CPAN is downloading and installing pre-requisite Perl module "Archive::Zip" .
CPAN is downloading and installing pre-requisite Perl module "Data::Dump" .
CPAN is downloading and installing pre-requisite Perl module "SOAP::Lite" .
In which directory do you want to install the executable files?
[/usr/bin]

Please wait while copying vSphere CLI files...

The installation of vSphere CLI 5.1.0 build-780721 for Linux completed
successfully. You can decide to remove this software from your system at any
time by invoking the following command:
"/usr/bin/vmware-uninstall-vSphere-CLI.pl".

This installer has successfully installed both vSphere CLI and the vSphere SDK
for Perl.

The following Perl modules were found on the system but may be too old to work
with vSphere CLI:
  Compress::Zlib 2.037 or newer
  Compress::Raw::Zlib 2.037 or newer
  version 0.78 or newer
  IO::Compress::Base 2.037 or newer
  IO::Compress::Zlib::Constants 2.037 or newer
  LWP::Protocol::https 5.805 or newer

Enjoy,
--the VMware team
{% endhighlight %}

So it looks successful, but there's a warning about some out-of-date
Perl modules. These are the versions that ship with CentOS, so we may
as well upgrade these with the CPAN versions to avoid any subtle
problems later. To test, we use one of the SDK's utility scripts:

{% highlight text %}
# /usr/lib/vmware-vcli/apps/general/connect.pl --server vc5.puppetlunch.com \
> --username 'DOMAIN\auto_provisioner' --password xxxxxx
{% endhighlight %}

### Credential Store

To store passwords so we don't have to keep supplying them on the command line, we can use a credential store.
This is how to add an entry to the store:

{% highlight text %}
# /usr/lib/vmware-vcli/apps/general/credstore_admin.pl add \
> --server vc5.puppetlunch.com \
> --username 'DOMAIN\auto_provisioner' --password xxxxxxx
{% endhighlight %}

This adds an entry to the file /root/.vmware/credstore/vicredentials.xml.

The SDK scripts automatically look for credentials in this file, so
once the credentials have been added, we can run the same comands as
before without the password option:

{% highlight text %}
# /usr/lib/vmware-vcli/apps/general/connect.pl --server vc5.puppetlunch.com \
> --username 'DOMAIN\auto_provisioner'

Connection Successful
Server Time : 2013-12-10T12:09:19.436507Z
{% endhighlight %}

Note that this doesn't work with the --url option, only the --server option.

### A Problem...

In hindsight, updating LWP might not have been the best idea. It
seemed to mess up the SOAP communications between the client and
server. When testing with the above script, it took a very long time
to connect, then a very long time to eventually tell us there was a
"SOAP request error - possibly a protocol issue".

The answer lay in downgrading Net::HTTP and libwww-perl, as described
in this [helpful blog post][helpful-blog].

In summary, this is the fix:

{% highlight text %}
# cpan GAAS/Net-HTTP-6.03.tar.gz
# cpan GAAS/libwww-perl-6.03.tar.gz
{% endhighlight %}

Once that's done, the connect script (which used to take about 3m 50s
to run) completes in about 4 seconds. So we can now get started by
reading the SDK documentation.

## Summary

The Perl SDK is enormously useful - if you have time to figure out how
it works. Unlike most Perl modules, the VMware modules come with very
little documentation. The official documentation is terse and only
marginally helpful.

I eventually cobbled together a script to do what I wanted - namely,
set the number of vCPUs and amount of system memory allocated to a VM
- after trawling through the example scripts and utility modules. This
script enables us to set the desired spec of the VM at provisioning
time, and plugs neatly into the 'hatch' auto-provisioning script.

It's easy when you know how! I'll make that script available as soon
as possible.


[sdk-install]: http://pubs.vmware.com/vsphere-51/index.jsp?topic=%2Fcom.vmware.perlsdk.install.doc%2Fcli_install.3.5.html
[sdk-download]: https://my.vmware.com/group/vmware/details?downloadGroup=VSP510-SDKPERL-510&productId=268
[helpful-blog]: http://monitoringtt.blogspot.co.uk/2013/08/checkesx3checkvmwareapi-error-soap.html
