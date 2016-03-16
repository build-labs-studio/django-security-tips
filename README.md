# Django and PostgreSQL security tips and practices

## Purpose and motivation

The aim of this guide/repository is to learn and promote secure system administration tips and practices in the Django community.
My motivation is that most articles that focus on getting a Django application up and running do not talk much about security, yet for example database security guides often feel too abstract and intimidating for newcomers.
So let's bridge that gap!
By pinning down PostgreSQL as the database I do not mean to discourage the use any other system -- it's just that I want to provide concrete copy-pastable commands, so a choice has to be made.

The scope of the guide is yet to be defined and will depend on the people who will get involved.
Your questions, feedback, and insight well be very welcome!

Future topics should include at least monitoring and deployment practices.

## Before we begin..

.. Make sure you have read the
[Django Deployment Checklist](https://docs.djangoproject.com/en/dev/howto/deployment/checklist/)
and
[Security in Django](https://docs.djangoproject.com/en/dev/topics/security/).

This skeleton has been created by commands
```sh
django-admin startproject playproject
django-admin startapp plaything
```

## Database roles, schemas, and migrations

PostgreSQL has something called [schemas](http://www.postgresql.org/docs/current/static/ddl-schemas.html), which are a bit like folders in the file system.
By default, all tables are created in a schema called `public` where all new users/roles have all permissions.
However, it is advisable to confine your web application to a specific schema and grant it as few privileges as possible.

To get started, let's create a database and log into it. If you don't want to type your password every time, create a [password file](http://www.postgresql.org/docs/current/static/libpq-pgpass.html).

```sh
sudo su postgres
createdb playdb && psql playdb
```

We will have no need for the public schema, so let's drop it so that we don't have to think about it anymore.
(Don't do this on an existing database!)

```sql
DROP SCHEMA public CASCADE;
```

We'll have two roles, `djangouser` and `djangomigrator`.
The `djangouser` will be used by your application in production, and `djangomigrator` will be used to perform migrations.
The `djangouser` will need permissions to select, insert, delete, and update rows on all the tables.
In addition, she'll need to access the [sequences](http://www.postgresql.org/docs/current/static/functions-sequence.html) to calculate the id of new model instances.
The lesson to learn is that she should have *only* those privileges, and not be able to for example create [trigger procedures](http://www.postgresql.org/docs/current/static/plpgsql-trigger.html).

As the `djangouser` will not be able to create or alter tables, we'll need another role for that purpose, `djangomigrator`, who will be the owner of the schema `playschema` of your django project `playproject`.
In addition, we'll set the [search path](http://www.postgresql.org/docs/current/static/runtime-config-client.html) of both users to `playschema`.

```sql
CREATE ROLE djangomigrator LOGIN ENCRYPTED PASSWORD 'migratorpass';
CREATE ROLE djangouser LOGIN ENCRYPTED PASSWORD 'userpass';
CREATE SCHEMA playschema AUTHORIZATION djangomigrator;
GRANT USAGE ON SCHEMA playschema TO djangouser;
ALTER ROLE djangouser SET SEARCH_PATH TO playschema;
ALTER ROLE djangomigrator SET SEARCH_PATH TO playschema;
```

(or try `psql -d playdb -f roles.sql` in this repo)

In order to juggle between these two roles you can create a special settings file `migrator_settings.py` for your migrator.
It's nothing more than

```python
from .settings import *
DATABASES['default']['USER'] = 'djangomigrator'
DATABASES['default']['PASSWORD'] = 'migratorpass'
```

and then you'll be able to run:

```sh
python manage.py migrate --settings=playproject.migrator_settings
```

By the way, make sure that `python manage.py migrate` really fails!
We are not quite done yet, though, because `djangouser` will, by default, not have any privileges on the newly created tables or sequences.
Before running `python manage.py runserver` you will have to say

```sql
GRANT SELECT, INSERT, DELETE, UPDATE ON ALL TABLES IN SCHEMA playschema TO djangouser;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA playschema TO djangouser;
```

Finally, you probably want the migrations process to be a single simple command. To do that, see for example the file `migrate.sh`.

## Database cluster firewall

Your database should be accessible only from certain IP address(es).
Even if you are not afraid of attackers, be afraid of yourself accidentally running `dropdb playdb` instead of `dropdb playdb_test` during a late night hacking session.

For example, on AWS, if you have a Virtual Private Cloud with CIDR 172.38.0.0/16 you can allow inbound TCP traffic on port 5432 from source 172.38.0.0/16.
That will allow connections from any machine in that VPC, so you may want to be even more restrictive.

## Multiple databases

(Disclaimer: make sure to understand the difference between database and database cluster.
A database cluster, e.g. your AWS RDS instance, can have multiple databases.
It's not uncommon for a role to be able to access more than one database.)

Privileges can be defined on a number of levels in PostgreSQL: e.g. row, table, schema, and database level.
All of them have their uses, but let's highlight the difference between database and other levels.

* An attacker can change the row, table, or schema with SQL commands,
* but to access a different database, a new connection has to be initiated.

So if something must be kept out of the reach of your web app but still in the same cluster, consider putting it in a different database.
Django makes this [reasonably easy](https://docs.djangoproject.com/en/dev/topics/db/multi-db/).

## Further reading

* [IBM develperWorks: Total security in a PostgreSQL database](http://www.ibm.com/developerworks/library/os-postgresecurity/)
* [OpenSCG: Security Hardening PostgreSQL](http://www.openscg.com/wp-content/uploads/2013/04/SecurityHardeningPostgreSQL.pdf)
* [OWASP: Backend Security Project PostgreSQL Hardening](https://www.owasp.org/index.php/OWASP_Backend_Security_Project_PostgreSQL_Hardening)

---
