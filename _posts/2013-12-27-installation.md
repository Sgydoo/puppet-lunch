---
layout: page
title:  "Chapter 2 - Puppet Enterprise Installation"
date:   2013-12-27 16:24:00
categories: puppet
---

We're not expecting our installation to vary much from the standard
Puppet installation instructions, but it's important to make a note of
all decisions made during installation, including details of
hostnames, server specifications etc. so your fellow administrators
have a record of how the platform was created.

Installation will follow documented best practices as closely
as possible.

## The Platform

As mentioned in the previous <a
href="{{ site.baseurl }}/puppet/planning.html">planning section</a>,
there will eventually be two Puppet Enterprise installations; one for
Production and one for Non-Production. We'll be starting by
installing the Non-Production (NP) platform, and host names will be
suffixed with "-np".

However the installation method will be the same for both platforms.

The initial Puppet environment will consist of:

* 1 x Puppet Enterprise Puppet Master with Hiera-based configuration
* 1 x Puppet Enterprise console server with Cloud Provisioner
* 1 x PuppetDB server

The PE servers will be VMware guests running CentOS 6. Here are the details:

<table>
<tr><th>Hostname</th><th>OS</th><th>CPU Count</th><th>RAM</th><th>Disk Capacity</th></tr>
<td>puppetmaster-np.puppetlunch.com</td>
<td>64-bit CentOS 6.4</td>
<td>4</td>
<td>4GB</td>
<td>40GB</td>
</tr>
<tr>
<td>puppetconsole-np.puppetlunch.com</td>
<td>64-bit CentOS 6.4</td>
<td>2</td>
<td>4GB</td>
<td>40GB</td>
</tr>
<tr>
<td>puppetdb-np.puppetlunch.com</td>
<td>64-bit CentOS 6.4</td>
<td>2</td>
<td>4GB</td>
<td>40GB</td>
</tr>
</table>

Note: Make sure that the system clocks are (roughly) in sync, and all
host names are resolvable in DNS from each host before starting the
installation, otherwise we may run into difficulties. See
[the installation docs][b4-u-start] for more information.

Also ensure that the iptables firewall is disabled on each host before
starting the installation.

## Installing Puppet Enterprise

We'll be following the official [Puppet Enterprise Installation Guide][pe-install_basic]
(overview) with help from the [PE Deployment Guide][pe-deploy_guide]
(detailed instructions).

## Installation Sequence

The installation should be performed in this order:

<table>
<tr><th>Install Step</th><th>Host Name</th><th>Puppet Enterprise Role</th></tr>
<tr><td>1</td><td>puppetmaster-np.puppetlunch.com</td><td>Master Role</td></tr>
<tr><td>2</td><td>puppetdb-np.puppetlunch.com</td><td>Database Support Role (PuppetDB)</td></tr>
<tr><td>3</td><td>puppetconsole-np.puppetlunch.com</td><td>Console Role</td></tr>
<tr><td>4</td><td>puppetconsole-np.puppetlunch.com</td><td>Cloud Provisioner Role</td></tr>
</table>

The Agent Role should also be installed on all hosts.

### Tarball Download

We're running on CentOS 6.4, so we need to download the tarball for
RHEL-based systems.

<table>
<tr><th>PE Version</th><th>OS Version</th><th>Tarball Location</th><th>File Size</th></tr>
<tr><td>3.0.1</td><td>x86_64 EL (RHEL, CentOS, Scientific Linux, Oracle Linux) 6</td>
<td>https://pm.puppetlabs.com/cgi-bin/download.cgi?ver=latest&dist=el&arch=x86_64&rel=6</td>
<td>236MB</td></tr>
</table>

Note: If you prefer to download directly using curl, do this:

{% highlight text %}
curl -L -o pe-latest.tgz 'https://pm.puppetlabs.com/cgi-bin/download.cgi?ver=latest&dist=el&arch=x86_64&rel=6'
{% endhighlight %}

On each host, we unpack the tarball into /tmp, cd into the unpacked directory and run the installer script as root:

{% highlight text %}
$ sudo ./puppet-enterprise-installer
{% endhighlight %}

The installer will ask which roles should be installed. Any answers
given during installation will be recorded in the answer file here:
/etc/puppetlabs/installer/answers.install

To run the installation again using any of the answers below, save
them to a file and run the installer again with the -A option. If any
answers are missing, the installer will prompt for input.

{% highlight text %}
$ sudo ./puppet-enterprise-installer -A <ANSWER FILE>
{% endhighlight %}

### Installing the Master

Installation answerfile for the Non-Prod Puppet Master:

