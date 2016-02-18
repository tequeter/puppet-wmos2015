# wmos2015::app

#### Table of Contents

1. [Module Description - What the module does and why it is useful](#module-description)
2. [Setup - The basics of getting started with wmos2015::app](#setup)
    * [What wmos2015::app affects](#what-wmos2015::app-affects)
    * [Setup requirements](#setup-requirements)
    * [Beginning with wmos2015::app](#beginning-with-wmos2015::app)
3. [Usage - Configuration options and additional functionality](#usage)
4. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
5. [Limitations - OS compatibility, etc.](#limitations)
6. [Development - Guide for contributing to the module](#development)

## Module Description

wmos2015::app installs the application tier of Manhattan's WMOS2015, a
Warehouse Management System.

It:
* Configures the operating system according to Manhattan's prerequisites.
* Generates answer files (.properties) for System Director (Manhattan's graphical
  installer).
* Supports a distributed setup for SD (ie. generates .properties for host A on
  host B if the SD running on host B is supposed to manage host A).
* Creates a unprivileged wmosrf user that can telnet/ssh in the server to use
  the rfclient, without access to the passwords etc. of the wmos application
  account.
* And more, see [What wmos2015::app affects].

We use it in production at Idgroup, but although quite configurable it has
some bias towards our use-cases (OS, features used etc.)

## Setup

### What wmos2015::app affects

* System configuration (sysctl and user limits) as per Manhattan.
* Installs several packages required by Manhattan (the default list includes
  ksh, sharutils, ant etc.)
* Creates a /usr/bin/ksh symlink to your ksh.
* Installs Java 7 (unless disabled).
* Installs JBoss 6.3.x from Redhat's RPMs (unless disabled, will not work if
  you don't have a JBoss subscription).
* Configures JBoss to match Manhattan's specs.
* Creates an application account (default: wmos) and group (default: manh).
* Creates a unprivileged user for rfclient (default: wmosrf).
* Configures the .bashrc and .bash_profile for these users.
* Creates a directory hierarchy to install WMOS in (default: /manh and
  /apps/scope). If installing in a different location, these Manhattan-standard
  paths will be created as symlinks instead.
* The default working space for SD is made unusable through file permissions,
  to force using the unsafe/ subdirectory (which is protected from wmosrf).
* Installs distribution files on the host that should act as the "distribution
  server", if you repackaged Manhattan's ZIPs in RPM.
* Installs System Director (unless disabled), if you repackaged Manhattan's ZIP
  in RPM. Configures SD for your Java home.
* Open the required ports in your local firewall (unless disabled, depends on
  the local_firewall module).
* Adds the Manhattan-mandated options in your sqlnet.ora Oracle client
  configuration (expected to be managed externally).
* Integrates WMOS components with your init system (currently only supports
  systemd, though).
* Applies post-install fixes provided by Manhattan (currently only a fix to
  support JBoss 6.3.3).
* Locks down the component installation directories to protect them from
  wmosrf, with some exceptions and ACLs in wms to allow running rfclient.
* Creates a pkms.env.safe configuration file without the passwords and readable
  by wmosrf.

### Setup requirements

* Puppet 4.
* PuppetDB for the distributed System Director feature.
* External modules:
    * puppetlabs/stdlib.
    * puppetlabs/concat.
    * camptocamp/systemd.
    * dobbymoodge/acl.
    * local_firewall (if $manage_local_firewall is true).
* Packages for your OS for System Director and distribution files (usually
  repackaged from the ZIPs of Manhattan).

Highly recommended: Hiera with a crypto backend like eyaml. You'll need to
provide several passwords.

See also: [Limitations].

### Beginning with wmos2015::app

Things you'll probably want to do somewhere in your Puppet config:

* Enabling the software distribution channel that provides packages like
  "sharutils" (Server Optional in Redhat-based OSes).
* Installing an Oracle client.
* Install JBoss 6.3, or enable the subscription-based JBoss SDC.
* Setup a CUPS server to talk to you barcode printers.
* Install and configure a Telnet server if needed.

You need to prepare a database server according to Manhattan's specifications.
It's not covered by wmos2015::app, and we didn't write wmos2015::db.

Then declare the main `wmos2015::app` class with the required arguments (see
below).

The actual installation has three steps:

1. Run Puppet a first time to configure the system, install System Director etc.
2. Use System Director to install the components you need.
3. Run Puppet again to fix permissions, install patches etc.

*Important*: There are about 40 fields to fill in per component in System
Director, but just use the "Saved Property" feature to the right of the screen
and select the .property file that Puppet generated for you. This will save you
a lot of time.

*Important*: You can manage several hosts with a single SD, and Puppet will
generate per-host .properties on the SD server. However, you need to run step 1
on the SD host *after* you run it on the slave servers. Otherwise, the PuppetDB
will not have the resources (configuration) exported from the slave servers to
generate the .properties on the SD server.

## Usage

The good news is that you only have to configure (and declare) the
wmos2015::app class, the bad news is that there is a significant amount of
non-optional configuration options. Have a look at wmos2015/manifests/app.pp
for the configuration options.

Here is an excerpt from our profile::wmos2015 class:

```puppet
class { '::wmos2015::app':
    manage_jboss       => true,
    manage_jboss_group => true,
    jboss_version      => '6.3.3',
    java_home          => $java_home,
    manage_sd          => true,
    sd_package         => 'idg-wmos2015-sd',
}
```

And Hiera configuration:

```yaml
    wmos2015::app::oracle_home: '/opt/oracle-client12/product/12.1.0/client'
    wmos2015::app::sd_managed_servers:
      - thisserver.ourdomain
      - thatotherserver.ourdomain
    wmos2015::app::smtp_server: smtp.ourdomain
    wmos2015::app::user_wmos_pass: ENC[...
    wmos2015::app::user_wmosrf_pass: ENC[...
    wmos2015::app::sd_username: system
    wmos2015::app::sd_pass: ENC[...
    wmos2015::app::pcmws_pass: ENC[...
    wmos2015::app::db_host: ourdbhost
    wmos2015::app::db_port: 1521
    wmos2015::app::db_user_mda: MDA15
    wmos2015::app::db_pass_mda: ENC[...
    wmos2015::app::db_user_wms: WMS15
    wmos2015::app::db_pass_wms: ENC[...
    wmos2015::app::distrib_host: ourdistributionhost
    wmos2015::app::distrib_user: wmos
    wmos2015::app::distrib_pass: ENC[...
    wmos2015::app::distrib_proto: sftp
    wmos2015::app::distrib_location: /manh/software/distribution/WMOS/RU4
    wmos2015::app::distrib_version: 2015.0.4
    wmos2015::app::dshbd_heap: 2048
    wmos2015::app::dshbd_perm: 512
    wmos2015::app::mda_heap: 1024
    wmos2015::app::mda_perm: 256
    wmos2015::app::mip_heap: 1024
    wmos2015::app::mip_perm: 256
    wmos2015::app::mrf_heap: 1024
    wmos2015::app::mrf_perm: 512
    wmos2015::app::wm_heap: 4096
    wmos2015::app::wm_perm: 1024
    wmos2015::app::db_ar_host: ''
    wmos2015::app::db_ar_port: 1521
    wmos2015::app::db_ar_sid: ''
    wmos2015::app::db_ar_user: ''
    wmos2015::app::db_ar_pass: ''
```

## Reference

### Classes

#### Public Classes

* `wmos2015::app` is your only interface to the module. See the source for the
  expected configuration options (it has comments, and typing information).
  
NB: there is a long list of parameters that is used to generate the .properties
files that configure installations made by System Director. This part is not
really well documented. You can see where they're used in
`modules/wmos2015/templates/opt/manh/scope/sd/data/saved-intall-prop/*.epp`, and
you should be able to match them to graphical input fields in the WMS
installation manual.

#### Private Classes

The files under wmos2015/manifests/app/ implement the listed features, usually
one per file. The names should be self-explicit and are not enumerated here.

## Limitations

* This module expects RHEL7 (with room for expansion for other OSes):
    * wmos2015::app::services will fail() if it doesn't have systemd.
    * The systemd detection itself expects an EL-based distribution and will
      need to be adapted otherwise.
* wmos2015::app has a weak dependency on the Oracle client (we didn't test it
  with another database, but it shouldn't be too difficult if Manhattan
  supports it).
* We expect that your sqlnet.ora is declared as a concat resource somewhere
  else.

## Disclaimer

The author of this module is *in no way associated with Manhattan*. I just
automated how we install WMOS, based on Manhattan's prerequisites.

There is absolutely no guarantee given, use this software at your own risk.

## Author and copyright

Copyright 2016, Idgroup. Published with permission.

Author: Thomas Equeter.

This software may be redistributed under the terms of the Apache 2.0 licence.
