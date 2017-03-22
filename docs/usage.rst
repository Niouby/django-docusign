=====
Usage
=====

Configuration
-------------

In your ``settings.py`` :

.. code-block:: python

    INSTALLED_APPS = [
        # ...
        "django_north",
    ]

    NORTH_MANAGE_DB = True
    NORTH_MIGRATIONS_ROOT = '/path/to/sql/migrations/'
    NORTH_TARGET_VERSION = '1.42'

List of available settings:

* ``NORTH_MANAGE_DB``: if ``True``, the database will be managed by north.
  Default value ``False``
* ``NORTH_MIGRATIONS_ROOT``: a path to your migration repository. **required**
* ``NORTH_TARGET_VERSION``: the target SQL version
  (the version needed for your codebase). **required**
* ``NORTH_SCHEMA_TPL``: default value ``schema_{}.sql``
* ``NORTH_FIXTURES_TPL``: default value ``fixtures_{}.sql``
* ``NORTH_ADDITIONAL_SCHEMA_FILES``: list of sql files to load before the schema.
  For example: a file of DB roles, some extensions.
  Default value; ``[]``

In production environnements, ``NORTH_MANAGE_DB`` should be disabled, because
the database is managed directly by the DBA team (database as a service).

Migration repository tree example:

.. code-block:: console

    1.0/
        manual/
            1.0-feature_a-020-dml.sql
        1.0-0-version-dml.sql
        1.0-feature_a-010-ddl.sql
    1.1/
        1.1-0-version-dml.sql
    2.0/
        2.0-0-version-dml.sql
    2.1/
        2.1-0-version-dml.sql
    fixtures/
        fixtures_1.0.sql
        fixtures_1.1.sql
        fixtures_2.0.sql
    schemas/
        schema_1.0.sql
        schema_1.1.sql
        schema_2.0.sql

See also some examples in ``tests/test_data/sql`` folder (used for unit tests),
or in ``tests/north_project/sql`` folder (used for realistic tests).

Available Commands
------------------

migrate
.......

.. code-block:: console

    $ ./tests_manage.py migrate

Create a DB from scratch and migrate it to the version defined in the
``NORTH_TARGET_VERSION`` setting, or update an existing DB to migrate it to
the correct version.

This command knows which migrations are already applied, which migrations
should be applied.

This command can only go forward: no possible revert like with south or django
migrations. But as the migrations written by the DBA team are blue/green, that
is not a problem !

This command has no effects if the ``NORTH_MANAGE_DB`` setting is disabled.

showfixtures
............

.. code-block:: console

    $ ./tests_manage.py showfixtures

List missing fixtures, and print SQL instructions to create them
(ask your DBA team to add a dml migration for that).

"Fixtures" designates here datas which are automatically created by django
on ``post_migrate`` signal, and required for the project.


Basically:

* content types (``django.contrib.contenttypes``)
* permissions (``django.contrib.auth``)

The site id 1 (``SITE_ID`` setting) is not checked by this command.

.. note::

    When you add a Model, you have to run this command twice to get:
    1/ the new content type
    2/ when the content type exists, the new permissions

showmigrations
..............

.. code-block:: console

    $ ./tests_manage.py showmigrations

List available migrations, and indicate if they where applied or not.

This command has no effects if the ``NORTH_MANAGE_DB`` setting is disabled.

Changed Commands
----------------

flush
.....

.. code-block:: console

    $ ./tests_manage.py flush

Did a truncate on all tables, where the original command did it only on tables
defined in the django models.

Reload the SQL fixtures, and reset the ContentType cache.

This command is essential for the tests, especially for TransactionTestCase tests.

This command has no effects if the ``NORTH_MANAGE_DB`` setting is disabled.

runserver
.........

.. code-block:: console

    $ ./tests_manage.py runserver

Display a warning if some migrations are not applied.

Disabled Commands
-----------------

These commands are disabled whatever the value of the ``NORTH_MANAGE_DB`` setting:

* ``makemigrations``
* ``sqlmigrate``
* ``squashmigrations``

Tips
----

Output SQL commands
...................

For example, if you want to use the ``sqlall`` command to prints the CREATE TABLE,
custom SQL and CREATE INDEX SQL statements for the init of a DB schema, for an
external app with a migration folder (as ``django.contrib.auth`` app for example),
put this in your ``settings.py`` :

.. code-block:: python

    class DisableMigrations(object):

        def __contains__(self, item):
            return True

        def __getitem__(self, item):
            return "notmigrations"


    MIGRATION_MODULES = DisableMigrations()

Then run the ``sqlall command``:

.. code-block:: console

    $ ./tests_manage.py sqlall <app>

Generate Schema Files
.....................

At the end of a SQL release, just do a sqldump (``pg_dump -s`` for posgtres for example).
