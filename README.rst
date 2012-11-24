Setting up foreman
------------------

For simplicity we're going to assume a single foreman server. This will include
a smart proxy, DHCP, TFTP and DNS. We also assume we have control over our
**example.org** domain. So let's call this server **manager.example.org**.
For this tutorial I'm going to assume a plain CentOS 6 install with EPEL.

Installing foreman
==================

For this part we'll use the foreman-installer_. The content of this chapter is
also available on the `foreman installer wiki`_. We'll use git to retrieve the
installer and then write our preferences in *manager.pp*.

.. code-block:: sh

        yum -y install git puppet
        git clone --recursive https://github.com/theforeman/foreman-installer

Now we have the installer you can look at the various options available. Good
starting points are init.pp and params.pp files. Now let's write our
*manager.pp* file.


.. code-block::

        class {'puppet':
        } ~>

        class {'puppet::server':
          git_repo => true,
        } ~>

        class {'foreman_proxy':
        } ~>

        package {'foreman-postgresql':
          ensure => installed,
        } ~>

        class {'foreman':
          storeconfigs   => true,
          authentication => true,
          use_sqlite     => false,
        }

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

        su - -s /bin/bash foreman -c 'RAILS_ENV=production rake -f /usr/share/foreman/Rakefile db:migrate'

.. _foreman-installer: https://github.com/theforeman/foreman-installer
.. _foreman installer wiki: http://theforeman.org/projects/foreman/wiki/Using_Puppet_Module_ready_to_use
