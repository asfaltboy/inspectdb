Inspect DB
==========

A quick look into Django's `inspectdb` command when used on a table without a primary key, inspired by a `StackOverflow question`__.

__ question_

Lets start with a simple example, create a table and insert a record:

.. code-block:: sql

    $ python manage.py dbshell

    > CREATE TABLE connections(src_MAC CHAR(17) NOT NULL, first_activity DATETIME NULL);
    > INSERT INTO connections VALUES ('abcd1234',date('now'));

Now let's inspect the db with Django:

.. code-block:: bash

    $ python manage.py inspectdb

    # This is an auto-generated Django model module.
    # You'll have to do the following manually to clean this up:
    #   * Rearrange models' order
    #   * Make sure each model has one field with primary_key=True
    #   * Make sure each ForeignKey has `on_delete` set to the desired behavior.
    #   * Remove `managed = False` lines if you wish to allow Django to create, modify, and delete the table
    # Feel free to rename the models, but don't rename db_table values or field names.
    from __future__ import unicode_literals

    from django.db import models


    class Connections(models.Model):
        src_mac = models.CharField(db_column='src_MAC', max_length=17)  # Field name made lowercase.
        first_activity = models.DateTimeField(blank=True, null=True)

        class Meta:
            managed = False
            db_table = 'connections'


Now we add this model to our app, and the app to the INSTALLED_APPS. But, as the output notes say, a primary_key is required, without it, Django will raise an error, if we run a query:

.. code-block:: bash

    $ python manage.py shell

    >>> from testapp.models import *
    >>> Connections.objects.all()
    Traceback (most recent call last):
         ...
    OperationalError: no such column: connections.id


Let's just add a PK and make table acceptable in Django right? Alas, not so easy:

.. code-block:: sql

    $ python manage.py dbshell

    > ALTER TABLE connections add column id integer NOT NULL PRIMARY KEY AUTOINCREMENT;
    Error: Cannot add a PRIMARY KEY column


Coincidentally, if we may want to create a migration that tries to adds the field. A proper migration would look somewhat like this:

.. code-block:: python


    class Migration(migrations.Migration):

        dependencies = [
            ('testapp', '0001_initial'),
        ]

        operations = [
            migrations.AddField(
                model_name='connections',
                name='id',
                field=models.AutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID'),
            ),
        ]

Unfortunately, in SQLite it pass-fails silently (no field created, no errors raised)

So for SQLite we're out of luck. Our best bet would be to dump the table data, drop the table, create a new table with a primary key and finally reload the table data.
Frankly, this seems also to be the safest way to do it generally.

Now, what if we use a more sofisticated database such as PostgreSQL ? Sadly, the migration to add the field fails again, this time
the primary key field **is created**; however, it is missing the auto-increment property.

Luckily altering or creating the required field in PostgreSQL is easy:

.. code-block:: sql

    # create the field as required:
    > ALTER TABLE connections ADD COLUMN id SERIAL PRIMARY KEY;

    # Or, if you prefer to update the previously created field
    > ALTER TABLE connections ALTER id SET default nextval('connections_id_seq');


.. _question: http://stackoverflow.com/questions/38232364/django-model-tries-to-auto-create-a-primary-key-field-even-though-it-is-already