{% highlight text %}
q_all_in_one_install=n
q_database_install=n
q_install=y
q_pe_database=n
q_puppet_cloud_install=n
q_puppet_enterpriseconsole_install=n
q_puppet_symlinks_install=y
q_puppetagent_certname=puppetmaster-np.puppetlunch.com
q_puppetagent_install=y
q_puppetagent_server=puppetmaster-np.puppetlunch.com
q_puppetdb_hostname=puppetdb-np.puppetlunch.com
q_puppetdb_install=n
q_puppetdb_port=8081
q_puppetmaster_certname=puppetmaster-np.puppetlunch.com
q_puppetmaster_dnsaltnames=puppetmaster-np,puppetmaster-np.puppetlunch.com
q_puppetmaster_enterpriseconsole_hostname=puppetconsole-np.puppetlunch.com
q_puppetmaster_enterpriseconsole_port=443
q_puppetmaster_install=y
q_run_updtvpkg=n
q_vendor_packages_install=y
{% endhighlight %}

Installation complete:

{% highlight text %}
------------------------------------------------------------------------
STEP 4: DONE
Thanks for installing Puppet Enterprise!

Puppet Enterprise has been installed to "/opt/puppet," and its
configuration files are located in "/etc/puppetlabs".

## Answers from this session saved to
'/tmp/puppet-enterprise-3.0.1-el-6-x86_64/answers.lastrun.puppetmaster-np.puppetlunch.com'
========================================================================

If you have a firewall running, please ensure the following TCP ports
are open: 8140, 61613

If you have a firewall running, please ensure outbound connections to
are allowed via the following TCP ports: 443, 8081

NOTICE: This system has 3832 MB of memory, which is below the 4 GB we
recommend for the puppet master role. Although this node will be a
fully functional puppet master, you may experience poor performance
with large numbers of nodes. You can improve the puppet master's
performance by increasing its memory.

========================================================================
{% endhighlight %}

### Installing PuppetDB

Installation answerfile for the Non-Prod PuppetDB:

{% highlight text %}
q_all_in_one_install=n
q_database_host=puppetdb-np.puppetlunch.com
q_database_install=y
q_database_port=5432
#q_database_root_password=REDACTED
q_database_root_user=pe-postgres
q_fail_on_unsuccessful_master_lookup=y
q_install=y
q_pe_database=y
q_puppet_cloud_install=n
q_puppet_enterpriseconsole_auth_database_name=console_auth
#q_puppet_enterpriseconsole_auth_database_password=REDACTED
q_puppet_enterpriseconsole_auth_database_user=console_auth
q_puppet_enterpriseconsole_database_name=console
#q_puppet_enterpriseconsole_database_password=REDACTED
q_puppet_enterpriseconsole_database_user=console
q_puppet_enterpriseconsole_install=n
q_puppet_symlinks_install=y
q_puppetagent_certname=puppetdb-np.puppetlunch.com
q_puppetagent_install=y
q_puppetagent_server=puppetmaster-np.puppetlunch.com
q_puppetdb_database_name=pe-puppetdb
#q_puppetdb_database_password=REDACTED
q_puppetdb_database_user=pe-puppetdb
q_puppetdb_hostname=puppetdb-np.puppetlunch.com
q_puppetdb_install=y
q_puppetdb_port=8081
q_puppetmaster_certname=puppetmaster-np.puppetlunch.com
q_puppetmaster_install=n
q_run_updtvpkg=n
q_vendor_packages_install=n
{% endhighlight %}

Installation complete:

{% highlight text %}
------------------------------------------------------------------------
STEP 4: DONE
Thanks for installing Puppet Enterprise!
Puppet Enterprise has been installed to "/opt/puppet," and its
configuration files are located in "/etc/puppetlabs".

## Answers from this session saved to
'/tmp/puppet-enterprise-3.0.1-el-6-x86_64/answers.lastrun.puppetdb-np.puppetlunch.com'

## In addition, auto-generated database users and passwords have been saved to
"/etc/puppetlabs/installer/database_info.install"

!!! WARNING: Do not discard these files! All auto-generated database users
and passwords have been saved in them. You will need this information
to configure the console role during installation.

========================================================================
If you have a firewall running, please ensure the following TCP ports
are open: 5432, 8081

If you have a firewall running, please ensure outbound connections to
are allowed via the following TCP ports: 8140, 61613

NOTICE: This system has 3832 MB of memory, which is below the 4 GB we
recommend for the PuppetDB role. Although this node will be a fully
functional PuppetDB, you may experience poor performance with large
numbers of nodes. You can improve PuppetDB's performance by increasing
its memory.

Use this guideline to determine the amount of memory required for the
number of nodes installed.

