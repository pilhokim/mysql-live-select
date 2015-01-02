# mysql-live-select

NPM Package to provide events when a MySQL select statement result set changes.

Built using the [`zongji` Binlog Tailer](https://github.com/nevill/zongji) and [`node-mysql`](https://github.com/felixge/node-mysql) projects.

## *Under Construction*

This package is yet completed. In order to test it, please use the following command sequence as a guide:

```bash
$ git clone https://github.com/numtel/mysql-live-select.git
$ cd mysql-live-select
$ npm install
$ cd node_modules
# Updated ZongJi module not yet available on NPM, clone repo instead:
$ git clone https://github.com/nevill/zongji.git
$ cd ..
# To use example as-is, copy example data into MySQL
$ mysql < example.sql
# Configure example to match your MySQL settings
# (Be sure to also to enable the binary log, as described in next section)
$ vim example.js
# Run example
$ node example.js
```

Coming soon:

* Automated tests

## Installation

* Enable MySQL binlog in `my.cnf`, restart MySQL server after making the changes.

  ```
  # binlog config
  server-id        = 1
  log_bin          = /usr/local/var/log/mysql/mysql-bin.log
  binlog_do_db     = employees   # optional
  expire_logs_days = 10          # optional
  max_binlog_size  = 100M        # optional

  # Very important if you want to receive write, update and delete row events
  binlog_format    = row
  ```
* Create an account with replication privileges:

  ```sql
  GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'user'@'localhost'
  ```

## LiveMysql Constructor

The `LiveMysql` constructor makes 3 connections to your MySQL database:

* Connection for executing `SELECT` queries
* Replication slave connection
* `information_schema` connection for column information

One argument, an object defining the settings. In addition the [`node-mysql` connection settings](#...), the following settings are available:

Setting | Type | Description
--------|------|------------------------------
`serverId`  | `integer` | [Unique number (1 - 2<sup>32</sup>)](http://dev.mysql.com/doc/refman/5.0/en/replication-options.html#option_mysqld_server-id) to identify this replication slave instance. Must be specified if running more than one instance.<br>**Default:** `1`
`minInterval` | `integer` | Pass a number of milliseconds to use as the minimum between result set updates. Omit to refresh results on every update.

```javascript
// Example:
var table = 'my_table';
var id = 11;

liveMysqlInstance.select(function(esc, escId){
  return (
    'select * from ' + escId(table) +
    'where `id`=' + esc(id)
  );
}, [
  table: table,
  condition: function(row, newRow){ return row.id === id; }
]).on('update', function(data){
  console.log(data);
});
```


### LiveMysql.prototype.select(query, triggers)

Argument | Type | Description
---------|------|----------------------------------
`query`  | `string` or `function` | `SELECT` SQL statement. See note below about passing function.
`triggers` | `[object]` | Array of objects defining which row changes to update result set

Returns `LiveMysqlSelect` object which inherits from [`EventEmitter`](http://nodejs.org/api/events.html), providing `update` and `error` events.

#### Function as `query`

A function may be passed as the `query` argument that accepts two arguments.

* The first argument, `esc` is a function that escapes values in the query.
* The second argument, `escId` is a function that escapes identifiers in the query.

#### Trigger options

Name | Type | Description
-----|------|------------------------------
`table` | `string` | Name of table (required)
`database` | `string` | Name of database (optional)<br>**Default:** `database` setting specified on connection
`condition` | `function` | Evaluate row values (optional)

#### Condition Function

A condition function accepts one or two arguments:

Argument Name | Description
--------------|-----------------------------
`row`         | Table row data
`newRow`      | New row data (only available on `UPDATE` queries)

Return `true` when the row data meets the condition to update the result set.

## License

MIT
