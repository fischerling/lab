.. highlight:: console

.. spelling::
    zabbix

.. author:: Florian Fischer <florian.fischer@vocosma.it>

.. tag:: monitoring
.. tag:: audience-admins

##########
Zabbix
##########

.. tag_list::

Zabbix_ is a collection of open-source software tools to monitor IT infrastructure such as networks, servers, virtual machines, and cloud services.
Zabbix_ consists of the server collecting data and storing it in a database, the frontend displaying the stored data, as well as data sources like the zabbix agent.
In this guide we install the zabbix server as well as the frontend.

----

.. note:: For this guide you should be familiar with the basic concepts of

  * :manual:`supervisord <daemons-supervisord>`
  * :manual:`MySQL <database-mysql>`
  * :manual:`Firewall Ports <basics-ports>`

License
=======

Zabbix_ is licensed under the GNU General Public License version 2.

All relevant legal information can be found here

  * https://github.com/zabbix/zabbix/blob/master/COPYING

Prerequisites
=============

MySQL
-----

Zabbix_ supports multiple database backends, but this guide will use :manual:`MySQL <database-mysql>`.

.. warning::
  Zabbix only supports MySQL version >= 10.5 and uberspace 7.12 has only MariaDB 10.3 available. 

Create the database used by zabbix:

::

[isabell@stardust ~]$ mysql -e "CREATE DATABASE ${USER}_zabbix DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_bin;"

We will prepare the created database later on with the sql scripts included in the zabbix sources.

ListenPort
----------

To allow zabbix agents to connect to our server we have to open a port:

.. include:: includes/open-port.rst

Remember the opened port, because we must configure the server to use it and each agent connecting to the server.

Installation
============

.. note:: Check out the latest LTS release on https://www.zabbix.com/download_sources. We'll set a temporary environment variable for this session so all commands are independent from the actual zabbix version.

Download and extract the sources of the latest stable release:

::

[isabell@stardust ~]$ MAJOR_VERSION=6.0
[isabell@stardust ~]$ VERSION=${MAJOR_VERSION}.2
[isabell@stardust ~]$ wget https://cdn.zabbix.com/zabbix/sources/stable/${MAJOR_VERSION}/zabbix-${VERSION}.tar.gz
[isabell@stardust ~]$ tar --extract --gzip --file=zabbix-${VERSION}.tar.gz
[isabell@stardust ~]$ cd zabbix-${VERSION}

Patch, configure, build and install zabbix:

Because the uberspace firewall ports are commonly greater than 32767 which is the default allowed maximum of the zabbix listen port we have to patch the zabbix server sources to allow a bigger maximum port range (1024; 65535).

To allow ports > 32767 run:

::

[isabell@stardust zabbix-6.0.2]$ sed -i 'N;s/\(.*ListenPort.*\n.*\)32767\(.*\)/\165535\2/' src/zabbix_server/server.c


In this guide we will build zabbix_ server with TLS, SSH, IPv6 and MySQL support.
The resulting binaries are installed into ``~/bin/`` in our home directory.
Everything else (manpages, sample configuration, ...) is installed to ``~/.local/`` to prevent cluttering our home directories.

.. code-block::

 [isabell@stardust zabbix-6.0.2]$ ./configure --prefix=${HOME}/.local \
                                              --bindir=${HOME}/bin \
                                              --sbindir=${HOME}/bin \
                                              --sysconfdir=${HOME}/etc \
                                              --enable-server \
                                              --with-mysql \
                                              --with-openssl \
                                              --with-ssh2 \
                                              --enable-ipv6
 [isabell@stardust zabbix-6.0.2]$ make install

What the ``configure`` options mean:

* ``--prefix=${HOME}/.local``: Use ``~/.local`` as installation location.
* ``--{s,}bindir=${HOME}/bin``: Install all binaries to ``~/bin``.
* ``--enable-server``: Build zappix_server.
* ``--enable-ipv2``: Build zappix_Server with IPv6 support.
* ``--with-*``: Enable MySql, TLS and SSH support.

.. note::

    For a list of available configuration options run:
    ::

    [isabell@stardust zabbix-6.0.2]$ ./configure --help


Install the frontend files located in the ``ui/`` source subdirectory to our ``html/`` directory.
Set the permission so that the webserver is allowed to access the front end files.
For more information on the permissions have a look at the `manual <https://manual.uberspace.de/web-documentroot/#permissions>`_.

::

[isabell@stardust zabbix-6.0.2]$ cp -a ui/* ${HOME}/html/     # install frontend files
[isabell@stardust zabbix-6.0.2]$ chmod -R u=rwX,go=rX ~/html  # Set the correct permissions

Prepare the earlier created MySQL database to be used by zabbix:

::

[isabell@stardust zabbix-6.0.2]$ mysql ${USER}_zabbix < database/mysql/schema.sql
[isabell@stardust zabbix-6.0.2]$ mysql ${USER}_zabbix < database/mysql/images.sql
[isabell@stardust zabbix-6.0.2]$ mysql ${USER}_zabbix < database/mysql/data.sql

Cleanup
=======

Since we only need the now installed files we can safely remove the downloaded archive and the extracted sources.

::

[isabell@stardust zabbix-6.0.2]$ cd
[isabell@stardust ~]$ rm -r zabbix-${VERSION}
[isabell@stardust ~]$ rm zabbix-${VERSION}.tar.gz

Configuration
=============

Edit the configuration installed to ``~/etc/zabbix_server.conf``.
Set the listening port to the port opened previously by adding a line like this:

.. code-block:: ini

  ListenPort=port

Set the correct database name, user and password by searching for line containing one of the three strings and replacing the default values:

.. code-block:: ini

  DBName=your-username_zabbix
  DBUser=your-username
  DBPassword=your-mysql-password

Allow zabbix to use our not supported MariaDB version (10.3.0; Minimum: 10.5.0) by adding this line:

.. code-block:: ini

  AllowUnsupportedDBVersions=1

To use supervisord's logging functionality add the following line:

.. code-block:: ini

  LogType=console

Setup daemon
------------

Create the file ``~/etc/services.d/zabbix.ini`` with the following content:

.. code-block:: ini

  [program:zabbix]
  command=zabbix_server -f -c %(ENV_HOME)s/etc/zabbix_server.conf
  autostart=yes
  autorestart=yes

PHP recommended configuration
-----------------------------

To serve the Zabbix frontend certain PHP settings are recommended.
Create a new ``zabbix_php.ini`` in `~/etc/php.d/` with the settings suggested by Zabbix:

.. code-block:: ini

  max_execution_time=300
  max_input_time=300

After creating the configuration file reload php using:

::

[isabell@stardust ~]$ uberspace tools restart php

Finishing installation
======================

Start zabbix
------------

.. include:: includes/supervisord.rst

Finish the frontend configuration
---------------------------------

Now point your browser to your uberspace and you should see the zabbix webinterface initial setup page.

.. warning::
  The default admin user is called ``Admin`` with the password ``zabbix``.
  You should disable this user after creating your own admin user or at least change its password!


.. _Zabbix: https://zabbix.com/

----

Tested with Zabbix_ 6.0.2, Uberspace 7.12.1

.. author_list::