NODES | MEMORY
------------------------------
1 - 100 | 192 - 512 MB
100 - 500 | 512 - 1024 MB
500 - 1000 | 1 - 2 GB
1000 - 2000 | 2 - 4 GB
> 2000 | 4 GB or greater
========================================================================
{% endhighlight %}

### Installing PE Console and Cloud Provisioner

Installation answerfile for Console and Cloud Provisioner:

{% highlight text %}
q_all_in_one_install=n
q_database_host=puppetdb-np.puppetlunch.com
q_database_install=n
q_database_port=5432
q_fail_on_unsuccessful_master_lookup=y
q_install=y
q_pe_database=n
q_puppet_cloud_install=y
q_puppet_enterpriseconsole_auth_database_name=console_auth
#q_puppet_enterpriseconsole_auth_database_password=REDACTED
q_puppet_enterpriseconsole_auth_database_user=console_auth
#q_puppet_enterpriseconsole_auth_password=REDACTED
q_puppet_enterpriseconsole_auth_user_email=simon@puppetlunch.com
q_puppet_enterpriseconsole_database_name=console
#q_puppet_enterpriseconsole_database_password=REDACTED
q_puppet_enterpriseconsole_database_user=console
q_puppet_enterpriseconsole_httpd_port=443
q_puppet_enterpriseconsole_install=y
q_puppet_enterpriseconsole_master_hostname=puppetmaster-np.puppetlunch.com
q_puppet_enterpriseconsole_smtp_host=mail.puppetlunch.com
#q_puppet_enterpriseconsole_smtp_password=REDACTED
q_puppet_enterpriseconsole_smtp_port=25
q_puppet_enterpriseconsole_smtp_use_tls=n
q_puppet_enterpriseconsole_smtp_user_auth=n
q_puppet_enterpriseconsole_smtp_username=
q_puppet_symlinks_install=y
q_puppetagent_certname=puppetconsole-np.puppetlunch.com
q_puppetagent_install=y
q_puppetagent_server=puppetmaster-np.puppetlunch.com
q_puppetca_install=n
q_puppetdb_database_name=pe-puppetdb
#q_puppetdb_database_password=REDACTED
q_puppetdb_database_user=pe-puppetdb
q_puppetdb_hostname=puppetdb-np.puppetlunch.com
q_puppetdb_install=n
q_puppetdb_port=8081
q_puppetmaster_enterpriseconsole_hostname=localhost
q_puppetmaster_install=n
q_run_updtvpkg=n
q_vendor_packages_install=y
{% endhighlight %}

Installation complete:

{% highlight text %}
------------------------------------------------------------------------
STEP 4: DONE
Thanks for installing Puppet Enterprise!

Puppet Enterprise has been installed to "/opt/puppet," and its
configuration files are located in "/etc/puppetlabs".

## Answers from this session saved to
'/tmp/puppet-enterprise-3.0.1-el-6-x86_64/answers.lastrun.puppetconsole-np.puppetlunch.com'
========================================================================
The console can be reached at the following URI:
* https://puppetconsole-np.puppetlunch.com

If you have a firewall running, please ensure the following TCP ports
are open: 443

If you have a firewall running, please ensure outbound connections to
are allowed via the following TCP ports: 8140, 61613, 5432
{% endhighlight %}

### Logging In To The Console

Once everything's installed, we're able to log in to the
Non-Production Puppet Enterprise Console.

### Securing The Installation

#### Database Security

PuppetDB uses (and installs) PostgreSQL. The database should be used
exclusively by PuppetDB. Details on how to secure the installation may
be found [here][postgres-hardening].

As a basic security measure, we're only allowing database connections
from the local machine, the puppet console and the puppet master. This
is configured in pg_hba.conf:

{% highlight text %}
File: /opt/puppet/var/lib/pgsql/9.2/data/pg_hba.conf

# Rule Name: allow access to all users
# Description: none
# Order: 100
#host all all 0.0.0.0/0 md5
#
# Edit: Only allow network access from PuppetDB, Master and Console
host all all 10.20.1.162/32 md5
host all all 10.20.1.163/32 md5
host all all 10.20.1.164/32 md5
{% endhighlight %}

## Next...

* <a href="{{ site.baseurl }}/puppet/hiera.html">Chapter 3 - Configuring Puppet With Hiera</a>
* <a href="{{ site.baseurl }}/contents">Table of Contents</a>


[b4-u-start]: http://docs.puppetlabs.com/guides/deployment_guide/index.html#before-starting-the-install
[pe-install_basic]: http://docs.puppetlabs.com/pe/latest/install_basic.html
[pe-deploy_guide]: http://docs.puppetlabs.com/guides/deployment_guide/index.html
[postgres-hardening]: https://www.owasp.org/index.php/OWASP_Backend_Security-_Project_PostgreSQL_Hardening
