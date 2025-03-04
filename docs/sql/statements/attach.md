---
layout: docu
title: ATTACH/DETACH Statement
railroad: statements/attach.js
---

The `ATTACH` statement adds a new database file to the catalog that can be read from and written to.

## Examples

```sql
-- attach the database "file.db" with the alias inferred from the name ("file")
ATTACH 'file.db';
-- attach the database "file.db" with an explicit alias ("file_db")
ATTACH 'file.db' AS file_db;
-- attach the database "file.db" in read only mode
ATTACH 'file.db' (READ_ONLY);
-- attach a SQLite database for reading and writing (see sqlite extension for more information)
ATTACH 'sqlite_file.db' AS sqlite (TYPE SQLITE);
-- attach the database "file.db" if inferred database alias "file_db" does not yet exist
ATTACH IF NOT EXISTS 'file.db';
-- attach the database "file.db" if explicit database alias "file_db" does not yet exist
ATTACH IF NOT EXISTS 'file.db' AS file_db;
-- create a table in the attached database with alias "file"
CREATE TABLE file.new_table (i INTEGER);
-- detach the database with alias "file"
DETACH file;
-- show a list of all attached databases
SHOW DATABASES;
-- change the default database that is used to the database "file"
USE file;
```

## Syntax

<div id="rrdiagram1"></div>

`ATTACH` allows DuckDB to operate on multiple database files, and allows for transfer of data between different database files. 

## Detach

<div id="rrdiagram2"></div>

The `DETACH` statement allows previously attached database files to be closed and detached, releasing any locks held on the database file. It is not possible to detach from the default database: if you would like to do so, issue the [`USE` statement](use) to change the default database to another one.

> Closing the connection, e.g., invoking the [`close()` function in Python](../../api/python/dbapi#connection), does not release the locks held on the database files as the file handles are held by the main DuckDB instance (in Python's case, the `duckdb` module).

## Name Qualification

The fully qualified name of catalog objects contains the *catalog*, the *schema* and the *name* of the object. For example:

```sql
-- attach the database "new_db"
ATTACH 'new_db.db';
-- create the schema "my_schema" in the database "new_db"
CREATE SCHEMA new_db.my_schema;
-- create the table "my_table" in the schema "my_schema"
CREATE TABLE new_db.my_schema.my_table (col INTEGER);
-- refer to the column "col" inside the table "my_table"
SELECT new_db.my_schema.my_table.col FROM new_db.my_schema.my_table;
```

Note that often the fully qualified name is not required. When a name is not fully qualified, the system looks for which entries to reference using the *catalog search path*. The default catalog search path includes the system catalog, the temporary catalog and the initially attached database together with the `main` schema.

Also note the rules on [identifiers and database names in particular](../keywords-and-identifiers#database-names).

### Default Database and Schema

When a table is created without any qualifications, the table is created in the default schema of the default database. The default database is the database that is launched when the system is created - and the default schema is `main`.

```sql
-- create the table "my_table" in the default database
CREATE TABLE my_table (col INTEGER);
```

### Changing the Default Database and Schema

The default database and schema can be changed using the `USE` command.

```sql
-- set the default database schema to `new_db.main`
USE new_db;
-- set the default database schema to `new_db.my_schema`
USE new_db.my_schema;
```

### Resolving Conflicts

When providing only a single qualification, the system can interpret this as *either* a catalog *or* a schema, as long as there are no conflicts. For example:

```sql
ATTACH 'new_db.db';
CREATE SCHEMA my_schema;
-- creates the table "new_db.main.tbl"
CREATE TABLE new_db.tbl (i INTEGER);
-- creates the table "default_db.my_schema.tbl"
CREATE TABLE my_schema.tbl (i INTEGER);
```

If we create a conflict (i.e., we have both a schema and a catalog with the same name) the system requests that a fully qualified path is used instead:

```sql
CREATE SCHEMA new_db;
CREATE TABLE new_db.tbl (i INTEGER);
-- Error: Binder Error: Ambiguous reference to catalog or schema "new_db" - use a fully qualified path like "memory.new_db"
```

### Changing the Catalog Search Path

The catalog search path can be adjusted by setting the `search_path` configuration option, which uses a comma-separated list of values that will be on the search path. The following example demonstrates searching in two databases:

```sql
ATTACH ':memory:' AS db1;
ATTACH ':memory:' AS db2;
CREATE table db1.tbl1 (i INTEGER);
CREATE table db2.tbl2 (j INTEGER);
-- reference the tables using their fully qualified name
SELECT * FROM db1.tbl1;
SELECT * FROM db2.tbl2;
-- or set the search path and reference the tables using their name
SET search_path = 'db1,db2';
SELECT * FROM tbl1;
SELECT * FROM tbl2;
```

## Transactional Semantics

When running queries on multiple databases, the system opens separate transactions per database. The transactions are started *lazily* by default - when a given database is referenced for the first time in a query, a transaction for that database will be started. `SET immediate_transaction_mode = true` can be toggled to change this behavior to eagerly start transactions in all attached databases instead.

While multiple transactions can be active at a time - the system only supports *writing* to a single attached database in a single transaction. If you try to write to multiple attached databases in a single transaction the following error will be thrown:

```text
Attempting to write to database "db2" in a transaction that has already modified database "db1" - a single transaction can only write to a single attached database.
```

The reason for this restriction is that the system does not maintain atomicity for transactions across attached databases. Transactions are only atomic *within* each database file. By restricting the global transaction to write to only a single database file the atomicity guarantees are maintained.
