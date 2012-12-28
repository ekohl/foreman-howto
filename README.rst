Setting up foreman
------------------

For simplicity we're going to assume a single foreman server. This will include
a smart proxy, DHCP, TFTP and DNS. We also assume we have control over our
**example.org** domain. So let's call this server **manager.example.org**.
We're going to use **192.168.100.0/24** with **192.168.100.1** as gateway. Our
manager will also be the recursing DNS server.

For this tutorial I'm going to assume a plain CentOS 6 install with EPEL, but
it should be easily applied to Debian as well.

It should be noted that foreman-installer_ is still very much in development
and things are going to change.

Installing foreman
==================

For this part we'll use the foreman-installer_. The content of this chapter is
also available on the `foreman installer wiki`_. We'll use git to retrieve the
installer and then write our preferences in *manager.pp*.

.. code-block:: sh

        yum -y install git puppet
        git clone --recursive -b develop https://github.com/theforeman/foreman-installer

Now we have the installer you can look at the various options available. Good
starting points are init.pp and params.pp files. Now let's write our
*manager.pp* file. To help you get started here's an example.


.. code-block:: puppet

        class {'puppet':
        } ~>

        class {'puppet::server':
          git_repo => true,
        } ~>

        class {'foreman_proxy':
          dhcp => true,
          dns  => true,
        } ~>

        package {'foreman-postgresql':
          ensure  => installed,
          require => Yumrepo['foreman_proxy'],
        } ~>

        class {'foreman':
          storeconfigs   => true,
          authentication => true,
          use_sqlite     => false,
        }

Now we have a *manager.pp* we can apply.

.. code-block:: sh

        puppet apply manager.pp --modulepath foreman-installer

It should be mentioned that there is work underway to replace this *manager.pp*
solution with an answer file. The git clone method will be replaced by
packages.

The database
============

Personally I prefer setting it up with postgresql. We're going to install
postgresql, initialize and start it.

.. code-block:: sh

        yum -y install postgresql-server
        service postgresql initdb
        service postgresql start
        chkconfig postgresql on

Now that we have postgres installed and running it's time to create a foreman
user and database. By default all management of postgres happens as the
postgres user so we're first going to change user. Note that we enter no
password. This means postgres will check the actual user logging in. This only
works because we'll use the socket on the same server.

.. code-block:: sh

        su - postgres -c 'createuser --no-createdb --no-createrole --no-superuser foreman'
        su - postgres -c 'createdb -O foreman foreman'

We just need to configure the database by editing */etc/foreman/database.yml*
and modify our *production* environment. Note that you don't need to remove
other options already in place.

.. code-block:: yaml

        production:
          adapter: postgresql
          database: foreman

Last but not least is the initialization.

.. code-block:: sh

        su - -s /bin/bash foreman -c 'RAILS_ENV=production bundle exec rake -f /usr/share/foreman/Rakefile db:migrate'

Setting up the puppet environment
=================================

Since we've told foreman-installer that we want a git repository it has
initialized one for us in */var/lib/puppet/puppet.git*. Each branch will be
converted into a puppet environment. The default branch is specified in HEAD
and defaults to master.

Configuring using the webinterface
==================================

We should now have a basic running system. Just go to
http://manager.example.org/ and check it out. In case you set up credentials
the default user is *admin*, but be sure to change the password from *changeme*
to something a little bit more secure.

First thing we're going to do is add our smart proxy. Navigate to *More* =>
*Smart Proxies* and click the *New Proxy*-button. Enter the name and URL. I
recommend calling it manager and connect it to http://localhost:8443/. After
it's added verify it has all the features you want. You should also be able to
import your DHCP subnet here.

With this smart proxy we can import our puppet classes. Navigate to *More* =>
*Puppet Classes* and click the *Import from manager*-button. It should detect
all your puppet classes and environments.

In order to install new servers we need to specify at least one architecture.
Again under *More* we have *Architectures* which in turn has a *New
Architecture*-button. I only have *x86_64* but maybe you have *i386* or more
exotic architectures.

With architectures set up we'll continu by adding operating systems. By now I
expect you'll find the *New Operating System*-button yourself. I also modified
the mirror under *Installation Media* to one that's a bit closer.

Setting up a domain and subnet should be straightforward as well.

Last you'll need to configure *Provisioning templates*.

Beyond the defaults
-------------------

Defaults are nice, but they're unlikely to fit everyone's needs.

Using a different network
=========================

While 192.168.100.0/24 may be a good place to start, it might not fit everyone.
In this example we're switching to **10.0.0.0/24** where we'll use
**10.0.0.50** to **10.0.0.200**. In this network we also have two other
recursors, **10.0.1.2** and **10.0.1.3**. It just comes down to changing our
foreman_proxy definition.

.. code-block:: puppet

        class {'foreman_proxy':
          dhcp             => true,
          gateway          => '10.0.0.1',
          range            => '10.0.0.50 10.0.0.200',
          dhcp_nameservers => '10.0.1.2,10.0.1.3',

          dns              => true,
          dns_reverse      => '0.0.10.in-addr.arpa',
        }

Using multiple networks
=======================

Suppose you have multiple networks on your smart proxy. We'll assume
**172.29.1.0/24** and no free lease. For this you need to configure an IP in
the range on some NIC. I'll assume you know how to do this yourself. Then we
only need to add another DHCP pool and DNS reverse range to our *manager.pp*:

.. code-block:: puppet

        dhcp::pool {'My extra DHCP pool':
          network => '172.29.1.0',
          mask    => '255.255.255.0',
          range   => false,
          gateway => '172.29.1.1',
        }

        dns::zone {'1.29.172.in-addr.arpa':
          reverse => true,
        }

That should give us another IP range we can use, including reverse DNS.

Bugs / missing features
=======================

While writing this document I ran into several bugs / missing features. This
section is also a TODO list for myself.

* Apache only listens on ipv4
* Setting up postgresql using puppet would be nice

Then there are also some points I want to expand in this document

* Setting up the puppet environment is a bit short
* Configuring using the webinterface only graces over domain, subnets and
  provisioning templates

.. _foreman-installer: https://github.com/theforeman/foreman-installer
.. _foreman installer wiki: http://theforeman.org/projects/foreman/wiki/Using_Puppet_Module_ready_to_use
