# How to deep-clean your WordPress MySQL database

I have a client who hired me after his website had been crashing for months. This week, I emptied 285 megabytes of transients from his `wp_options` table.

A deleted social media plugin had left thousands of rows of transients in the database. Because the plugin was gone and didn't clean up after itself, the transients were in database purgatory—they couldn't expire, so a plugin like WP-Optimize couldn't see and clear them. So they sat there for probably years, crashing this website over and over.

After I deleted them, the `wp_options` table went from 285 MB to 2. Seriously. 

Don't let this happen to your WordPress website.

Here is an extensive list of MySQL CLI snippets to inspect, diagnose, and clean up your WordPress database. You can also use this SQL code inside phpMyAdmin.

Key things to watch for:

Spring cleaning: you can fix these with a plugin such as WP-Optimize or clean them by hand. 
* Revision size in the `wp_posts` table 
* Spam and trashed comments in `wp_comments`

Advanced cleaning: 

* The `wp_options` table loads on every WordPress page and is crucial to optimize—aim for under 2 MB and 500 rows.
* Transient size in the `wp_options` table
* Orphaned autoload options in `wp_options` 
* Funny business or orphaned data in `wp_postmeta`—like a plugin taking up 100MB duplicating your post content. Yes, I've seen that, too.
* Large tables with unnecessary plugin logs, such as EWWW or WordFence

---

Check the size of the database, using your own 'database_name'. 

```
SELECT (SUM(DATA_LENGTH + INDEX_LENGTH))/1048567 FROM INFORMATION_SCHEMA.TABLES 
  WHERE TABLE_SCHEMA = 'database_name';
```

Check an individual table size:

```
SELECT (SUM(DATA_LENGTH + INDEX_LENGTH))/1048567 FROM INFORMATION_SCHEMA.TABLES 
     WHERE TABLE_NAME = 'wp_posts';
```

## `wp_users` Table

_Not clean-up, just a step I take in evaluating a database._

Count the rows—always do this first before dumping 10,000 rows into your CLI.	
```
SELECT COUNT(*) FROM wp_users;
```

Check the user list
```
SELECT * FROM wp_users;
```

Security: check for the “admin” user. Use something else for your site admin as this is a default and thus a hacker target.
```
SELECT * FROM wp_users
	WHERE user_login = 'admin';
```

## `wp_usermeta` Table

Check for orphaned metadata by listing usermeta user IDs not found in users:
```
SELECT DISTINCT user_id 
  FROM wp_usermeta
  LEFT JOIN 
  wp_users
  ON wp_users.id = wp_usermeta.user_id
  WHERE wp_users.id IS NULL;
```

Check the list of meta keys for plugin additions - WordPress uses 20+ by default.
```
SELECT DISTINCT meta_key FROM wp_usermeta;
```

Security: check who the admins are
```
SELECT *
  FROM wp_usermeta;
  WHERE meta_value LIKE ('%administrator%');
```

Or search for all user roles:
```
SELECT * 
  FROM wp_usermeta
  WHERE `meta_key` = 'wp_capabilities' 
  OR `meta_key` = 'wp_user_level'
  ORDER BY user_id;
```

## `wp-posts` Table

Check post type, data size, and row counts. This is a good way to check for table clutter from post revisions.
```
SELECT 
  post_type, 
  COUNT(*) AS `Rows`,
  ROUND(
    SUM(
      LENGTH(
        CONCAT(
          ID, post_author, post_date, post_date_gmt, post_content, post_title, post_excerpt, post_status, comment_status, ping_status, post_password, post_name, to_ping, pinged, post_modified, post_modified_gmt, post_parent, guid, menu_order, post_type
        )
      )
    )/1048567, 2
  ) AS `Data_in_MB`
  FROM wp_posts
  GROUP BY post_type
  ORDER BY `Data_in_MB` DESC;
```

Delete all revisions
```
DELETE FROM wp_posts WHERE post_type = 'revision';
```

## `wp_postmeta` Table

Check `wp_postmeta` keys, rows, and data size. This will give you a grouped, ranked list of the meta keys, which you can ID to the plugins that generated them with their naming prefix. 

```
SELECT
  meta_key, 
  COUNT(*) AS `Rows`,  
  ROUND(
    SUM(
      LENGTH(
        CONCAT(meta_id, post_id, meta_key, meta_value)
      )
    )/1048567, 2
  ) AS `Size_in_MB`
  FROM wp_postmeta
  GROUP BY meta_key
  ORDER BY `Size_in_MB` DESC;
```

See also this [in-depth article](https://www.rawkblog.com/2018/01/how-to-clean-up-the-wordpress-wp_postmeta-database-table/) on digging through meta keys. 

## `wp_options` Table:

Get table size
```
SELECT  
  ROUND(
    SUM(
      LENGTH(
        CONCAT(option_id, option_name, option_value, autoload)
      )
    )/1048567, 2
  ) AS `Options_Table_Size_In_MB`
  FROM wp_options;
```

Get transient size and row count

```
SELECT 
  COUNT(*) AS `Transients_Rows`, 
  ROUND(
    SUM(
      LENGTH(
        CONCAT(option_id, option_name, option_value, autoload)
      )
    )/1048567, 2
  ) AS `Transients_data_in_MB`
FROM wp_options
WHERE option_name LIKE ('%\_transient\_%');
```

Check how many transients autoload

```
SELECT COUNT(*) AS `Transient_Rows_That_Autoload`
FROM wp_options
WHERE option_name LIKE ('%\_transient\_%') AND autoload = 'yes';
```

Delete all transients:

_You might want to delete by plugin namespace or a date window instead of deleting all of them, especially if you have current e-commerce or logged-in user sessions running via this method._

_If you're using an object cache plugin such as Memcached or Redis, you might not have transients in your database at all. Congrats!_

```
DELETE FROM wp_options WHERE option_name LIKE ('%\_transient\_%');
```

Get the size and row count of options which autoload

```
SELECT
  COUNT(*) AS `Autoload_Row_Count`,
  ROUND(
    SUM(
      LENGTH(
        CONCAT(option_id, option_name, option_value, autoload)
      )
    )/1048567, 2
  ) AS `Autoload_Data_in_MB`
FROM wp_options
WHERE autoload = 'yes';
```

See all autoload options:

_This is a good method to check for orphaned options still loading on your site every time even after you deleted the plugin that created them. I guarantee you have at least a dozen._

```
SELECT option_name FROM wp_options WHERE autoload = 'yes';
```
