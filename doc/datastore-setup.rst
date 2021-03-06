========================
Setting up the DataStore
========================


.. note:: The DataStore requires PostgreSQL 9.0 or later. It is possible to use the DataStore on versions prior to 9.0 (for example 8.4). However, the :meth:`~ckanext.datastore.logic.action.datastore_search_sql` will not be available and the set-up is slightly different. Make sure, you read :ref:`legacy_mode` for more details.

.. warning:: The DataStore does not support hiding resources in a private dataset.

1. Enable the extension
=======================

Since the DataStore is an optional extension, it has to be enabled separately. To do so, ensure that the ``datastore`` extension is enabled in your CKAN config file::

 ckan.plugins = datastore

2. Set-up the database
======================

.. warning:: Make sure that you follow the steps in `Set Permissions`_ below correctly. Wrong settings could lead to serious security issues.

The DataStore requires a separate PostgreSQL database to save the resources to.

List existing databases::

 sudo -u postgres psql -l

Check that the encoding of databases is ``UTF8``, if not internationalisation may be a problem. Since changing the encoding of PostgreSQL may mean deleting existing databases, it is suggested that this is fixed before continuing with the datastore setup.

Create users and databases
--------------------------

.. tip::

 If your CKAN database and DataStore databases are on different servers, then
 you need to create a new database user on the server where the DataStore
 database will be created. As in :doc:`install-from-source` we'll name the
 database user |database_user|:

 .. parsed-literal::

    sudo -u postgres createuser -S -D -R -P -l |database_user|

Create a database_user called |datastore_user|. This user will be given
read-only access to your DataStore database in the `Set Permissions`_ step
below:

.. parsed-literal::

 sudo -u postgres createuser -S -D -R -P -l |datastore_user|

Create the database (owned by |database_user|), which we'll call
|datastore|:

.. parsed-literal::

 sudo -u postgres createdb -O |database_user| |datastore| -E utf-8

Set URLs
--------

Now, uncomment the :ref:`ckan.datastore.write_url` and
:ref:`ckan.datastore.read_url` lines in your CKAN config file and edit them
if necessary, for example:

.. parsed-literal::

 ckan.datastore.write_url = postgresql://|database_user|:pass@localhost/|datastore|
 ckan.datastore.read_url = postgresql://|datastore_user|:pass@localhost/|datastore|

Replace ``pass`` with the passwords you created for your |database_user| and
|datastore_user| database users.

Set Permissions
---------------

.. tip:: See :ref:`legacy_mode` if these steps continue to fail or seem too complicated for your set-up. However, keep in mind that the legacy mode is limited in its capabilities.

Once the DataStore database and the users are created, the permissions on the DataStore and CKAN database have to be set. Since there are different set-ups, there are different ways of setting the permissions. Only **one** of the options should be used.

Option 1: Paster command
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This option is preferred if CKAN and PostgreSQL are on the same server.

To set the permissions, use this paster command after you've set the database URLs (make sure to have your virtualenv activated):

.. parsed-literal::

 paster datastore set-permissions postgres -c |development.ini|

The ``postgres`` in this command should be the name of a postgres
user with permission to create new tables and users, grant permissions, etc.
Typically this user is called "postgres". See ``paster datastore
set-permissions -h``.

Option 2: Command line tool
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This option should be used if the CKAN server is different from the database server.

Copy the content from the ``datastore/bin/`` directory to the database server. Then run the command line tool ``datastore_setup.py`` to set the permissions on the database. To see all available options, run::

 python datastore_setup.py -h

Once you are confident that you know the right names, set the permissions
(assuming that the CKAN database is called |database| and the CKAN |postgres|
user is called |database_user|):

.. parsed-literal::

 python datastore_setup.py |database| |datastore| |database_user| |database_user| |datastore_user| -p postgres


Option 3: SQL script
~~~~~~~~~~~~~~~~~~~~

This option is for more complex set-ups and requires understanding of SQL and |postgres|.

Copy the ``set_permissions.sql`` file to the server that the database runs on. Make sure you set all variables in the file correctly and comment out the parts that are not needed for you set-up. Then, run the script::

 sudo -u postgres psql postgres -f set_permissions.sql


3. Test the set-up
==================

The DataStore is now set-up. To test the set-up, (re)start CKAN and run the
following command to list all resources that are in the DataStore::

 curl -X GET "http://127.0.0.1:5000/api/3/action/datastore_search?resource_id=_table_metadata"

This should return a JSON page without errors.

To test the whether the set-up allows writing, you can create a new resource in
the DataStore. To do so, run the following command:: 

 curl -X POST http://127.0.0.1:5000/api/3/action/datastore_create -H "Authorization: {YOUR-API-KEY}" -d '{"resource_id": "{RESOURCE-ID}", "fields": [ {"id": "a"}, {"id": "b"} ], "records": [ { "a": 1, "b": "xyz"}, {"a": 2, "b": "zzz"} ]}'

Replace ``{YOUR-API-KEY}`` with a valid API key and ``{RESOURCE-ID}`` with a
resource id of an existing CKAN resource.

A table named after the resource id should have been created on your DataStore
database. Visiting this URL should return a response from the DataStore with
the records inserted above::

 http://127.0.0.1:5000/api/3/action/datastore_search?resource_id={RESOURCE_ID}

You can now delete the DataStore table with::

    curl -X POST http://127.0.0.1:5000/api/3/action/datastore_delete -H "Authorization: {YOUR-API-KEY}" -d '{"resource_id": "{RESOURCE-ID}"}' 

To find out more about the DataStore API, go to :doc:`datastore-api`.


.. _legacy_mode:

Legacy mode: use the DataStore with old PostgreSQL versions
===========================================================

.. tip:: The legacy mode can also be used to simplify the set-up since it does not require you to set the permissions or create a separate user.

The DataStore can be used with a PostgreSQL version prior to 9.0 in *legacy mode*. Due to the lack of some functionality, the :meth:`~ckanext.datastore.logic.action.datastore_search_sql` and consequently the :ref:`datastore_search_htsql` cannot be used. To enable the legacy mode, remove the declaration of the ``ckan.datastore.read_url``.

The set-up for legacy mode is analogous to the normal set-up as described above with a few changes and consists of the following steps:

1. Enable the extension
2. The legacy mode is enabled by **not** setting the ``ckan.datastore.read_url``
#. Set-Up the database

    a) Create a separate database
    #) Create a write user on the DataStore database (optional since the CKAN user can be used)

#. Test the set-up

There is no need for a read-only user or special permissions. Therefore the legacy mode can be used for simple set-ups as well.
