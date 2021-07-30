---
layout: post
comments: true
title:  "Creating concurrent indexes without down time in Django + PostgreSQL"
description: "We need to add indexes to tables to speed up the select query time. Applying indexes on the large databases is quite expensive. so, for this kinds of cases we need to create the indexes concurrently. Let's see how we can do it with django ORM"
keywords: "django, database, postgres"
date: 2021-07-31
categories: [django, postgresql]
author: Anjaneyulu Batta
---

## Why to add indexes?
- We add indexes on database table columns to speed up the select query as indexes allow faster lookup.
- Adding indexes on multiple columns will slow down the write query, so we need to add indexes only on required columns.

## Why do we face downtime when adding indexes?
- In most of the databases, adding an index requires an exclusive lock on the table.
- Because, we don't want to execute any `UPDATE`, `INSERT`, and `DELETE` operations while the index is created.
- Locking a table might create a problem if the system needs to be available when creating the indexes. If the table has more data, then it will take more time to create the indexes.
- As a result, the system will be unavailable till the index creation process completes.

## What can be done to prevent the database unavailability when adding indexes?
- If the database vendors didn't provide the functionality to create the indexes without locking the table then we can't do anything.
- But, some database vendors like PostgreSQL provide the functionality to create the indexes without locking the table.
- We can create indexes on PostgreSQL concurrently using keyword `CONCURRENTLY`.
- Let's see a simple SQL query to create an index concurrently.

```sql
CREATE INDEX CONCURRENTLY "my_index" ON my_table("my_column");
```

## Django migration to apply indexes concurrently on a PostgreSQL database
Let's consider a simple scenario of product table.

```python
from django.db import models

class Product(models.Model):
  name = models.CharField()
  description = models.TextField()
  
  class Meta:
    db_table="product"
```
Let's say we didn't add the indexes on the table and the data is on the table is huge and it's resulting in slow queries. so, we need to create the indexes for the column `name`.

By defult django runs the migrations by acquiring the lock on the table. To avoid it, we need to write a custom migration.

Let's create a migration to create indexes for the table concurrently.

> 002_product_index.py

```python
import django.contrib.postgres.indexes
from django.db import migrations, models
from django.contrib.postgres.operations import AddIndexConcurrently


class Migration(migrations.Migration):

    atomic = False

    dependencies = [
        ("app_name", "001_initial"),
    ]

    operations = [
        AddIndexConcurrently(
            model_name="product",
            index=models.Index(
                fields=["name"], name="name_idx"
            ),
        ),
    ]
```

- When we run the concurrent migrations on django we need to set `atomic = False` on the migration file.
- To create the index concurrently on PostgreSQL django ORM provides class `AddIndexConcurrently`
- By using `AddIndexConcurrently` and `atomic=False` we can create the indexes concurrently.


Let's create a SQL for the above migration to confirm the SQL

```sh
python manage.py sqlmigrate app_name 002_product_index
```

It will generate the SQL like below

```sql
CREATE INDEX CONCURRENTLY "name_idx" ON product("name");
```

Now, we can confirm that the migration will create the indexes concurrently.
Let's apply the migration with the below command

```sh
python manage.py migrate app_name 002_product_index
```

Now, check the database. We should be able to see the index created for the table.

That's it folks. stay tuned for more articles.


References:
1. https://docs.djangoproject.com/en/dev/ref/contrib/postgres/operations/#django.contrib.postgres.operations.AddIndexConcurrently
2. https://docs.djangoproject.com/en/3.2/howto/writing-migrations/#non-atomic-migrations
3. https://www.postgresql.org/docs/9.1/sql-createindex.html