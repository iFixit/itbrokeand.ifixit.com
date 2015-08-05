---
layout: with-comments
title: "Renaming MySQL Tables With Zero Downtime"
author: Marc Zych
author_url: https://github.com/marczych
summary: We found a way to rename MySQL tables with zero downtime by using
         updatable views and atomic rename table commands.
---


Sometimes you end up with tables in your database schema that just aren't named what you want anymore.
At first glance, renaming them without any downtime isn't feasible because updating the code that uses the table at the same instant that the table is renamed isn't easily accomplished.

We recently came across a few database tables in our schema that were begging to be renamed.
Bringing our site down for several minutes to run the migrations and deploy the updated code was unacceptable so I started investigating to come up with a solution.

In this example, we will be renaming `people` to `users`.

# Using a View

The first thing I came up with was using [MySQL Views] to alias the table so it would be accessible by both names (`people` and `users`).
I came to find out that [many MySQL Views are updatable and insertable][updatable views]:

> Some views are updatable and references to them can be used to specify tables to be updated in data change statements. That is, you can use them in statements such as UPDATE, DELETE, or INSERT to update the contents of the underlying table.

This is super cool because it means that _all_ of the queries, reads and writes, can be done against the new name.
Here we create a view with the new name that we want to end up with:

{% highlight mysql %}
CREATE VIEW `users` AS SELECT * FROM `people`;
{% endhighlight %}

# Updating Queries in Code

Now that we have a view that acts just like a proper table, we can deploy the code that updates all queries to reference `users` instead of `people`.
All actions, including writes, proxy through to the underlying table, `people`.

# Replacing the View With the Table

The last piece of the puzzle is to rename the table and drop the old view.
Fortunately, MySQL's [rename table] command allows you to atomically rename multiple tables:

{% highlight mysql %}
RENAME TABLE
   `users` TO `users_old_view`,
   `people` to `users`;

DROP VIEW `users_old_view`;
{% endhighlight %}

And just like that we have renamed the table with zero downtime.

# Caveats

This approach has a few caveats that you should be aware of:

## Broken Views

While foreign keys _are_ updated when renaming the table, queries in views are _not_.
In fact, immediately after the `RENAME TABLE` command in this example, the query backing `users_old_view` is left untouched so it is still referencing `people` which doesn't exist anymore.
This doesn't matter in particular because we end up dropping the view in the next statement, but leaving a bad view does cause `mysqldump` to fail.

## Missing Data in the Information Schema

Also, we discovered one minor difference between the table and the view: the primary key for the view is not available in the `information_schema`.
Our ORM uses this information to build its model of the object so it can generate queries to interact with the underlying table.
Fortunately, we could easily provide the information it needed in code while the view was in use so it wouldn't have to look it up in the information schema table at all.

There may be more differences between tables and views that we didn't come across, so exercise caution and do thorough testing before using them in production.

# Conclusion

All of the SQL commands were nearly instantaneous and worked flawlessly when used on a fairly large table with a modest number of reads and writes.
Consider using this approach if you are running MySQL and have some tables that need to be renamed.

Happy table renaming!

[MySQL Views]: https://dev.mysql.com/doc/refman/5.5/en/views.html
[updatable views]: https://dev.mysql.com/doc/refman/5.5/en/view-updatability.html
[rename table]: https://dev.mysql.com/doc/refman/5.5/en/rename-table.html
