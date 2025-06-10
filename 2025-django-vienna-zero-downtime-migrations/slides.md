---
# https://github.com/estruyf/slidev-theme-the-unnamed
theme: the-unnamed
# Metadata
title: "Achieving Zero-Downtime Migrations in High-Traffic Django Systems"
info: |
  Everybody knows how customers love downtime. But sometimes you feel like it cannot be avoided.
# https://sli.dev/features/drawing
drawings:
  enabled: false
# https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
layout: cover
---

# Achieving Zero-Downtime Migrations in High-Traffic Django Systems

Created by [Kamil Marut](https://kamilmarut.com)

<div class="flex flex-col justify-center items-end gap-2 pb-4">
    <span class="text-sm">If they did it, so can we!</span>
    <img width="100" class="invert" alt="https://www.youtube.com/watch?v=vQ5MA685ApE" src="./assets/inspiration_qr_code.png" />
</div>

<!--
Hi everybody, my names is Kamil. I'm sure everybody here knows how customers love downtime. But sometimes you feel like it cannot be avoided. There's an old video of how a couple of German dudes managed to move their server across Berlin using public transport without shutting it down. I will try to show you that if those guys did it, you can do it too.

I am a firm believer that when designing the architecture of a new application, you should spend the majority of your time thinking about data structures. 
Data structures naturally dictate algorithms. They guide your instinct. Nearly every feature starts by extending the database schema.
You need to think about how the data will be used, how it will be queried, and how it will be stored. 

In the end you will end up with a schema that is not only a representation of your data but also a blueprint for your application.
But a blueprint is not a final product. It is a starting point. A plan. At some point the requirements will change, the data will change, and the schema will change.
-->

---
layout: default
---

# The migration hell

<div class="flex flex-col justify-center items-center">
  <img src="./assets/sisyphus.jpeg" alt="Sisyphus rolling the stone up the mountain" class="w-1/2 invert"/>
</div>

<!-- 
Enter the migration hell. Where the database schema is trying to catch up with the ever-changing requirements, and you are trying to keep your sanity while doing so. And this what this talk is about.
-->

---
layout: default
---

# Challenges around schema changes

<ul>
    <v-click><li><strong>Locking nightmares:</strong> Schema changes can block reads and writes, causing unexpected downtime.</li></v-click>
    <v-click><li><strong>Environment mismatches:</strong> Migrations that work fine in dev/staging but crash in prod.</li></v-click>
    <v-click><li><strong>Additive-only changes:</strong> Adding fields to avoid breaking anything, but never cleaning up, leading to bloated, messy schemas.</li></v-click>
</ul>

<!-- 
So, what problems do we face when changing the schema?
The dreaded "error: deadlock detected" or "database is locked" errors.
Data migrations that work fine in development but fail in production due to different data volumes or configurations.
And after you felt the pain of the first two, you get scared and start with the additive-only change pattern, where you keep adding new fields to your models without ever cleaning up the old ones.

I didn't mention it in the slide, but something similar to additive-only change pattern is also abusing the JSONField to persist all of the data. Please don't do this. There's a reason why you're using a relational database in the first place.
-->

---
layout: default
---

# Assumptions for a Django app that strives for zero-downtime migrations

- **Multiple instances:** You have multiple instances of your application running behind a load balancer.
- **One Postgres database:** You have a single PostgreSQL database that is shared by all your services.
- **Django migrations:** You are using and want to keep using the Django migration system.

<!-- 
Multiple instances or at the very least the option to spin up a new instance before destroying the old one. Otherwise the downtime is inevitable.

If you sharded your database, you probably figured your migration strategy out already. If you didn't, you're on your own, pal.

This talk is only focused on the built-in Django migration system. If you are using a different migration system (your own or a third-party like Alembic) then your system is probably free of some of the Django-specific problems as well as it might suffer from problems that Django solved.

We also don't want to switch to another migration system. There are cool tools that automate the migration process, like pgroll, but rarely can they actually be implemented in an active production environment.
-->

---
layout: center
---

<div class="flex flex-col gap-8">
  <img src="./assets/pgroll.png" alt="pgroll banner" class="w-full invert"/>
  <a class="w-1/4 mx-auto text-center" href="https://github.com/xataio/pgroll">github.com/xataio/pgroll</a>
</div>

<!--
Nevertheless, you should check out pgroll. Why listen about Django migrations for 45 minutes when you can just use something that does 99% of the work for you? 

Of course it feels underpowered compared to Django migrations and the syntax is not as nice, but it does make it easier to avoid making breaking changes. 
-->

---
layout: default
---

# The naive approach

1. Alter the schema in a way that breaks existing code.
2. Deploy code that only works with the new schema.
3. Lock the table for the duration of the migration.
4. Make the application unresponsive for a long time.
5. Get scolded by your manager for losing money.

---
layout: default
---

# The ideal approach

1. Perform schema changes in a way that won't break existing code.
2. Deploy code that works with both the old and new schema simultaneously, populating any new rows according to the new schema.
3. Perform a data migration that backfills any old data correctly.
4. Optionally, add constraints like NOT NULL to the new column once all the data is backfilled.
5. Deploy code that only expects to see the new schema.
6. Drop the old column.

<!-- 
What do you need in order do it the ideal way?
You need time. Preferably you are used to doing continuous deployments and you have a good CI/CD pipeline that allows you to deploy code quickly.

1. For example, by temporarily allowing new non-nullable columns to be NULL. More on that later.

Why call it the ideal approach? Because rarely anybody has the luxury to do it the ideal way all the time. Usually you have to make compromises.
Sometimes you deliberately take shortcuts to save development time, sometimes you just can't spin up a sidecar container to run the migration in the background.
-->

---
layout: default
---

# Migration operations

<div class="flex flex-col justify-center items-center">
  <img src="./assets/migration_operations.jpeg" alt="Migration operations" class="w-3/4 invert"/>
</div>

<!--
Now we get to the actual practical part of the talk.
-->

---
layout: default
---

# Adding a new field to the schema

When adding a field, always ensure one of the following:

* field is nullable
* field has a default value

```python
operations = [
    migrations.AddField(
        model_name="invoice",
        name="invoice_number",
        field=models.CharField(blank=True, default="", max_length=512),
    ),

    # or use `db_default=""` in Django 5.0 or later
    migrations.RunSQL(
        "ALTER TABLE payment_invoice ALTER COLUMN invoice_number SET DEFAULT '';", 
        migrations.RunSQL.noop,
    ),
]
```

<!--
Please note that Django doesn't propagate default values onto the database; that's why you need to add an SQL statement that would manually set the default value on existing rows.

In Django 5.0 or later, you can use the `db_default` argument to set the default value on the database level.
-->

---
layout: default
---

# Renaming field

* Add a new field with the new name
* Deploy code that writes data to both fields
* Backfill the new field with data from the old field
* Deploy code that only writes to the new field
* Drop the old field

---
layout: default 
---

# Change field type

For example, changing a `CharField` to an `IntegerField`, you need to combine the steps of adding a new field without a default value and renaming the field.

---
layout: default
---

# Removing a field

Separate database and state by dropping the field from the ORM and ensure that the database field is nullable.

```python
operations = [
    migrations.SeparateDatabaseAndState(
        database_operations=[
            migrations.AlterField(
                model_name="example",
                name="old_field",
                field=models.CharField(
                    blank=True, null=True
                ),
            ),
        ],
        state_operations=[
            migrations.RemoveField(model_name="example", name="old_field"),
        ],
    )
]
```
<!--
This allows rollback if needed before dropping the actual column.
-->

---
layout: default
---

# Adding an index

Creating an index can lock the table for several hours. To avoid such a scenario you need to create an index concurrently.

```python
from django.db import migrations
from django.db.models import Index
from django.contrib.postgres.operations import AddIndexConcurrently

class Migration(migrations.Migration):
    atomic = False

    operations = [
        AddIndexConcurrently(
            model_name="user",
            index=Index(fields=["city_id"], name="account_user_city_id_index")
        ),
    ]
```

<!--
Please note that the line `atomic = False` is needed to proceed with concurrent index creation. 

Please don't use non-atomic migrations for other operations.
-->

---
layout: default
---

# Removing an index

Similar to adding an index, you need to drop the index concurrently.

```python
from django.db import migrations
from django.contrib.postgres.operations import RemoveIndexConcurrently

class Migration(migrations.Migration):
    atomic = False

    operations = [
        RemoveIndexConcurrently(
            model_name="user",
            name="account_user_city_id_index"
        )
    ]
```

---
layout: default
---

# Adding a foreign key field

Adding a `ForeignKey` in Django creates an index on that field which is not specified explicitly in the model definition.
This index will be created in a blocking way, which we want to avoid as already mentioned.

```python
class User(models.Model):
    city = models.ForeignKey(
        City,
        blank=True,
        null=True,
        on_delete=models.SET_NULL,
        db_index=False,
    )

    class Meta:
        indexes = [
            models.Index(fields=["city"], name="account_user_city_id_index")
        ]
```

<!--
To create the index concurrently, you need to disable the creation of the index by setting `db_index=False` and move the creation of the index to `Meta.indexes`, like on the slide.
And with this, you can use `AddIndexConcurrently`. Migration containing the `ForeignKey` creation should be separated from migration that creates an index concurrently.
-->

---
layout: default
---

# Data migrations

* Use a separate process to perform data migrations (Jupyter notebook, script)
* Schedule a Celery task from within the migration file

```python
from django.db import migrations
from django.db.models.signals import post_migrate
from django.apps import apps
from .tasks import migration_task

def migration(apps, schema_editor):
    def on_migrations_complete(sender=None, **kwargs):
        migration_task.delay()

    sender = apps.get_app_config("your_app_name")
    post_migrate.connect(on_migrations_complete, weak=False, sender=sender)

class Migration(migrations.Migration):
    dependencies = []

    operations = [
        migrations.RunPython(migration, migrations.RunPython.noop)
    ]
```

<!-- 
Depending on your infrastructure, you might want to avoid running data migrations in the actual migration file with `migrations.RunPython`.
Instead, you can run them as a separate script, in a Jupyter notebook or in a Celery task.

(in case of weak=False question)
Django stores signal handlers as weak references by default. Thus, if your receiver is a local function, it may be garbage collected. To prevent this, pass weak=False when you call the signal’s connect() method.
-->


---
layout: default
---

# Remember to make your migrations reversible

<div class="flex flex-row justify-center items-center">
  <img src="./assets/schema_changes_rollback_choice.jpg" alt="Rollback schema change or add code workaround? A tough choice to make at 3am." class="w-2/7 invert"/>
</div>

<!--
If there's one thing scarier than performing schema changes, it's the thought that you might have to undo them under time constraints.

Without backward compatibility, you can not have blue-green deployments.

As a nice side-effect, keeping the migrations backward compatible helps you switching branches during development. 
-->

---
layout: default
---

# Let robots help you

Detect backward incompatible migrations with tools like [django-migration-linter](https://github.com/3YOURMIND/django-migration-linter).

```
$ python manage.py lintmigrations
(accommodations, 0001_initial)... OK (cached)
(accommodations, 0002_accommodation_area_accommodation_pet_friendly)... ERR (cached)
        NOT NULL constraint on columns
(accommodations, 0003_alter_accommodation_image_field_1_and_more)... OK (cached)
...
(emails, 0004_remove_emailtemplate_unique_kind_accommodation_and_more)... ERR (cached)
        DROPPING columns (table: emails_emailtemplate, column: is_global)
        ADDING unique constraint
        DROP INDEX locks table
        DROP INDEX locks table
...
*** Summary ***
Valid migrations: 28/44
Erroneous migrations: 14/44
Migrations with warnings: 2/44
Ignored migrations: 0/44
```

<!--
This is an actual output of one of my Django projects. As you can see, it's run as a regular Django management command.
It scans all of the migrations for backward incompatible changes. As you can see, I have added a non-nullable field in a migration.
In another migration I dropped a column and added a unique constraint, which might be ok and then we would ignore it.
Otherwise, the summary says that I am a complete buffoon with 14 erroneous migrations.
Fortunately, this is not a production ready application and I'm going to squash the migrations later.
-->

---
layout: default
---

# Migrations are code too - test them!

Using libraries likes [django-test-migrations](https://github.com/wemake-services/django-test-migrations) you can test how your migrations behave in different database state scenarios.

Works with both `unittest` and `pytest`.

---
layout: default
---

# BONUS: Downtime without any downtime

Hide your downtime behind arbitrary stalled traffic. If you are confident your migration will complete within a short window (e.g. <60 seconds) if no users will hit your application, you can try to suspend traffic at the load balancer (doable in HAProxy, Traefik), run migrations and then resume traffic.

<!--
As a bonus, I want to share with you a trick that I've heard about recently. Depending on how you define downtime, you can actually stop accepting user traffic, perform a migration that locks the table and then resume traffic.

This is not truly zero downtime for end-users; it's just hidden.
-->

---
layout: default 
---

# References

* [github.com/tbicr/django-pg-zero-downtime-migrations](https://github.com/tbicr/django-pg-zero-downtime-migrations)
* [github.com/yandex/zero-downtime-migrations](https://github.com/yandex/zero-downtime-migrations)
* [Zero Downtime Migrations](https://docs.saleor.io/developer/community/zero-downtime-migrations)
* [github.com/wemake-services/django-test-migrations](https://github.com/wemake-services/django-test-migrations)
* [github.com/3YOURMIND/django-migration-linter](https://github.com/3YOURMIND/django-migration-linter)
* [<mark>Internet Archive</mark> How Balanced does Database Migrations with Zero-Downtime](https://web.archive.org/web/20181007025039/https://blog.balancedpayments.com/payments-infrastructure-suspending-traffic-zero-downtime-migrations/)

---
layout: center
---

<div class="flex flex-row gap-8">
<div class="w-1/2">
  <h1>Thank you!</h1>

  <div class="flex flex-col gap-4 mt-4">
  <a href="https://kamilmarut.com">
    <span>kamilmarut.com</span>
  </a>

  <a href="https://github.com/exler">
    <carbon:logo-github />
    <span class="pl-1">github.com/exler</span>
  </a>

  <a href="https://linkedin.com/in/kamilmarut">
    <carbon:logo-linkedin />
    <span class="pl-1">linkedin.com/in/kamilmarut</span>
  </a>
  </div>
</div>

<div class="flex flex-row justify-center items-center w-full">
  <img src="./assets/deadlock.jpg" alt="I: Explain us deadlock and we'll hire you. Me: Hire me and I'll explain it to you." class="w-4/5 invert"/>
</div>
</div>
