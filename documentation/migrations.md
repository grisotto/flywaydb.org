---
layout: documentation
menu: migrations
subtitle: Migrations
---
# Migrations

<div id="toc"></div>

## Overview

With Flyway all changes to the database are called **migrations**. Migrations can be either *versioned* or
*repeatable*. Versioned migrations come in 2 forms: regular and *undo*.

**Versioned migrations** have a *version*, a *description* and a *checksum*. The version must be unique. The description is purely
informative for you to be able to remember what each migration does. The checksum is there to detect accidental changes.
Versioned migrations are the most common type of migration. They are applied in order exactly once.

Optionally their effect can be undone by supplying an **undo migration** with the same version.
 
**Repeatable migrations** have a description and a checksum, but no version. Instead of being run just once, they are
(re-)applied every time their checksum changes.

Within a single migration run, repeatable migrations are always applied last, after all pending versioned migrations 
have been executed. Repeatable migrations are applied in the order of their description.

By default both versioned and repeatable migrations can be written either in **[SQL](/documentation/migrations#sql-based-migrations)**
or in **[Java](/documentation/migrations#java-based-migrations)** and can consist of multiple statements.

Flyway automatically discovers migrations on the *filesystem* and on the Java *classpath*.

To keep track of which migrations have already been applied when and by whom, Flyway adds a *schema history table*
to your schema.

## Versioned Migrations

The most common type of migration is a **versioned migration**. Each versioned migration has a *version*, a *description*
and a *checksum*. The version must be unique. The description is purely
informative for you to be able to remember what each migration does. The checksum is there to detect accidental changes.
Versioned migrations are applied in order exactly once.

Versioned migrations are typically used for:
- Creating/altering/dropping tables/indexes/foreign keys/enums/UDTs/...
- Reference data updates
- User data corrections

Here is a small example:

```sql
CREATE TABLE car (
    id INT NOT NULL PRIMARY KEY,
    license_plate VARCHAR NOT NULL,
    color VARCHAR NOT NULL
);

ALTER TABLE owner ADD driver_license_id VARCHAR;

INSERT INTO brand (name) VALUES ('DeLorean');
```

Each versioned migration must be assigned a **unique version**. Any version is valid as long as it conforms to the usual
dotted notation. For most cases a simple increasing integer should be all you need. However Flyway is quite flexible and
all these versions are valid versioned migration versions:
- 1
- 001
- 5.2
- 1.2.3.4.5.6.7.8.9
- 205.68
- 20130115113556
- 2013.1.15.11.35.56
- 2013.01.15.11.35.56

Versioned migrations are applied in the order of their versions. Versions are sorted numerically as you would normally expect.

## Undo Migrations
{% include pro.html %}

**Undo migrations** are the opposite of regular versioned migrations. An undo migration is responsible for undoing the effects
of the versioned migration with the same version. Undo migrations are optional and not required to run regular versioned migrations.

For the example above, this is how the undo migration would look like:
```sql
DELETE FROM brand WHERE name='DeLorean';

ALTER TABLE owner DROP driver_license_id;

DROP TABLE car;
```

### Important Notes

While the idea of undo migrations is nice, unfortunately it sometimes breaks down in practice. As soon as
you have destructive changes (drop, delete, truncate, ...), you start getting into trouble. And even if you don't,
you end up creating home-made alternatives for restoring backups, which need to be properly tested as well.

Undo migrations assume the whole migration succeeded and should now be undone. This does not help with failed versioned
migrations on databases without DDL transactions. Why? A migration can fail at any point. If you have 10 statements,
it is possible for the 1st, the 5th, the 7th or the 10th to fail. There is
simply no way to know in advance. In contrast, undo migrations are written to undo an entire versioned migration and will not
help under such conditions.

An alternative approach which we find preferable is to **maintain backwards compatibility
between the DB and all versions of the code currently deployed in production**. This way a
failed migration is not a disaster. The old version of the application is still compatible with the DB, so you
can simply roll back the application code, investigate, and take corrective measures.

This should be complemented with a **proper, well tested, backup and restore strategy**. It is independent
of the database structure, and once it is tested and proven to work, no migration script can break it. For
optimal performance, and if your infrastructure supports this, we recommend using the snapshot
technology of your underlying storage solution. Especially for larger data volumes, this can be
several orders of magnitude faster than traditional backups and restores.

## Repeatable Migrations

**Repeatable migrations** have a description and a checksum, but no version. Instead of being run just once, they are
(re-)applied every time their checksum changes. 

This is very useful for managing database objects whose definition can then simply be maintained in a single file in
version control. They are typically used for
- (Re-)creating views/procedures/functions/packages/...
- Bulk reference data reinserts

Within a single migration run, repeatable migrations are always applied last, after all pending versioned migrations have been executed. Repeatable migrations are applied in the order of their description.

It is your responsibility to ensure the same repeatable migration can be applied multiple times. This usually
involves making use of `CREATE OR REPLACE` clauses in your DDL statements.

Here is an example of what a repeatable migration looks like:

```sql
CREATE OR REPLACE VIEW blue_cars AS 
    SELECT id, license_plate FROM cars WHERE color='blue';
```

## SQL-based migrations

Migrations are most commonly written in **SQL**. This makes it easy to get started and leverage any existing scripts,
tools and skills. It gives you access to the full set of capabilities of your database and eliminates the need to
understand any intermediate translation layer. 

SQL-based migrations are typically used for
- DDL changes (CREATE/ALTER/DROP statements for TABLES,VIEWS,TRIGGERS,SEQUENCES,...)
- Simple reference data changes (CRUD in reference data tables)
- Simple bulk data changes (CRUD in regular data tables)

### Naming

In order to be picked up by Flyway, SQL migrations must comply with the following naming pattern:

<div class="row">
    <div class="col-md-4">
        <h4>Versioned Migrations</h4>
        <pre>Prefix  Separator       Suffix
   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>                <i class="fa fa-long-arrow-down" style="padding: 4px"></i>
   <span style="color: white; font-weight: bold"><span style="background-color: #0000AA; padding: 4px">V</span><span style="background-color: #AA0000; padding: 4px">2</span><span style="background-color: #00AA00; padding: 4px">__</span><span style="background-color: #AAAA00; padding: 4px">Add_new_table</span><span style="background-color: #00AAAA; padding: 4px">.sql</span></span>
     <i class="fa fa-long-arrow-up" style="padding: 4px"></i>         <i class="fa fa-long-arrow-up" style="padding: 4px"></i>
 Version    Description</pre>
    </div>
    <div class="col-md-4">
        <h4>Undo Migrations</h4>
        <pre>Prefix  Separator       Suffix
   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>                <i class="fa fa-long-arrow-down" style="padding: 4px"></i>
   <span style="color: white; font-weight: bold"><span style="background-color: #0000AA; padding: 4px">U</span><span style="background-color: #AA0000; padding: 4px">2</span><span style="background-color: #00AA00; padding: 4px">__</span><span style="background-color: #AAAA00; padding: 4px">Add_new_table</span><span style="background-color: #00AAAA; padding: 4px">.sql</span></span>
     <i class="fa fa-long-arrow-up" style="padding: 4px"></i>         <i class="fa fa-long-arrow-up" style="padding: 4px"></i>
 Version    Description</pre>
    </div>
    <div class="col-md-4">
        <h4>Repeatable Migrations</h4>
        <pre>Prefix Separator       Suffix
    <i class="fa fa-long-arrow-down" style="padding: 4px"></i> <i class="fa fa-long-arrow-down" style="padding: 4px"></i>                <i class="fa fa-long-arrow-down" style="padding: 4px"></i>
    <span style="color: white; font-weight: bold"><span style="background-color: #0000AA; padding: 4px">R</span><span style="background-color: #00AA00; padding: 4px">__</span><span style="background-color: #AAAA00; padding: 4px">Add_new_table</span><span style="background-color: #00AAAA; padding: 4px">.sql</span></span>
               <i class="fa fa-long-arrow-up" style="padding: 4px"></i>
           Description</pre>
    </div>
</div>

The file name consists of the following parts:
- **Prefix**: `V` for versioned ([configurable](/documentation/commandline/migrate#sqlMigrationPrefix)),
`U` for undo ([configurable](/documentation/commandline/migrate#undoSqlMigrationPrefix)) and
`R` for repeatable migrations ([configurable](/documentation/commandline/migrate#repeatableSqlMigrationPrefix))
- **Version**: Version with dots or underscores separate as many parts as you like (Not for repeatable migrations)
- **Separator**: `__` (two underscores) ([configurable](/documentation/commandline/migrate#sqlMigrationSeparator))
- **Description**: Underscores or spaces separate the words
- **Suffix**: `.sql` ([configurable](/documentation/commandline/migrate#sqlMigrationSuffixes))

### Discovery

Flyway discovers SQL-based migrations both on the **filesystem** and on the Java **classpath**. 
Migrations reside in one or more directories referenced by the **[`locations`](/documentation/commandline/migrate#locations)**
property.

Locations with the `filesystem:` prefix target the file system.<br>
Unprefixed locations or locations with the `classpath:` prefix target the Java classpath.

<pre class="filetree"><i class="fa fa-folder-open"></i> my-project
  <i class="fa fa-folder-open"></i> src
    <i class="fa fa-folder-open"></i> main
      <i class="fa fa-folder-open"></i> resources
        <span><i class="fa fa-folder-open"></i> db
  <i class="fa fa-folder-open"></i> migration</span>                <i class="fa fa-long-arrow-left"></i> <code>classpath:db/migration</code> 
            <i class="fa fa-file-text"></i> R__My_view.sql
            <i class="fa fa-file-text"></i> U1.1__Fix_indexes.sql
            <i class="fa fa-file-text"></i> U2__Add a new table.sql
            <i class="fa fa-file-text"></i> V1__Initial_version.sql
            <i class="fa fa-file-text"></i> V1.1__Fix_indexes.sql
            <i class="fa fa-file-text"></i> V2__Add a new table.sql
  <span><i class="fa fa-folder-open"></i> my-other-folder</span>                  <i class="fa fa-long-arrow-left"></i> <code>filesystem:/my-project/my-other-folder</code>
    <i class="fa fa-file-text"></i> U1.2__Add_constraints.sql
    <i class="fa fa-file-text"></i> V1.2__Add_constraints.sql</pre>
    
New SQL-based migrations are **discovered automatically** through filesystem and Java classpath scanning at runtime.
Once you have configured the [`locations`](/documentation/commandline/migrate#locations) you want to use, Flyway will
automatically pick up any new SQL migrations as long as they conform to the configured *naming convention*.

This scanning is recursive. All migrations in non-hidden directories below the specified ones are also picked up.

### Syntax

Flyway supports all regular SQL syntax elements including:
- Single- or multi-line statements
- Single- (--) or Multi-line (/* */) comments spanning complete lines
- Database-specific SQL syntax extensions (PL/SQL, T-SQL, ...) typically used to define stored procedures, packages, ...

Additionally in the case of Oracle, Flyway also supports [SQL*Plus commands](/documentation/database/oracle#sqlplus-commands).

### Placeholder Replacement
In addition to regular SQL syntax, Flyway also supports placeholder replacement with configurable pre- and suffixes.
By default it looks for Ant-style placeholders like `${myplaceholder}`.

This can be very useful to abstract differences between environments.

### Example
Here is a small example of the supported syntax:

```sql
/* Single line comment */
CREATE TABLE test_user (
  name VARCHAR(25) NOT NULL,
  PRIMARY KEY(name)
);

/*
Multi-line
comment
*/

-- Placeholder
INSERT INTO ${tableName} (name) VALUES ('Mr. T');
```

## Java-based migrations

Java-based migrations are a great fit for all changes that can not easily be expressed using SQL.

These would typically be things like
- BLOB &amp; CLOB changes
- Advanced bulk data changes (Recalculations, advanced format changes, ...)

### Naming

In order to be picked up by Flyway, Java-based Migrations must implement the
[`JavaMigration`](/documentation/api/javadoc/org/flywaydb/core/api/migration/JavaMigration) interface. Most users
however should inherit from the convenience class [`BaseJavaMigration`](/documentation/api/javadoc/org/flywaydb/core/api/migration/BaseJavaMigration)
instead as it encourages Flyway's default naming convention, enabling Flyway to automatically extract the version and
the description from the class name. To be able to do so, the class name must comply with the following naming pattern:

<div class="row">
    <div class="col-md-4">
        <h4>Versioned Migrations</h4>
        <pre>Prefix  Separator
   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>
   <span style="color: white; font-weight: bold"><span style="background-color: #0000AA; padding: 4px">V</span><span style="background-color: #AA0000; padding: 4px">2</span><span style="background-color: #00AA00; padding: 4px">__</span><span style="background-color: #AAAA00; padding: 4px">Add_new_table</span></span>
     <i class="fa fa-long-arrow-up" style="padding: 4px"></i>         <i class="fa fa-long-arrow-up" style="padding: 4px"></i>
 Version    Description</pre>
    </div>
    <div class="col-md-4">
        <h4>Undo Migrations</h4>
        <pre>Prefix  Separator
   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>   <i class="fa fa-long-arrow-down" style="padding: 4px"></i>
   <span style="color: white; font-weight: bold"><span style="background-color: #0000AA; padding: 4px">U</span><span style="background-color: #AA0000; padding: 4px">2</span><span style="background-color: #00AA00; padding: 4px">__</span><span style="background-color: #AAAA00; padding: 4px">Add_new_table</span></span>
     <i class="fa fa-long-arrow-up" style="padding: 4px"></i>         <i class="fa fa-long-arrow-up" style="padding: 4px"></i>
 Version    Description</pre>
    </div>
    <div class="col-md-4">
        <h4>Repeatable Migrations</h4>
        <pre>Prefix Separator
    <i class="fa fa-long-arrow-down" style="padding: 4px"></i> <i class="fa fa-long-arrow-down" style="padding: 4px"></i>
    <span style="color: white; font-weight: bold"><span style="background-color: #0000AA; padding: 4px">R</span><span style="background-color: #00AA00; padding: 4px">__</span><span style="background-color: #AAAA00; padding: 4px">Add_new_table</span></span>
               <i class="fa fa-long-arrow-up" style="padding: 4px"></i>
           Description</pre>
    </div>
</div>

The file name consists of the following parts:
- **Prefix**: `V` for versioned migrations, `U` for undo migrations, `R` for repeatable migrations
- **Version**: Underscores (automatically replaced by dots at runtime) separate as many parts as you like (Not for repeatable migrations)
- **Separator**: `__` (two underscores)
- **Description**: Underscores (automatically replaced by spaces at runtime) separate the words

If you need more control over the class name, you can override the default convention by implementing the
[`JavaMigration`](/documentation/api/javadoc/org/flywaydb/core/api/migration/JavaMigration) interface directly.

This will allow you to name your class as you wish. Version, description and migration category are provided by
implementing the respective methods.

### Discovery

Flyway discovers Java-based migrations on the Java classpath in the packages referenced by the
[`locations`](/documentation/commandline/migrate#locations) property.

<pre class="filetree"><i class="fa fa-folder-open"></i> my-project
  <i class="fa fa-folder-open"></i> src
    <i class="fa fa-folder-open"></i> main
      <i class="fa fa-folder-open"></i> java
        <span><i class="fa fa-folder-open"></i> db
  <i class="fa fa-folder-open"></i> migration</span>            <i class="fa fa-long-arrow-left"></i> <code>classpath:db/migration</code> 
            <i class="fa fa-file-text"></i> R__My_view
            <i class="fa fa-file-text"></i> U1_1__Fix_indexes
            <i class="fa fa-file-text"></i> V1__Initial_version
            <i class="fa fa-file-text"></i> V1_1__Fix_indexes
  <i class="fa fa-file-text"></i> pom.xml</pre>

New java migrations are **discovered automatically** through classpath scanning at runtime. The
scanning is recursive. Java migrations in subpackages of the specified ones are also picked up.

### Checksums and Validation

Unlike SQL migrations, Java migrations by default do not have a checksum and therefore do not participate in the
change detection of Flyway's validation. This can be remedied by implementing the 
`getChecksum()` method, which you can then use to provide your own checksum, which will then be
stored and validated for changes.

### Sample Class
```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.PreparedStatement;

/**
 * Example of a Java-based migration.
 */
public class V1_2__Another_user extends BaseJavaMigration {
    public void migrate(Context context) throws Exception {
        try (PreparedStatement statement = 
                 context
                     .getConnection()
                     .prepareStatement("INSERT INTO test_user (name) VALUES ('Obelix')")) {
            statement.execute();
        }
    }
}
```

### Spring

If your application already uses Spring and you do not want to use JDBC directly you can easily use Spring JDBC's
`JdbcTemplate` instead:

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.PreparedStatement;

/**
 * Example of a Java-based migration using Spring JDBC.
 */
public class V1_2__Another_user extends BaseJavaMigration {
    public void migrate(Context context) {
        new JdbcTemplate(new SingleConnectionDataSource(context.getConnection(), true))
                .execute("INSERT INTO test_user (name) VALUES ('Obelix')");
    }
}
```

## Transactions

By default, Flyway always wraps the execution of an entire migration within a single transaction.

Alternatively you can also configure Flyway to wrap the entire execution of all migrations of a single migration run
within a single transaction by setting the [`group`](/documentation/commandline/migrate#group) property to `true`.

If Flyway detects that a specific statement cannot be run within a transaction due to technical limitations of your
database, it won't run that migration within a transaction. Instead it will be marked as *non-transactional*.

By default transactional and non-transactional statements cannot be mixed within a migration run. You can however allow
this by setting the [`mixed`](/documentation/commandline/migrate#mixed) property to `true`. Note that this is only
applicable for PostgreSQL, Aurora PostgreSQL, SQL Server and SQLite which all have statements that do not run at all
within a transaction. This is not to be confused with implicit transaction, as they occur in MySQL or Oracle, where even
though a DDL statement was run within within a transaction, the database will issue an implicit commit before and after
its execution.

### Important Note

If your database cleanly supports DDL statements within a transaction, failed migrations will always be rolled back
(unless they were marked as non-transactional).

If on the other hand your database does NOT cleanly supports DDL statements within a transaction (by for example
issuing an implicit commit before and after every DDL statement), Flyway won't be able to perform a clean rollback in
case of failure and will instead mark the migration as failed, indicating that some manual cleanup may be required. 

## Query Results
{% include pro.html %}

Migrations are primarily meant to be executed as part of release and deployment automation processes and there is rarely
the need to visually inspect the result of SQL queries.

There are however some scenarios where such manual inspection makes sense, and therefore Flyway Pro and Enterprise
Edition also display query results in the usual tabular form when a `SELECT` statement (or any other statement that
returns results) is executed. 

## Schema History Table

To keep track of which migrations have already been applied when and by whom, Flyway adds a special
**schema history table** to your schema. You can think of this table as a complete audit trail of all changes
performed against the schema. It also tracks migration checksums and whether or not the migrations were successful.

Read more about this in our getting started guide on [how Flyway works](/getstarted/how).

## Migration States

Migrations are either *resolved* or *applied*. Resolved migrations have been detected by Flyway's filesystem
and classpath scanner. Initially they are **pending**. Once they are executed against the database,
they become applied.

When the migration *succeeds* it is marked as **success** in Flyway's *schema history table*.

When the migration *fails* and the database supports *DDL transactions*, the migration is *rolled back* and
nothing is recorded in the schema history table.

When the migration *fails* and the database doesn't supports *DDL transactions*, the migration
is marked as **failed** in the schema history table, indicating manual database cleanup may be required.

Versioned migrations whose effects have been undone by an undo migration are marked as **undone**.

Repeatable migrations whose checksum has changed since they are last applied are marked as **outdated** until
they are executed again.

When Flyway discovers an applied versioned migration with a version that is higher than the highest known version
(this happens typically when a newer version of the software has migrated that schema), that migration is marked as **future**.

<p class="next-steps">
    <a class="btn btn-primary" href="/documentation/callbacks">Callbacks <i class="fa fa-arrow-right"></i></a>
</p>
