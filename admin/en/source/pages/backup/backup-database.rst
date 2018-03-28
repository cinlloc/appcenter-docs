.. Copyright 2018 FUJITSU LIMITED

.. _backup-database:

UForge Databases Basic Backup
-----------------------------

UForge has three databases to store all its data, namely:

	* LDAP (OpenDJ) for storing user credential information
	* IDM (Syncope using MariaDB) for storing user entitlements and roles
	* UForge (MariaDB) for storing everything else for UForge features.

MariaDB provides many different types of backup methods you can choose from, including:

	* Doing a hot backup (for example with Percona XtraBackup)
	* Copying Table files
	* Delimited-Text backups
	* Using the mysqldump or mysqlhotcopy commands
	* Creating incremental backups by enabling the binary log
	* Creating a backup from a slave
	* Making backups using a file system snapshot

For more details on all these back up strategies, refer to: `http://dev.mysql.com/doc/refman/5.5/en/backup-methods.html <http://dev.mysql.com/doc/refman/5.5/en/backup-methods.html>`_

MariaDB holds both UForge and IDM data. All this information is located under:

	``/var/lib/mysql``

By copying this information, you are taking a snapshot of the entire database. 

LDAP (OpenDJ) has its own tool for doing backup, called ``backup``.  To ensure data consistency across the entire platform, we recommend you backup all the databases at the same time.  Here is an example of how to back up the MariaDB (containing IDM and UForge information) and LDAP databases.  Note, we are also stopping the web service instances; this ensures we do not generate connection error messages in the logs:

.. code-block:: shell

	$ service tomcat stop
	$ systemctl stop mysql
	$ f=$(date +%y%m%d)
	$ l="$f-ldap-backup.tgz" #what to name ldap backups
	$ f="$f-mysql-backup.tgz" #what to name backups
	$ tar -czf /tmp/$f /var/lib/mysql/*
	$ /opt/opendj/bin/backup -a -c -d /tmp/ldap_backup
	$ tar -czf /tmp/$l /tmp/ldap_backup
	$ systemctl start msql
	$ service tomcat start

If using SAN mountpoint, add:

.. code-block:: shell

	$ rsync /tmp/$f /<SAN-MOUNTPOINT-BACKUP-DESTINATION>
	$ rsync /tmp/$l /<SAN-MOUNTPOINT-BACKUP-DESTINATION>

If using remote backup, add:

.. code-block:: shell

	$ rsync /tmp/$f root@<BACKUP-DESTINATION>
	$ rsync /tmp/$l root@<BACKUP-DESTINATION>

.. warning:: This only works if the database instance is stopped to ensure a consistent dump of the database. Otherwise you get a corrupt, inconsistent backup.

The issue with this method is if you have only one database instance, then you are effectively stopping your service to do the backup. To overcome this you need to use replication, for example master-slave.

.. note:: For LDAP, the service does not need to be stopped.


.. _basic-restore:

Basic Restore
-------------

Restoring the database is a simple copy using rsync back to the database directory. Note the use of slashes (/ ) is important.  You should stop the web service to avoid creating connection error messages in the logs.

.. code-block:: shell

	$ service tomcat stop
	$ service OpenDJ stop
	$ systemctl stop mysql
	$ cd /tmp
	$ tar -xvf /tmp/<ldap tarball>
	$ /opt/opendj/bin/restore -d /tmp/<ldap dir>/userRoot
	$ /opt/opendj/bin/restore -d /tmp/<ldap dir>/tasks
	$ /opt/opendj/bin/restore -d /tmp/<ldap dir>/config
	$ /opt/opendj/bin/restore -d /tmp/<ldap dir>/schema
	$ tar -xvf /tmp/<mysql tarball>
	$ rsync -a --delete-before /tmp/<mysql dir>/* /var/lib/mysql/
	$ service OpenDJ start
	$ systemctl start mysql

To restore the SAN mountpoint:

.. code-block:: shell

	$ rsync /<SAN-MOUNTPOINT-BACKUP-DESTINATION>/<ldap tarball> /tmp
	$ rsync /<SAN-MOUNTPOINT-BACKUP-DESTINATION>/<mysql tarball> /tmp

To restore remote backup:

.. code-block:: shell

	$ rsync root@<BACKUP-DESTINATION>/<ldap tarball> /tmp
	$ rsync root@<BACKUP-DESTINATION>/<mysql tarball> /tmp

