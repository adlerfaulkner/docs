---
title: Writing to Dolt
---

# Introduction

Uploading a file to DoltHub is the lowest barrier to entry for putting data into Dolt. Every repository has a button to upload a file, and you can learn more about those specific steps [here](/tutorials/dolthub/#data-publishing).

In the last section we saw how to read data from Dolt using several familiar interfaces. Those interfaces were:

- the Dolt command line interface (CLI) that will feel familiar to Git users
- a SQL interface (via the shell or by sending queries to a server process)
- a Python API that is comfortable for folks that use Python elsewhere in their stack
- using R on top of an existing MySQL package.

This tutorial will walk through how to put data into a Dolt database using the same interfaces. For each interface we will initially write the following CSV to create some sample data to work with:

```
$ cat great_players.csv
name,id
rafa,1
roger,2
novak,3
```

And then update the data with the as follows to illustrate some of Dolt's more unique features:

```
$ cat great_players_with_majors.csv
name,id
rafa,1
roger,2
novak,3
andy,4
```

# CLI

First let's create a new Dolt database:

```
$ mkdir tennis-players && cd tennis-players
$ dolt init
Successfully initialized dolt data repository.
$ ls
../    ./     .dolt/
```

This created our database, now let's load in our initial data:

```
$ dolt table import -c --pk id great_players great_players.csv
Rows Processed: 3, Additions: 3, Modifications: 0, Had No Effect: 0
Import completed successfully.
$ dolt status
On branch master
Untracked files:
  (use "dolt add <table|doc>" to include in what will be committed)
	new table:      great_players
```

Now let's generate a commit for that data:

```
$ dolt add great_players && dolt commit -m 'Added some great players'
.
.
.
```

Now suppose that we would like to add a player:

```
$ cat great_players.csv
name,id
rafa,1
roger,2
novak,3
andy,4
$ dolt table import -u great_players great_players.csv
Rows Processed: 4, Additions: 1, Modifications: 0, Had No Effect: 3
Import completed successfully.
$ dolt diff
diff --dolt a/great_players b/great_players
--- a/great_players @ c2tpkad9e5345sjq2h7e6d9pdp7383a6
+++ b/great_players @ 6sdov20gjulkskcm9t97qmbq9dkj3l5v
+-----+------+----+
|     | name | id |
+-----+------+----+
|  +  | andy | 4  |
+-----+------+----+
$ dolt add great_players && dolt commit -m 'do not forget Andy!'
```

We were able to reimport our file, and let Dolt figure out the differences.

We just saw a simple example of how to create and import data into the Dolt data format. We now use the same example to illustrate alternative write interfaces.

# SQL Shell

First let's create a new database:

```
$ mkdir tennis-players && cd tennis-players
$ dolt init
Successfully initialized dolt data repository.
$ ls
../    ./     .dolt/
```

Now let's get into the SQL console and create a table, noting that we specify a primary key column as Dolt requires a primary key (for now):

```
$ dolt sql
# Welcome to the DoltSQL shell.
# Statements must be terminated with ';'.
# "exit" or "quit" (or Ctrl-D) to exit.
tennis_players> CREATE TABLE great_players (name VARCHAR(32), id INT, PRIMARY KEY (id));
tennis_players> DESCRIBE great_players;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| name  | VARCHAR(32) | YES  |     |         |       |
| id    | INT         | NO   | PRI |         |       |
+-------+-------------+------+-----+---------+-------+
```

Now let's execute some insert statements to get some data in there:

```
tennis_players> INSERT INTO great_players VALUES ("rafa", 1);
Query OK, 1 row affected
tennis_players> INSERT INTO great_players VALUES ("roger", 2);
Query OK, 1 row affected
tennis_players> INSERT INTO great_players VALUES ("novak", 3);
Query OK, 1 row affected
```

We can go to the command line to check the status of our tables, which will show we have created a new table that is now in our working set:

```
$ dolt status
On branch master
Untracked files:
  (use "dolt add <table|doc>" to include in what will be committed)
	new table:      great_players
```

Then we can execute the usual Git-like workflow:

```
$ dolt add great_players
$ dolt commit -m 'Added some great players'
.
.
.
```

Now let's append a row and generate another commit

```
$ dolt sql
# Welcome to the DoltSQL shell.
# Statements must be terminated with ';'.
# "exit" or "quit" (or Ctrl-D) to exit.
tennis_players> INSERT INTO great_players VALUES ("andy", 4);
Query OK, 1 row affected
```

And again we can generate a commit:

```
$ dolt add great_players
$ dolt commit -m 'Added Andy!'
.
.
.
```

We just executed an identical set of updates to our database using pure SQL.

# Python

First let's create a new database, which can be done from Python using a convenience function:

```python
from doltpy.core import Dolt

dolt = Dolt.init('~/temp/tennis-players')
```

Now we can use the bulk import function:

```python
from doltpy.core.write  import bulk_import
bulk_import(dolt, 'great_players', open('path/to/great_players.csv'), ['id'], 'create')
```

Or if we would prefer to use Pandas:

```python
import pandas as pd
from doltpy.core.write import import_df
import_df(dolt, 'great_players', pd.read_csv('path/to/great_players.csv'), ['id'], 'create')
```

This is exactly equivalent to the CLI commands we saw in the previous section, and you can create a commit similarly:

```python
dolt.add('great_players')
dolt.commit('Added some great players')
```

The update case is again similar:

```python
from doltpy.core.write import bulk_import

bulk_import(dolt, 'great_players', open('path/to/great_players.csv'), ['id'], 'update')
```

Or:

```python
import pandas as pd
from doltpy.core.write import import_df

import_df(dolt, 'great_players', pd.read_csv('path/to/great_players.csv'), ['id'], 'update')
```

And again we generate a commit:

```python
dolt.add('great_players')
dolt.commit('do not forget Andy!')
```

In this section we used the example from both the CLI and SQL sections, but executed our operations in pure Python.

# Summary

Much like the [Reading from Dolt](../reading-from-dolt) tutorial we followed exactly the same steps across three different interfaces. The goal of doing so is to illustrate that Dolt, just like existing relational database solutions, offers a variety of mechanisms for working with the underlying data. Users should choose the one best suited to their particular use-case.
