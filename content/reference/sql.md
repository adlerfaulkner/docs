---
title: SQL
---

# Overview

Dolt data repositories are unique in that they arrive capable of running MySQL compatible SQL statements against them. Your Dolt binary can execute SQL against any Dolt repo you have happen to have locally. Existing RDBMS solutions have optimized their on-disk layout to support the needs of physical plan execution. Dolt makes a different set of tradeoffs, instead using a Merkel Tree to support robust version control semantics, and thus portability of the data. The combination of portability, robust version control semantics, and a SQL interface make Dolt a uniquely compelling data format.

## Executing SQL

There are two ways of executing SQL against your data repository. The first is via the `dolt sql` command, which runs SQL queries from your shell, and the second is the sql-server command, which starts a MySQL compatible server you can connect to with any standard database client.

### dolt sql

Using `dolt sql` you can issue SQL statements against a local data repository directly. Any writes made to the repository will be reflected in the working set, which can be added via `dolt add` and committed via `dolt commit` as per usual.

There are 2 basic modes which the `dolt sql` command operate in. The first is command line query mode where a semicolon delimited list of queries is provided on the command line using the -q parameter. The second is using the Dolt SQL interactive shell which can be started by running the `dolt sql` command without any arguments in a directory containing a Dolt data repository.

View the `dolt sql` command documentation [here](../cli/#dolt-sql)

### dolt sql-server

The `dolt sql-server` command will run a MySQL compatible server which clients can execute queries against. This provides a programmatic interface to get data into or out of a Dolt data repository. It can also be used to connect with third party tooling which supports MySQL.

View the `dolt sql-server` command documentation [here](../cli/#dolt-sql-server) to learn how to configure the server.


## Dolt CLI in SQL

You can operate several of dolt cli commands in the sql layer directly. This is especially useful if you are using sql in the application layer and want to the query a Dolt repository.


## System Tables

Many of Dolt's unique features are accessible via system tables. These tables allow you to query the same information available from various Dolt commands, such as branch information, the commit log, and much more. You can write queries that examine the history of a table, or that select the diff between two commits. See the individual sections below for more details.

- [dolt_log table](#dolt-system-tables_dolt_log)
- [dolt_branches table](#dolt-system-tables_dolt_branches)
- [dolt_docs table](#dolt-system-tables_dolt_docs)
- [dolt_diff tables](#dolt-system-tables_dolt_diff_tablename)
- [dolt_history tables](#dolt-system-tables_dolt_history_tablename)
- [dolt_schemas table](#dolt-system-tables_dolt_schemas)

## Concurrency

When any client initiates a SQL session against a Dolt data repository, that session will be pointing to a specific commit even if other clients make changes. Therefore, modifications made by other clients will not be visible. There are two commit modes which determine how you are able to write to the database, commit those writes, and get modifications made by other clients.

### Autocommit mode

```
NOTE: This may be confusing to some readers. Commit in this context is Commit in the database context not Commit in the
version control context.
```

In autocommit mode, when a client connects they will be pinned to the working data for the Dolt data repository. Any write queries will modify the in memory state, and automatically update the working set. This is the most intuitive model in that it works identically to the `dolt sql` command. Any time a write query completes against the server, you could open up a separate terminal window and see the modifications with `dolt diff` or by running a SELECT query using `dolt sql`. That same client will be able to see his modifications in read queries, however if there was a second client that connected at the same time, they will not see eachother's writes, and if both tried to make writes to the database the last write would win, and the first would be overwritten. This is why maximum connections should be set to 1 when working in this mode (See the `dolt sql-server` docs [here](../cli/#dolt-sql-server) to see how to configure
the server).

### Manual commit mode

In manual-commit mode users will manually set what commit they are pinned to and user writes are not written to the database until the user creates a commit manually. Manually created commits can be used in insert and update statements on the [dolt_branches](../sql/#dolt-system-tables) table. In manual commit mode it is possible for multiple users to interact with the database simultaneously, however until merge support with conflict resolution is supported in dolt there are limitations.

See the full [manual commit mode documentation](../sql/#concurrency)

# Querying non-HEAD revisions of a database

Dolt SQL supports a variant of [SQL
2011](https://en.wikipedia.org/wiki/SQL:2011) syntax to query non-HEAD
revisions of a database via the `AS OF` clause:

```sql
SELECT * FROM myTable AS OF 'kfvpgcf8pkd6blnkvv8e0kle8j6lug7a';
SELECT * FROM myTable AS OF 'myBranch';
SELECT * FROM myTable AS OF 'HEAD^2';
SELECT * FROM myTable AS OF TIMESTAMP('2020-01-01');
SELECT * FROM myTable AS OF 'myBranch' JOIN myTable AS OF 'yourBranch' AS foo;
```

The `AS OF` expression must name a valid Dolt reference, such as a
commit hash, branch name, or other reference. Timestamp / date values
are also supported. Each table in a query can use a different `AS OF`
clause.

In addition to this `AS OF` syntax for `SELECT` statements, Dolt also
supports an extension to the standard MySQL syntax to examine the
database schema for a previous revision:

```sql
SHOW TABLES AS OF 'kfvpgcf8pkd6blnkvv8e0kle8j6lug7a';
```

# DOLT CLI in SQL


## `DOLT_COMMIT()`


### Description

Commits staged tables to HEAD. Works exactly like `dolt commit` with each value directly following the flag. Note that you must always support the message flag with the intended message right after.

By default, when running in server mode with dolt sql-server, dolt does not automatically update the working set of your repository with data updates unless @@autocommit is set to 1. You can also issue manual COMMIT statements to flush the working set to disk. See the section on [concurrency](https://www.dolthub.com/docs/reference/sql/#concurrency).


```sql
SELECT DOLT_COMMIT('-a', '-m', 'This is a commit');
SELECT DOLT_COMMIT('-m', 'This is a commit');
SELECT DOLT_COMMIT('-m', 'This is a commit', '--author', 'John Doe <johndoe@example.com>');
```

### Options

`-m`, `--message`:
Use the given `<msg>` as the commit message. **Required**

`-a`:
Stages all tables with changes before committing

`--allow-empty`:
Allow recording a commit that has the exact same data as its sole parent. This is usually a mistake, so it is disabled by default. This option bypasses that safety.

`--date`:
Specify the date used in the commit. If not specified the current system time is used.

`--author`:
Specify an explicit author using the standard "A U Thor <author@example.com>" format.

### Example

```sql
-- Set the current database for the session
USE mydb;

-- Make modifications
UPDATE table
SET column = "new value"
WHERE pk = "key";

-- Stage all changes and commit.
SELECT DOLT_COMMIT('-a', '-m', 'This is a commit', '--author', 'John Doe <johndoe@example.com>');
```

# Dolt System Tables

## `dolt_branches`

### Description

Queryable system table which shows the Dolt data repository branches.

### Schema

```
+------------------------+----------+
| Field                  | Type     |
+------------------------+----------+
| name                   | TEXT     |
| hash                   | TEXT     |
| latest_committer       | TEXT     |
| latest_committer_email | TEXT     |
| latest_commit_date     | DATETIME |
| latest_commit_message  | TEXT     |
+------------------------+----------+
```

### Example Queries

Get all the branches.

```sql
SELECT *
FROM dolt_branches
```

```
+--------+----------------------------------+------------------+------------------------+-----------------------------------+-------------------------------+
| name   | hash                             | latest_committer | latest_committer_email | latest_commit_date                | latest_commit_message         |
+--------+----------------------------------+------------------+------------------------+-----------------------------------+-------------------------------+
| 2011   | t2sbbg3h6uo93002frfj3hguf22f1uvh | bheni            | brian@dolthub.com     | 2020-01-22 20:47:31.213 +0000 UTC | import 2011 column mappings   |
| 2012   | 7gonpqhihgnv8tktgafsg2oovnf3hv7j | bheni            | brian@dolthub.com     | 2020-01-22 23:01:39.08 +0000 UTC  | import 2012 allnoagi data     |
| 2013   | m9seqiabaefo3b6ieg90rr4a14gf6226 | bheni            | brian@dolthub.com     | 2020-01-22 23:50:10.639 +0000 UTC | import 2013 zipcodeagi data   |
| 2014   | v932nm88f5g3pjmtnkq917r2q66jm0df | bheni            | brian@dolthub.com     | 2020-01-23 00:00:43.673 +0000 UTC | update 2014 column mappings   |
| 2015   | c7h0jc23hel6qbh8ro5ertiv15to9g9o | bheni            | brian@dolthub.com     | 2020-01-23 00:04:35.459 +0000 UTC | import 2015 allnoagi data     |
| 2016   | 0jntctp6u236le9qjlt9kf1q1if7mp1l | bheni            | brian@dolthub.com     | 2020-01-28 20:38:32.834 +0000 UTC | fix allnoagi zipcode for 2016 |
| 2017   | j883mmogbd7rg3cfltukugk0n65ud0fh | bheni            | brian@dolthub.com     | 2020-01-28 16:43:45.687 +0000 UTC | import 2017 allnoagi data     |
| master | j883mmogbd7rg3cfltukugk0n65ud0fh | bheni            | brian@dolthub.com     | 2020-01-28 16:43:45.687 +0000 UTC | import 2017 allnoagi data     |
+--------+----------------------------------+------------------+------------------------+-----------------------------------+-------------------------------+
```

Get the current branch for a database named "mydb".

```sql
SELECT *
FROM dolt_branches
WHERE hash = @@mydb_head
```

```
+--------+----------------------------------+------------------+------------------------+-----------------------------------+-------------------------------+
| name   | hash                             | latest_committer | latest_committer_email | latest_commit_date                | latest_commit_message         |
+--------+----------------------------------+------------------+------------------------+-----------------------------------+-------------------------------+
| 2016   | 0jntctp6u236le9qjlt9kf1q1if7mp1l | bheni            | brian@dolthub.com     | 2020-01-28 20:38:32.834 +0000 UTC | fix allnoagi zipcode for 2016 |
+--------+----------------------------------+------------------+------------------------+-----------------------------------+-------------------------------+
```

Create a new commit, and then create a branch from that commit

```sql
SET @@mydb_head = COMMIT("my commit message")

INSERT INTO dolt_branches (name, hash)
VALUES ("my branch name", @@mydb_head);
```

## `dolt_diff_$TABLENAME`

### Description

For every user table named `$TABLENAME`, there is a queryable system table named `dolt_diff_$TABLENAME` which can be queried
to see how rows have changed over time. Each row in the result set represent a row that has changed between two commits.

### Schema

Every Dolt diff table will have the columns

```
+-------------+------+
| field       | type |
+-------------+------+
| from_commit | TEXT |
| to_commit   | TEXT |
| diff_type   | TEXT |
+-------------+------+
```

The remaining columns will be dependent on the schema of the user table. For every column X in your table at `from_commit`, there will be a column in the result set named `from_$X` with the same type as `X`, and for every column `Y` in your table at `to_commit` there will be a column in the result set named `to_$Y` with the same type as `Y`.

### Example Schema

For a hypothetical table named states with a schema that changes between `from_commit` and `to_commit` as shown below

```
# schema at from_commit    # schema at to_commit
+------------+--------+    +------------+--------+
| field      | type   |    | field      | type   |
+------------+--------+    +------------+--------+
| state      | TEXT   |    | state      | TEXT   |
| population | BIGINT |    | population | BIGINT |
+------------+--------+    | area       | BIGINT |
                           +-------------+-------+
```

The schema for `dolt_diff_states` would be

```
+-----------------+--------+
| field           | type   |
+-----------------+--------+
| from_state      | TEXT   |
| to_state        | TEXT   |
| from_population | BIGINT |
| to_population   | BIGINT |
| to_state        | TEXT   |
| from_commit     | TEXT   |
| to_commit       | TEXT   |
| diff_type       | TEXT   |
+-----------------+--------+
```

### Query Details

Doing a `SELECT *` query for a diff table will show you every change that has occurred to each row for every commit in this
branches history. Using `to_commit` and `from_commit` you can limit the data to specific commits. There is one special `to_commit`
value `WORKING` which can be used to see what changes are in the working set that have yet to be committed to HEAD. It
is often useful to use the `HASHOF()` function to get the commit hash of a branch, or an anscestor commit. To get the
differences between the last commit and it's parent you could use `to_commit=HASHOF("HEAD") and from_commit=HASHOF("HEAD^")`

For each row the field `diff_type` will be one of the values `added`, `modified`, or `removed`. You can filter which rows appear in the result set to one or more of those `diff_type` values in order to limit which types of changes will be returned.

### Example Query

Taking the [`dolthub/wikipedia-ngrams`](https://www.dolthub.com/repositories/dolthub/wikipedia-ngrams) data repository from [DoltHub](https://www.dolthub.com/) as our example, the following query will retrieve the bigrams whose total counts have changed the most between 2 versions.

```sql
SELECT from_bigram, to_bigram, from_total_count, to_total_count, ABS(to_total_count-from_total_count) AS delta
FROM dolt_diff_bigram_counts
WHERE from_commit = HASHOF("HEAD~3") AND diff_type = "modified"
ORDER BY delta DESC
LIMIT 10;
```

```
+-------------+-------------+------------------+----------------+-------+
| from_bigram | to_bigram   | from_total_count | to_total_count | delta |
+-------------+-------------+------------------+----------------+-------+
| of the      | of the      | 21566470         | 21616678       | 50208 |
| _START_ The | _START_ The | 19008468         | 19052410       | 43942 |
| in the      | in the      | 14345719         | 14379619       | 33900 |
| _START_ In  | _START_ In  | 8212684          | 8234586        | 21902 |
| to the      | to the      | 7275659          | 7291823        | 16164 |
| _START_ He  | _START_ He  | 5722362          | 5737483        | 15121 |
| at the      | at the      | 4273616          | 4287398        | 13782 |
| for the     | for the     | 4427780          | 4438872        | 11092 |
| and the     | and the     | 4871852          | 4882874        | 11022 |
| is a        | is a        | 4632620          | 4643068        | 10448 |
+-------------+-------------+------------------+----------------+-------+
```

## `dolt_docs`

### Description

System table that stores the contents of Dolt docs (`LICENSE.md`, `README.md`).

### Schema

```
+----------+------+
| field    | type |
+----------+------+
| doc_name | text |
| doc_text | text |
+----------+------+
```

### Usage

Dolt users do not have to be familiar with this system table in order to make a `LICENSE.md` or `README.md`. Simply run `dolt init` or `touch README.md` and `touch LICENSE.md` from a Dolt repository to get started. Then, `dolt add` and `dolt commit` the docs like you would a table.

## `dolt_history_$TABLENAME`

### Description

For every user table named $TABLENAME, there is a queryable system table named dolt\_history\_$TABLENAME which can be queried to find a rows value at every commit in the current branches commit graph.

### Schema

Every Dolt history table will have the columns

```
+-------------+----------+
| field       | type     |
+-------------+----------+
| commit_hash | TEXT     |
| committer   | TEXT     |
| commit_date | DATETIME |
+-------------+----------+
```

The rest of the columns will be the superset of all columns that have existed throughout the history of the table. As the query.

### Example Schema

For a hypothetical data repository with the following commit graph:

```
   A
  / \
 B   C
      \
       D
```

Which has a table named states with the following schemas at each commit:

```
# schema at A              # schema at B              # schema at C              # schema at D
+------------+--------+    +------------+--------+    +------------+--------+    +------------+--------+
| field      | type   |    | field      | type   |    | field      | type   |    | field      | type   |
+------------+--------+    +------------+--------+    +------------+--------+    +------------+--------+
| state      | TEXT   |    | state      | TEXT   |    | state      | TEXT   |    | state      | TEXT   |
| population | BIGINT |    | population | BIGINT |    | population | BIGINT |    | population | BIGINT |
+------------+--------+    | capital    | TEXT   |    | area       | BIGINT |    | area       | BIGINT |
                           +------------+--------+    +------------+--------+    | counties   | BIGINT |
                                                                                 +------------+--------+
```

The schema for dolt_history_states would be:

```
+-------------+----------+
| field       | type     |
+-------------+----------+
| state       | TEXT     |
| population  | BIGINT   |
| capital     | TEXT     |
| area        | BIGINT   |
| counties    | BIGINT   |
| commit_hash | TEXT     |
| committer   | TEXT     |
| commit_date | DATETIME |
+-------------+----------+
```

### Example Query

Taking the above table as an example. If the data inside dates for each commit was:

- At commit "A" the state data from 1790
- At commit "B" the state data from 1800
- At commit "C" the state data from 1800
- At commit "D" the state data from 1810

```sql
SELECT *
FROM dolt_history_states
WHERE state = "Virginia";
```

```
+----------+------------+----------+--------+----------+-------------+-----------+---------------------------------+
| state    | population | capital  | area   | counties | commit_hash | committer | commit_date                     |
+----------+------------+----------+--------+----------+-------------+-----------+---------------------------------+
| Virginia | 691937     | <NULL>   | <NULL> | <NULL>   | HASH_AT(A)  | billybob  | 1790-01-09 00:00:00.0 +0000 UTC |
| Virginia | 807557     | Richmond | <NULL> | <NULL>   | HASH_AT(B)  | billybob  | 1800-01-01 00:00:00.0 +0000 UTC |
| Virginia | 807557     | <NULL>   | 42774  | <NULL>   | HASH_AT(C)  | billybob  | 1800-01-01 00:00:00.0 +0000 UTC |
| Virginia | 877683     | <NULL>   | 42774  | 99       | HASH_AT(D)  | billybob  | 1810-01-01 00:00:00.0 +0000 UTC |
+----------+------------+----------+--------+----------+-------------+-----------+---------------------------------+

# Note in the real result set there would be actual commit hashes for each row.  Here I have used notation that is
# easier to understand how it relates to our commit graph and the data associated with each commit above
```

## `dolt_log`

### Description

Queryable system table which shows the commit log

### Schema

```
+-------------+----------+
| field       | type     |
+-------------+--------- +
| commit_hash | text     |
| committer   | text     |
| email       | text     |
| date        | datetime |
| message     | text     |
+-------------+--------- +
```

### Example Query

```sql
SELECT *
FROM dolt_log
WHERE committer = "bheni" and date > "2019-04-01"
ORDER BY "date";
```

```
+----------------------------------+-----------+--------------------+-----------------------------------+---------------+
| commit_hash                      | committer | email              | date                              | message       |
+----------------------------------+-----------+--------------------+-----------------------------------+---------------+
| qi331vjgoavqpi5am334cji1gmhlkdv5 | bheni     | brian@dolthub.com | 2019-06-07 00:22:24.856 +0000 UTC | update rating |
| 137qgvrsve1u458briekqar5f7iiqq2j | bheni     | brian@dolthub.com | 2019-04-04 22:43:00.197 +0000 UTC | change rating |
| rqpd7ga1nic3jmc54h44qa05i8124vsp | bheni     | brian@dolthub.com | 2019-04-04 21:07:36.536 +0000 UTC | fixes         |
+----------------------------------+-----------+--------------------+-----------------------------------+---------------+
```

## `dolt_schemas`

### Description

SQL Schema fragments for a dolt database value that are versioned alongside the database itself. Certain DDL statements will modify this table and the value of this table in a SQL session will affect what database entities exist in the session.

### Schema

```
+-------------+----------+
| field       | type     |
+-------------+--------- +
| type        | text     |
| name        | text     |
| fragment    | text     |
+-------------+--------- +
```

Currently on view definitions are stored in `dolt_schemas`. `type` is currently always the string `view`. `name` is the name of the view as supplied in the `CREATE VIEW ...` statement. `fragment` is the `select` fragment that the view is defined as.

The values in this table are partly implementation details associated with the implementation of the underlying database objects.

### Example Query

```sql
CREATE VIEW four AS SELECT 2+2 FROM dual;
SELECT * FROM dolt_schemas;
```

```
+------+------+----------------------+
| type | name | fragment             |
+------+------+----------------------+
| view | four | select 2+2 from dual |
+------+------+----------------------+
```

# Supported SQL Features

Dolt's goal is to be a drop-in replacement for MySQL, with every query and statement that works in MySQL behaving identically in Dolt. For most syntax and technical questions, you should feel free to refer to the [MySQL user manual](https://dev.mysql.com/doc/refman/8.0/en/select.html). Any deviation from the MySQL manual should be documented on this page, or else indicates a bug. Please [file issues](https://github.com/dolthub/dolt/issues) with any incompatibilities you discover.

## Data types

<table class="sql-table">
  <thead>
      <tr>
        <th>Data type</th>
        <th>Supported</th>
        <th>Notes</th>
      </tr>
  </thead>
    <tbody>
      <tr>
        <td><code>BOOLEAN</code></td>
        <td class="green">✓</td>
        <td>Alias for <code>TINYINT</code></td>
      </tr>
      <tr>
        <td><code>INTEGER</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>TINYINT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>SMALLINT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>MEDIUMINT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>INT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>BIGINT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>DECIMAL</code></td>
        <td class="green">✓</td>
        <td>Max (precision + scale) is 65</td>
      </tr>
      <tr>
        <td><code>FLOAT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>DOUBLE</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>BIT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>DATE</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>TIME</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>DATETIME</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>TIMESTAMP</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>YEAR</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>CHAR</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>VARCHAR</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>BINARY</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>VARBINARY</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>BLOB</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>TINYTEXT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>TEXT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>MEDIUMTEXT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>LONGTEXT</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>ENUM</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>SET</code></td>
        <td class="green">✓</td>
        <td></td>
      </tr>
      <tr>
        <td><code>GEOMETRY</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>POINT</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>LINESTRING</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>POLYGON</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>MULTIPOINT</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>MULTILINESTRING</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>MULTIPOLYGON</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>GEOMETRYCOLLECTION</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
      <tr>
        <td><code>JSON</code></td>
        <td class="red">X</td>
        <td></td>
      </tr>
    </tbody>
</table>

## Constraints

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Not Null</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Unique</td>
      <td class="green">✓</td>
      <td>Unique constraints are supported via creation of indexes with <code>UNIQUE</code> keys.</td>
    </tr>
    <tr>
      <td>Primary Key</td>
      <td class="green">✓</td>
      <td>Dolt tables must have a primary key.</td>
    </tr>
    <tr>
      <td>Check</td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td>Foreign Key</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Default Value</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Transactions

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>BEGIN</code></td>
      <td class="orange">O</td>
      <td><code>BEGIN</code> parses correctly, but is a no-op: it doesn&#39;t create a checkpoint that can be returned to with <code>ROLLBACK</code>.</td>
    </tr>
    <tr>
      <td><code>COMMIT</code></td>
      <td class="green">✓</td>
      <td><code>COMMIT</code> will write any pending changes to the working set when <code>@@autocommit = false</code></td>
    </tr>
    <tr>
      <td><code>COMMIT(MESSAGE)</code></td>
      <td class="green">✓</td>
      <td>The <code>COMMIT()</code> function creates a commit of the current database state and returns the hash of this new commit. See <a href="#concurrency">concurrency</a> for details.</td>
    </tr>
    <tr>
      <td><code>LOCK TABLES</code></td>
      <td class="red">X</td>
      <td><code>LOCK TABLES</code> parses correctly but does not prevent access to those tables from other sessions.</td>
    </tr>
    <tr>
      <td><code>ROLLBACK</code></td>
      <td class="red">X</td>
      <td><code>ROLLBACK</code> parses correctly but is a no-op.</td>
    </tr>
    <tr>
      <td><code>SAVEPOINT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SET @@autocommit = 1</code></td>
      <td class="green">✓</td>
      <td>When <code>@@autocommit = true</code>, changes to data will update the working set after every statement. When <code>@@autocommit = false</code>, the working set will only be updated after <code>COMMIT</code> statements.</td>
    </tr>
  </tbody>
</table>

## Indexes

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Indexes</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Multi-column indexes</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Full-text indexes</td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td>Spatial indexes</td>
      <td class="red">X</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Schema

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>ALTER TABLE</code> statements</td>
      <td class="orange">O</td>
      <td>Some limitations. See the <a href="#supported-statements">supported statements doc</a>.</td>
    </tr>
    <tr>
      <td>Database renames</td>
      <td class="red">X</td>
      <td>Database names are read-only, and configured by the server at startup.</td>
    </tr>
    <tr>
      <td>Adding tables</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Dropping tables</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Table renames</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Adding views</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Dropping views</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>View renames</td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td>Column renames</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Adding columns</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Removing columns</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Reordering columns</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Adding constraints</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Removing constaints</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Creating indexes</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Index renames</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Removing indexes</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>AUTO INCREMENT</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Statements

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Common statements</td>
      <td class="green">✓</td>
      <td>See the <a href="#supported-statements">supported statements doc</a></td>
    </tr>
  </tbody>
</table>

## Clauses

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>WHERE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>HAVING</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LIMIT</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>OFFSET</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GROUP BY</code></td>
      <td class="green">✓</td>
      <td>Group-by columns can be referred to by their ordinal (e.g. <code>1</code>, <code>2</code>), a MySQL dialect extension.</td>
    </tr>
    <tr>
      <td><code>ORDER BY</code></td>
      <td class="green">✓</td>
      <td>Order-by columns can be referred to by their ordinal (e.g. <code>1</code>, <code>2</code>), a MySQL dialect extension.</td>
    </tr>
    <tr>
      <td>Aggregate functions</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DISTINCT</code></td>
      <td class="orange">O</td>
      <td>Only supported for certain expressions.</td>
    </tr>
    <tr>
      <td><code>ALL</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Table expressions

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Tables and views</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Table and view aliases</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Joins</td>
      <td class="orange">O</td>
      <td><code>LEFT INNER</code>, <code>RIGHT INNER</code>, <code>INNER</code>, <code>NATURAL</code>, and <code>CROSS JOINS</code> are supported. <code>OUTER</code> joins are not supported.</td>
    </tr>
    <tr>
      <td>Subqueries</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UNION</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Scalar expressions

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Common operators</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IF</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CASE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NULLIF</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COALESCE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IFNULL</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>AND</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>OR</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LIKE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IN</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INTERVAL</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Scalar subqueries</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Column ordinal references</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Functions and operators

**Currently supporting 124 of 436 MySQL functions.**

Most functions are simple to implement. If you need one that isn't
implemented, [please file an
issue](https://github.com/dolthub/dolt/issues). We can fulfill
most requests for new functions within 24 hours.

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>%</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&amp;</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>|</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>*</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>+</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>-&gt;&gt;</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>-&gt;</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>-</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>/</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>:=</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&lt;&lt;</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&lt;=&gt;</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&lt;=</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&lt;&gt;</code>, <code>!=</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&lt;</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>=</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&gt;=</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&gt;&gt;</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>&gt;</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>^</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ABS()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ACOS()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ADDDATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ADDTIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>AES_DECRYPT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>AES_ENCRYPT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>AND</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ANY_VALUE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ARRAY_LENGTH()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ASCII()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ASIN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ASYMMETRIC_DECRYPT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ASYMMETRIC_DERIVE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ASYMMETRIC_ENCRYPT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ASYMMETRIC_SIGN()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ASYMMETRIC_VERIFY()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ATAN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ATAN2()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>AVG()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BENCHMARK()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BETWEEN ... AND ...</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BIN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BIN_TO_UUID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BIT_AND()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BIT_COUNT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BIT_LENGTH()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>BIT_OR()</code></td>
      <td class="red">X</td>
      <td><code>|</code> is supported</td>
    </tr>
    <tr>
      <td><code>BIT_XOR()</code></td>
      <td class="red">X</td>
      <td><code>^</code> is supported</td>
    </tr>
    <tr>
      <td><code>CAN_ACCESS_COLUMN()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CAN_ACCESS_DATABASE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CAN_ACCESS_TABLE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CAN_ACCESS_VIEW()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CASE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CAST()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CEIL()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CEILING()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CHAR()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CHARACTER_LENGTH()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CHARSET()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CHAR_LENGTH()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COALESCE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COERCIBILITY()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COLLATION()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COMMIT()</code></td>
      <td class="green">✓</td>
      <td>Creates a new Dolt commit and returns the hash of it. See <a href="#concurrency">concurrency</a></td>
    </tr>
    <tr>
      <td><code>COMPRESS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CONCAT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CONCAT_WS()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CONNECTION_ID()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CONV()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CONVERT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CONVERT_TZ()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COS()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COUNT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>COUNT(DISTINCT)</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CRC32()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE_ASYMMETRIC_PRIV_KEY()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE_ASYMMETRIC_PUB_KEY()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE_DH_PARAMETERS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE_DIGEST()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CUME_DIST()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CURDATE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CURRENT_DATE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CURRENT_ROLE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CURRENT_TIME()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CURRENT_TIMESTAMP()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CURRENT_USER()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CURTIME()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATABASE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATEDIFF()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATETIME()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATE_ADD()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATE_FORMAT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DATE_SUB()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DAY()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DAYNAME()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DAYOFMONTH()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DAYOFWEEK()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DAYOFYEAR()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DEFAULT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DEGREES()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DENSE_RANK()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DIV</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ELT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>EXP()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>EXPLODE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>EXPORT_SET()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>EXTRACT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ExtractValue()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FIELD()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FIND_IN_SET()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FIRST()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FIRST_VALUE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FLOOR()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FORMAT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FORMAT_BYTES()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FORMAT_PICO_TIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FOUND_ROWS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FROM_BASE64()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FROM_DAYS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>FROM_UNIXTIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GET_DD_COLUMN_PRIVILEGES()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GET_DD_CREATE_OPTIONS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GET_DD_INDEX_SUB_PART_LENGTH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GET_FORMAT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GET_LOCK()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GREATEST()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GROUPING()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GROUP_CONCAT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GTID_SUBSET()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GTID_SUBTRACT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GeomCollection()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GeometryCollection()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>HASHOF()</code></td>
      <td class="green">✓</td>
      <td>Returns the hash of a reference, e.g. <code>HASHOF(&quot;master&quot;)</code>. See <a href="#concurrency">concurrency</a></td>
    </tr>
    <tr>
      <td><code>HEX()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>HOUR()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ICU_VERSION()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IF()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IFNULL()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INET6_ATON()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INET6_NTOA()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INET_ATON()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INET_NTOA()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INSERT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INSTR()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>INTERVAL()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS NOT NULL</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS NOT</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS NULL</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ISNULL()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_BINARY()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_FREE_LOCK()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_IPV4()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_IPV4_COMPAT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_IPV4_MAPPED()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_IPV6()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_USED_LOCK()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS_UUID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IS</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_ARRAY()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_ARRAYAGG()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_ARRAY_APPEND()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_ARRAY_INSERT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_CONTAINS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_CONTAINS_PATH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_DEPTH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_EXTRACT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_INSERT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_KEYS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_LENGTH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_MERGE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_MERGE_PATCH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_MERGE_PRESERVE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_OBJECT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_OBJECTAGG()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_OVERLAPS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_PRETTY()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_QUOTE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_REMOVE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_REPLACE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_SCHEMA_VALID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_SCHEMA_VALIDATION_REPORT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_SEARCH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_SET()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_STORAGE_FREE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_STORAGE_SIZE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_TABLE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_TYPE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_UNQUOTE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_VALID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>JSON_VALUE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LAG()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LAST()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LAST_DAY</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LAST_INSERT_ID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LAST_VALUE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LCASE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LEAD()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LEAST()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LEFT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LENGTH()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LIKE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOAD_FILE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOCALTIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOCALTIMESTAMP()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOCATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOG()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOG10()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOG2()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LOWER()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LPAD()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LTRIM()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>LineString()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MAKEDATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MAKETIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MAKE_SET()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MASTER_POS_WAIT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MATCH</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MAX()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBRContains()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBRCoveredBy()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBRCovers()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBRDisjoint()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBREquals()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBRIntersects()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBROverlaps()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBRTouches()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MBRWithin()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MD5()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MEMBER OF()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MICROSECOND()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MID()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MIN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MINUTE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MOD()</code></td>
      <td class="red">X</td>
      <td><code>%</code> is supported</td>
    </tr>
    <tr>
      <td><code>MONTH()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MONTHNAME()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MultiLineString()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MultiPoint()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MultiPolygon()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NAME_CONST()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOT</code>, <code>!</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOT BETWEEN ... AND ...</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOT IN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOT LIKE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOT MATCH</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOT REGEXP</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOT RLIKE</code></td>
      <td class="red">X</td>
      <td><code>NOT REGEXP</code> is supported</td>
    </tr>
    <tr>
      <td><code>NOT</code>, <code>!</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NOW()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NTH_VALUE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NTILE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>NULLIF()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>OCT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>OCTET_LENGTH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ORD()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>OR</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>PERCENT_RANK()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>PERIOD_ADD()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>PERIOD_DIFF()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>PI()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>POSITION()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>POW()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>POWER()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>PS_CURRENT_THREAD_ID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>PS_THREAD_ID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>Point()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>Polygon()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>QUARTER()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>QUOTE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RADIANS()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RAND()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RANDOM_BYTES()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RANK()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REGEXP_INSTR()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REGEXP_LIKE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REGEXP_MATCHES()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REGEXP_REPLACE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REGEXP_SUBSTR()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REGEXP</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RELEASE_ALL_LOCKS()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RELEASE_LOCK()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REPEAT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REPLACE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REVERSE()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RIGHT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RLIKE</code></td>
      <td class="red">X</td>
      <td><code>REGEXP</code> is supported</td>
    </tr>
    <tr>
      <td><code>ROLES_GRAPHML()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ROUND()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ROW_COUNT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ROW_NUMBER()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RPAD()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RTRIM()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SCHEMA()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SECOND()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SEC_TO_TIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SESSION_USER()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHA()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHA1()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHA2()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SIGN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SIN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SLEEP()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SOUNDEX()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SPACE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SPLIT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SQRT()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STATEMENT_DIGEST()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STATEMENT_DIGEST_TEXT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STD()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STDDEV()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STDDEV_POP()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STDDEV_SAMP()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STRCMP()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>STR_TO_DATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Area()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_AsBinary()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_AsGeoJSON()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_AsText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Buffer()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Buffer_Strategy()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Centroid()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Contains()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_ConvexHull()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Crosses()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Difference()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Dimension()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Disjoint()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Distance()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Distance_Sphere()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_EndPoint()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Envelope()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Equals()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_ExteriorRing()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeoHash()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeomCollFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeomCollFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeomFromGeoJSON()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeomFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeomFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeometryN()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_GeometryType()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_InteriorRingN()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Intersection()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Intersects()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_IsClosed()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_IsEmpty()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_IsSimple()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_IsValid()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_LatFromGeoHash()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Latitude()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Length()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_LineFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_LineFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_LongFromGeoHash()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Longitude()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_MLineFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_MLineFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_MPointFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_MPointFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_MPolyFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_MPolyFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_MakeEnvelope()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_NumGeometries()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_NumInteriorRing()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_NumPoints()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Overlaps()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_PointFromGeoHash()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_PointFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_PointFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_PointN()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_PolyFromText()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_PolyFromWKB()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_SRID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Simplify()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_StartPoint()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_SwapXY()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_SymDifference()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Touches()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Transform()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Union()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Validate()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Within()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_X()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ST_Y()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SUBDATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SUBSTR()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SUBSTRING()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SUBSTRING_INDEX()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SUBTIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SUM()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SYSDATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SYSTEM_USER()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TAN()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TIMEDIFF()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TIMESTAMP()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TIMESTAMPADD()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TIMESTAMPDIFF()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TIME_FORMAT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TIME_TO_SEC()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TO_BASE64()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TO_DAYS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TO_SECONDS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TRIM()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>TRUNCATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UCASE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UNCOMPRESS()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UNCOMPRESSED_LENGTH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UNHEX()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UNIX_TIMESTAMP()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UPPER()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>USER()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UTC_DATE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UTC_TIME()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UTC_TIMESTAMP()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UUID()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UUID_SHORT()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UUID_TO_BIN()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>UpdateXML()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>VALIDATE_PASSWORD_STRENGTH()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>VALUES()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>VARIANCE()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>VAR_POP()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>VAR_SAMP()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>VERSION()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>WAIT_FOR_EXECUTED_GTID_SET()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>WEEK()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>WEEKDAY()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>WEEKOFYEAR()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>WEIGHT_STRING()</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>YEAR()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>YEARWEEK()</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

# Permissions

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Users</td>
      <td class="orange">O</td>
      <td>Only one user is configurable, and must be specified in the config file at startup.</td>
    </tr>
    <tr>
      <td>Privileges</td>
      <td class="red">X</td>
      <td>Only one user is configurable, and they have all privileges.</td>
    </tr>
  </tbody>
</table>

## Misc features

<table class="sql-table">
  <thead>
    <tr>
      <th>Component</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Information schema</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Views</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td>Window functions</td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td>Common table expressions (CTEs)</td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td>Stored procedures</td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td>Cursors</td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td>Triggers</td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Collations and character sets

Dolt currently only supports a single collation and character set, the
same one that Go uses: `utf8_bin` and `utf8mb4`. We will add support
for more character sets and collations as required by
customers. Please [file an
issue](https://github.com/dolthub/dolt/issues) explaining your
use case if current character set and collation support isn't
sufficient.

# Supported Statements

Dolt's goal is to be a drop-in replacement for MySQL, with every query
and statement that works in MySQL behaving identically in Dolt. For
most syntax and technical questions, you should feel free to refer to
the [MySQL user
manual](https://dev.mysql.com/doc/refman/8.0/en/select.html). Any
deviation from the MySQL manual should be documented on this page, or
else indicates a bug. Please [file
issues](https://github.com/dolthub/dolt/issues) with any
incompatibilities you discover.

## Data manipulation statements

<table class="sql-table">
  <thead>
    <tr>
      <th>Statement</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>CALL</code></td>
      <td class="red">X</td>
      <td>Stored procedures are not yet implemented.</td>
    </tr>
    <tr>
      <td><code>CREATE TABLE AS</code></td>
      <td class="red">X</td>
      <td><code>INSERT INTO SELECT *</code> is supported.</td>
    </tr>
    <tr>
      <td><code>CREATE TABLE LIKE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DO</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DELETE</code></td>
      <td class="green">✓</td>
      <td>No support for referring to  more than one table in a single <code>DELETE</code> statement.</td>
    </tr>
    <tr>
      <td><code>HANDLER</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>IMPORT TABLE</code></td>
      <td class="red">X</td>
      <td>Use <code>dolt table import</code></td>
    </tr>
    <tr>
      <td><code>INSERT</code></td>
      <td class="green">✓</td>
      <td>No support for <code>ON DUPLICATE KEY</code> clauses</td>
    </tr>
    <tr>
      <td><code>LOAD DATA</code></td>
      <td class="red">X</td>
      <td>Use <code>dolt table import</code></td>
    </tr>
    <tr>
      <td><code>LOAD XML</code></td>
      <td class="red">X</td>
      <td>Use <code>dolt table import</code></td>
    </tr>
    <tr>
      <td><code>REPLACE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SELECT</code></td>
      <td class="green">✓</td>
      <td>Most select statements, including <code>UNION</code> and <code>JOIN</code>, are supported.</td>
    </tr>
    <tr>
      <td><code>SELECT FROM AS OF</code></td>
      <td class="green">✓</td>
      <td>Selecting from a table as of any known revision or commit timestamp is supported. See <a href="#querying-non-head-revisions-of-a-database">AS OF queries</a>.</td>
    </tr>
    <tr>
      <td><code>SELECT FOR UPDATE</code></td>
      <td class="red">X</td>
      <td>Locking and concurrency are currently very limited.</td>
    </tr>
    <tr>
      <td><code>SUBQUERIES</code></td>
      <td class="green">✓</td>
      <td>Subqueries work, but must be given aliases. Some limitations apply.</td>
    </tr>
    <tr>
      <td><code>TABLE</code></td>
      <td class="red">X</td>
      <td>Equivalent to <code>SELECT * FROM TABLE</code> without a <code>WHERE</code> clause.</td>
    </tr>
    <tr>
      <td><code>TRUNCATE</code></td>
      <td class="red">X</td>
      <td><code>DELETE FROM table</code> with no <code>WHERE</code> clause will delete all rows in constant time from the <code>dolt sql</code> shell.</td>
    </tr>
    <tr>
      <td><code>UPDATE</code></td>
      <td class="green">✓</td>
      <td>No support for referring to more than one table in a single <code>UPDATE</code> statement.</td>
    </tr>
    <tr>
      <td><code>VALUES</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>WITH</code></td>
      <td class="red">X</td>
      <td>Common table expressions (CTEs) are not yet supported</td>
    </tr>
  </tbody>
</table>

## Data definition statements

<table class="sql-table">
  <thead>
    <tr>
      <th>Statement</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>ADD COLUMN</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ADD CHECK</code></td>
      <td class="red">X</td>
      <td><code>NOT NULL</code> is the only check currently possible.</td>
    </tr>
    <tr>
      <td><code>ADD CONSTRAINT</code></td>
      <td class="red">X</td>
      <td><code>NOT NULL</code> is the only constraint currently possible.</td>
    </tr>
    <tr>
      <td><code>ADD FOREIGN KEY</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ADD PARTITION</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ALTER COLUMN</code></td>
      <td class="green">✓</td>
      <td>Name and order changes are supported, but not type or primary key changes.</td>
    </tr>
    <tr>
      <td><code>ALTER DATABASE</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ALTER INDEX</code></td>
      <td class="red">X</td>
      <td>Indexes can be created and dropped, but not altered.</td>
    </tr>
    <tr>
      <td><code>ALTER PRIMARY KEY</code></td>
      <td class="red">X</td>
      <td>Primary keys of tables cannot be changed.</td>
    </tr>
    <tr>
      <td><code>ALTER TABLE</code></td>
      <td class="green">✓</td>
      <td>Not all <code>ALTER TABLE</code> statements are supported. See the rest of this table for details.</td>
    </tr>
    <tr>
      <td><code>ALTER TYPE</code></td>
      <td class="red">X</td>
      <td>Column type changes are not supported.</td>
    </tr>
    <tr>
      <td><code>ALTER VIEW</code></td>
      <td class="red">X</td>
      <td>Views can be created and dropped, but not altered.</td>
    </tr>
    <tr>
      <td><code>CHANGE COLUMN</code></td>
      <td><span style="color:orange">O</span></td>
      <td>Columns can be renamed and reordered, but type changes are not implemented.</td>
    </tr>
    <tr>
      <td><code>CREATE DATABASE</code></td>
      <td class="red">X</td>
      <td>Create new repositories with <code>dolt clone</code> or <code>dolt init</code></td>
    </tr>
    <tr>
      <td><code>CREATE EVENT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE FUNCTION</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE INDEX</code></td>
      <td><span style="color:orange">O</span></td>
      <td>Unique indexes are not yet supported. Fulltext and spatial indexes are not supported.</td>
    </tr>
    <tr>
      <td><code>CREATE TABLE</code></td>
      <td class="green">✓</td>
      <td>Tables must have primary keys.</td>
    </tr>
    <tr>
      <td><code>CREATE TABLE AS</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE TRIGGER</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE VIEW</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DESCRIBE TABLE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP COLUMN</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP CONSTRAINT</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP DATABASE</code></td>
      <td class="red">X</td>
      <td>Delete a repository by deleting its directory on disk.</td>
    </tr>
    <tr>
      <td><code>DROP EVENT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP FUNCTION</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP INDEX</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP TABLE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP PARTITION</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP TRIGGER</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP VIEW</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>MODIFY COLUMN</code></td>
      <td><span style="color:orange">O</span></td>
      <td>Columns can be renamed and reordered, but type changes are not implemented.</td>
    </tr>
    <tr>
      <td><code>RENAME COLUMN</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RENAME CONSTRAINT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RENAME DATABASE</code></td>
      <td class="red">X</td>
      <td>Database names are read-only, but can be configured in the server config.</td>
    </tr>
    <tr>
      <td><code>RENAME INDEX</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RENAME TABLE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW COLUMNS</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW CONSTRAINTS</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW CREATE TABLE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW CREATE VIEW</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW DATABASES</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW INDEX</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW SCHEMAS</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SHOW TABLES</code></td>
      <td class="green">✓</td>
      <td><code>SHOW FULL TABLES</code> reveals whether a table is a base table or a view.</td>
    </tr>
    <tr>
      <td><code>TRUNCATE TABLE</code></td>
      <td class="red">X</td>
      <td><code>DELETE FROM table</code> with no <code>WHERE</code> clause will delete all rows in constant time from the <code>dolt sql</code> shell.</td>
    </tr>
  </tbody>
</table>

## Transactional statements

Transactional semantics are a work in progress. Dolt isn't like other
databases: a "commit" in dolt creates a new entry in the repository
revision graph, as opposed to updating one or more rows atomically as
in other databases. By default, updating data through SQL statements
modifies the working set of the repository. Committing changes to the
repository requires the use of `dolt add` and `dolt commmit` from the
command line. But it's also possible for advanced users to create
commits and branches directly through SQL statements.

Not much work has been put into supporting the true transaction and
concurrency primitives necessary to be an application server, but
limited support for transactions does exist. Specifically, when
running the SQL server with `@@autocommit = false`, the working set
will not be updated with changes until a `COMMIT` statement is
executed.

<table class="sql-table">
  <thead>
    <tr>
      <th>Statement</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>BEGIN</code></td>
      <td><span style="color:orange">O</span></td>
      <td><code>BEGIN</code> parses correctly, but is a no-op: it doesn&#39;t create a checkpoint that can be returned to with <code>ROLLBACK</code>.</td>
    </tr>
    <tr>
      <td><code>COMMIT</code></td>
      <td class="green">✓</td>
      <td><code>COMMIT</code> will write any pending changes to the working set when <code>@@autocommit = false</code></td>
    </tr>
    <tr>
      <td><code>COMMIT(MESSAGE)</code></td>
      <td class="green">✓</td>
      <td>The <code>COMMIT()</code> function creates a commit of the current database state and returns the hash of this new commit. See <a href="#concurrency">concurrency</a> for details.</td>
    </tr>
    <tr>
      <td><code>LOCK TABLES</code></td>
      <td class="red">X</td>
      <td><code>LOCK TABLES</code> parses correctly but does not prevent access to those tables from other sessions.</td>
    </tr>
    <tr>
      <td><code>ROLLBACK</code></td>
      <td class="red">X</td>
      <td><code>ROLLBACK</code> parses correctly but is a no-op.</td>
    </tr>
    <tr>
      <td><code>SAVEPOINT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RELEASE SAVEPOINT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>ROLLBACK TO SAVEPOINT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SET @@autocommit = 1</code></td>
      <td class="green">✓</td>
      <td>When <code>@@autocommit = true</code>, changes to data will update the working set after every statement. When <code>@@autocommit = false</code>, the working set will only be updated after <code>COMMIT</code> statements.</td>
    </tr>
    <tr>
      <td><code>SET TRANSACTION</code></td>
      <td class="red">X</td>
      <td>Different isolation levels are not yet supported.</td>
    </tr>
    <tr>
      <td><code>START TRANSACTION</code></td>
      <td><span style="color:orange">O</span></td>
      <td><code>START TRANSACTION</code> parses correctly, but is a no-op: it doesn&#39;t create a checkpoint that can be returned to with <code>ROLLBACK</code>.</td>
    </tr>
    <tr>
      <td><code>UNLOCK TABLES</code></td>
      <td class="green">✓</td>
      <td><code>UNLOCK TABLES</code> parses correctly, but since <code>LOCK TABLES</code> doesn&#39;t prevent concurrent access it&#39;s essentially a no-op.</td>
    </tr>
  </tbody>
</table>

## Prepared statements

<table class="sql-table">
  <thead>
    <tr>
      <th>Statement</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>PREPARE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>EXECUTE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Access management statements

Access management via SQL statements is not yet supported. This table
will be updated as access management features are implemented. Please
[file an issue](https://github.com/dolthub/dolt/issues) if lack
of SQL access management is blocking your use of Dolt, and we will
prioritize accordingly.

A root user name and password can be specified in the config for
[`sql-server`](/cli/dolt-sql-server/). This user has full privileges
on the running database.

<table class="sql-table">
  <thead>
    <tr>
      <th>Statement</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>ALTER USER</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE ROLE</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>CREATE USER</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP ROLE</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>DROP USER</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>GRANT</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>RENAME USER</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>REVOKE</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SET DEFAULT ROLE</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SET PASSWORD</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SET ROLE</code></td>
      <td class="red">X</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Session management statements

<table class="sql-table">
  <thead>
    <tr>
      <th>Statement</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>SET</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>SET CHARACTER SET</code></td>
      <td class="orange">O</td>
      <td>`SET CHARACTER SET` parses correctly, but Dolt supports only the `utf8mb4` collation</td>
    </tr>
    <tr>
      <td><code>SET NAMES</code></td>
      <td class="orange">O</td>
      <td>`SET NAMES` parses correctly, but Dolt supports only the `utf8mb4` collation for identifiers</td>
    </tr>
    <tr>
      <td><code>KILL QUERY</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

## Utility statements

<table class="sql-table">
  <thead>
    <tr>
      <th>Statement</th>
      <th>Supported</th>
      <th>Notes and limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>EXPLAIN</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
    <tr>
      <td><code>USE</code></td>
      <td class="green">✓</td>
      <td></td>
    </tr>
  </tbody>
</table>

# Concurrency

Currently, the only way the Dolt SQL server interface can handle modifications from multiple clients is by pushing some of the complexity onto the users. By default this mode is disabled, and autocommit mode is enabled, but concurrent connections can be enabled using either the `dolt sql-server` command line arguments, or the supported YAML configuration file. Read the [dolt sql-server documentation](../cli/#sql-server) for details.

### @@dbname_head

The session variable `dbname_head` (Where dbname is the name of the database) provides an interface for reading and writing the HEAD commit for a session.

```sql
# Set the head commit to a specific hash.
SET @@mydb_head = 'fe31vq5c0qj1afnghl0d9448652smlo0';

# Get the head commit
SELECT @@mydb_head;
```

### HASHOF()

The HASHOF function returns the hash of a branch such as `HASHOF("master")`.

### COMMIT()

The COMMIT function writes a new commit to the database and returns the hash of that commit. The argument passed to the function is the commit message. The author's name and email for this commit will be determined by the server or can be provided by the user.
on the repo.

Dolt provides a manual commit mode where a user works with a detached HEAD whose value is accessible and modifiable through the session variable @@dbname_head (where dbname is the name of the database whose pointer you wish to read or write). You can write new commits to the database by inserting and updating rows in the dolt_branches table. See below for details on how this works.

See the below examples as well as the section on [concurrency](https://www.dolthub.com/docs/reference/sql/#concurrency) for details.

Example:

```sql
SET @@mydb_head = COMMIT('-m', 'my commit message');
```

## Options

-m, --message: Use the given `<msg>` as the commit message. Required

-a: Stages all tables with changes before committing

--allow-empty: Allow recording a commit that has the exact same data as its sole parent. This is usually a mistake, so it is disabled by default. This option bypasses that safety.

--date: Specify the date used in the commit. If not specified the current system time is used.

--author: Specify an explicit author using the standard "A U Thor author@example.com" format.


### MERGE()

The MERGE function merges a branch reference into HEAD. The argument passed to the function is a reference to a branch (its name). The author's name and email for this commit will be determined by the server or can be provided by the user. 

Example:

```sql
SET @@mydb_head = MERGE('feature-branch');
```

## Options

--author: Specify an explicit author using the standard "A U Thor author@example.com" format.

### dolt_branches

dolt_branches is a system table that can be used to create, modify and delete branches in a dolt data repository via
SQL.

### Putting it all together

An example showing how to make modifications and create a new feature branch from those modifications.

```sql
-- Set the current database
USE mydb;

-- Set the HEAD commit to the latest commit of the branch "master"
SET @@mydb_head = HASHOF("master");

-- Make modifications
UPDATE table
SET column = "new value"
WHERE pk = "key";

-- Create a new commit containing these modifications and set the HEAD commit for this session to that commit
SET @@mydb_head = COMMIT("modified something")

-- Create a new branch with these changes
INSERT INTO dolt_branches (name,hash)
VALUES ("new_branch", @@mydb_head);
```

An example attempting to change the value of master, but only if nobody else has modified it since we read it.

```sql
-- Set the current database for the session
USE mydb;

-- Set the HEAD commit to the latest commit to the branch "master"
SET @@mydb_head = HASHOF("master");

-- Make modifications
UPDATE table
SET column = "new value"
WHERE pk = "key";

-- Modify master if nobody else has changed it
UPDATE dolt_branches
SET hash = COMMIT("modified something")
WHERE name == "master" and hash == @@mydb_head;

-- Set the HEAD commit to the latest commit to the branch "master" which we just wrote
SET @@mydb_head = HASHOF("master");
```

An example of merging in a feature branch.

```sql

-- Set the current database for the session
USE mydb;

-- Set the HEAD commit to the latest commit to the branch "feature-branch"
SET @@mydb_head = HASHOF("feature-branch");

-- Make modifications
UPDATE table
SET column = "new value"
WHERE pk = "key";

-- MERGE the feature-branch into master and get a commit
SET @@mydb_head = MERGE('feature-branch');

-- Set the HEAD commit to the latest commit to the branch "master" which we just wrote
INSERT INTO dolt_branches (name, hash)
VALUES("master", @@bug_head);

```

# Benchmarks

This section provides benchmarks for Dolt. The current version of Dolt is 0.22.11, and we benchmark against MySQL 5.7.

## Data

Here we present the result of running `sysbench` MySQL tests against Dolt SQL for the most recent release of Dolt. We will update this with every release. The tests attempt to run as many queries as possible in a fixed 10 second time window. The `Dolt` and `MySQL` columns show how many transactions each database completed respectively in that time window.

Dolt is slower than MySQL. The goal is to get Dolt to within 2-4 times the speed of MySQL common operations. If a query takes MySQL 1 second, we expect it to take Dolt 2-4 seconds. Or, if MySQL can run 8 queries in 10 seconds, then we want Dolt to run 2-4 queries in 10 seconds. The `multiple` column represents this relationship.

|Test|Dolt|MySQL|Multiple|
| :-------------------- | :----- | :----- | :------- |
|bulk_insert|161120|324772|2.0|
|oltp_delete|683|18982|27.8|
|oltp_point_select|3883|27402|7.0|
|oltp_read_only|178|1701|9.6|
|oltp_read_write|76|1179|15.5|
|oltp_update_index|470|6155|13.1|
|oltp_update_non_index|624|6447|10.3|
|oltp_write_only|115|3120|27.1|
|select_random_points|287|3993|13.9|
|select_random_ranges|178|5794|32.6|
| _mean_                |        |        | _15.9_   |

In the spirit of ["dog fooding"](https://en.wikipedia.org/wiki/Eating_your_own_dog_food) we created a Dolt database on [DoltHub](https://www.dolthub.com/repositories/dolthub/dolt-benchmarks) with our performance metrics. You can find the full set of metrics produced by `sysbench` there, and explore them via our SQL console.

## Approach

We adopted an industry standard benchmarking tool, [`sysbench`](https://github.com/akopytov/sysbench). `sysbench` provides a series of benchmarks for examining various aspects of database performance, and was authored by developers who worked on MySQL.

You can read more about our benchmarking approach [here](https://github.com/dolthub/dolt/tree/master/benchmark/perf_tools). The basic idea is to provide our developers and contributors with simple tools for producing robust comparisons of Dolt SQL against MySQL. We do this by building and running on all benchmarks on the same hardware and associating the run with a unique identifier with each invocation of `run_benchmarks.sh`, our wrapper.

For example, suppose that a developer made a bunch of changes that are supposed to speed up bulk inserts:

```
$ cd $DOLT_CHECKOUT/benchmark/perf_tools
$ ./run_benchmarks.sh \
  bulk_insert \
  10000 \
  <username> \
  current
```

This will do the following:

- build a copy of Dolt using the locally checked out Git repository located at `$DOLT_CHECKOUT` inside a Docker container
- build and launch a Docker container running Dolt SQL
- build and launch a Docker container running `sysbench` that executes `bulk_insert` using a table with 10000 records for the test
- repeat the process using MySQL for comparison

All of the data produced will be associated with a unique run ID.

## Code

The benchmarking tools are part of Dolt, which is free and open source. You can find a more detailed description of the tools on [GitHub](https://github.com/dolthub/dolt/tree/master/benchmark/perf_tools).