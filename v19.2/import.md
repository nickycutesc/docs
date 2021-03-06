---
title: IMPORT
summary: Import data into your CockroachDB cluster.
toc: true
---

The `IMPORT` [statement](sql-statements.html) imports the following types of data into CockroachDB:

- [CSV/TSV][csv]
- [Postgres dump files][postgres]
- [MySQL dump files][mysql]
- [CockroachDB dump files](sql-dump.html)

{{site.data.alerts.callout_success}}
`IMPORT` only works for creating new tables. For information on how to import into existing tables, see [`IMPORT INTO`](import-into.html). Also, for instructions and working examples on how to migrate data from other databases, see the [Migration Overview](migration-overview.html).
{{site.data.alerts.end}}

{{site.data.alerts.callout_danger}}
`IMPORT` cannot be used within a [transaction](transactions.html) or during a [rolling upgrade](upgrade-cockroach-version.html).
{{site.data.alerts.end}}

## Required privileges

Only members of the `admin` role can run `IMPORT`. By default, the `root` user belongs to the `admin` role.

## Synopsis

**Import a table from CSV**

<div>
  {% include {{ page.version.version }}/sql/diagrams/import_csv.html %}
</div>

**Import a database or table from dump file**

<div>
  {% include {{ page.version.version }}/sql/diagrams/import_dump.html %}
</div>

## Parameters

### For import from CSV

Parameter | Description
----------|------------
`table_name` | The name of the table you want to import/create.
`table_elem_list` | The table schema you want to use.  
`CREATE USING file_location` | If not specifying the table schema inline via `table_elem_list`, this is the [URL](#import-file-urls) of a CSV file containing the table schema.
`file_location` | The [URL](#import-file-urls) of a CSV file containing the table data. This can be a comma-separated list of URLs to CSV files. For an example, see [Import a table from multiple CSV files](#import-a-table-from-multiple-csv-files) below.
`WITH kv_option_list` | Control your import's behavior with [these options](#import-options).

### For import from dump file

Parameter | Description
----------|------------
`table_name` | The name of the table you want to import/create. Use this when the dump file contains a specific table. Leave out `TABLE table_name FROM` when the dump file contains an entire database.
`import_format` | [PGDUMP](#import-a-postgres-database-dump) or [MYSQLDUMP](#import-a-mysql-database-dump)
`file_location` | The [URL](#import-file-urls) of a dump file you want to import.
`WITH kv_option_list` | Control your import's behavior with [these options](#import-options).

### Import file URLs

URLs for the files you want to import must use the format shown below.  For examples, see [Example file URLs](#example-file-urls).

{% include {{ page.version.version }}/misc/external-urls.md %}

### Import options

You can control the `IMPORT` process's behavior using any of the following key-value pairs as a `kv_option`.

<a name="delimiter"></a>

| Key                 | Context         | Value                                                                                                                                                                                       | Required? | Example                                                                                           |
|---------------------+-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------|
| `delimiter`         | CSV             | The unicode character that delimits columns in your rows. **Default: `,`**.                                                                                                                 | No        | To use tab-delimited values: `IMPORT TABLE foo (..) CSV DATA ('file.csv') WITH delimiter = e'\t'` |
| `comment`           | CSV             | The unicode character that identifies rows to skip.                                                                                                                                         | No        | `IMPORT TABLE foo (..) CSV DATA ('file.csv') WITH comment = '#'`                                  |
| `nullif`            | CSV             | The string that should be converted to *NULL*.                                                                                                                                              | No        | To use empty columns as *NULL*: `IMPORT TABLE foo (..) CSV DATA ('file.csv') WITH nullif = ''`    |
| `skip`              | CSV             | The number of rows to be skipped while importing a file. **Default: `'0'`**.                                                                                                                | No        | To import CSV files with column headers: `IMPORT ... CSV DATA ('file.csv') WITH skip = '1'`       |
| `decompress`        | General         | The decompression codec to be used: `gzip`, `bzip`, `auto`, or `none`.  **Default: `'auto'`**, which guesses based on file extension (`.gz`, `.bz`, `.bz2`). `none` disables decompression. | No        | `IMPORT ... WITH decompress = 'bzip'`                                                             |
| `skip_foreign_keys` | Postgres, MySQL | Ignore foreign key constraints in the dump file's DDL. **Off by default**.  May be necessary to import a table with unsatisfied foreign key constraints from a full database dump.          | No        | `IMPORT TABLE foo FROM MYSQLDUMP 'dump.sql' WITH skip_foreign_keys`                               |
| `max_row_size`      | Postgres        | Override limit on line size. **Default: 0.5MB**.  This setting may need to be tweaked if your Postgres dump file has extremely long lines, for example as part of a `COPY` statement.       | No        | `IMPORT PGDUMP DATA ... WITH max_row_size = '5MB'`                                                |

For examples showing how to use these options, see the [Examples](#examples) section below.

For instructions and working examples showing how to migrate data from other databases and formats, see the [Migration Overview](migration-overview.html).

## Requirements

### Prerequisites

Before using `IMPORT`, you should have:

- The schema of the table you want to import.
- The data you want to import, preferably hosted on cloud storage. This location must be equally accessible to all nodes using the same import file location.  This is necessary because the `IMPORT` statement is issued once by the client, but is executed concurrently across all nodes of the cluster.  For more information, see the [Import file location](#import-file-location) section below.

### Import targets

Imported tables must not exist and must be created in the `IMPORT` statement. If the table you want to import already exists, you must drop it with [`DROP TABLE`](drop-table.html) or use [`IMPORT INTO`](import-into.html).

You can specify the target database in the table name in the `IMPORT` statement. If it's not specified there, the active database in the SQL session is used.

### Create table

Your `IMPORT` statement must reference a `CREATE TABLE` statement representing the schema of the data you want to import.  You have several options:

- Specify the table's columns explicitly from the [SQL client](use-the-built-in-sql-client.html). For an example, see [Import a table from a CSV file](#import-a-table-from-a-csv-file) below.

- Load a file that already contains a `CREATE TABLE` statement. For an example, see [Import a Postgres database dump](#import-a-postgres-database-dump) below.

We also recommend [specifying all secondary indexes you want to use in the `CREATE TABLE` statement](create-table.html#create-a-table-with-secondary-and-inverted-indexes). It is possible to [add secondary indexes later](create-index.html), but it is significantly faster to specify them during import.

{{site.data.alerts.callout_info}}
By default, the [Postgres][postgres] and [MySQL][mysql] import formats support foreign keys. However, the most common dependency issues during import are caused by unsatisfied foreign key relationships that cause errors like `pq: there is no unique constraint matching given keys for referenced table tablename`. You can avoid these issues by adding the [`skip_foreign_keys`](#import-options) option to your `IMPORT` statement as needed. Ignoring foreign constraints will also speed up data import.
{{site.data.alerts.end}}

### Available storage

Each node in the cluster is assigned an equal part of the imported data, and so must have enough temp space to store it. In addition, data is persisted as a normal table, and so there must also be enough space to hold the final, replicated data. The node's first-listed/default [`store`](start-a-node.html#store) directory must have enough available storage to hold its portion of the data.

On [`cockroach start`](start-a-node.html), if you set `--max-disk-temp-storage`, it must also be greater than the portion of the data a node will store in temp space.

### Import file location

We strongly recommend using cloud/remote storage (Amazon S3, Google Cloud Platform, etc.) for the data you want to import.

Local files are supported; however, they must be accessible to all nodes in the cluster using identical [Import file URLs](#import-file-urls).

To import a local file, you have the following options:

- Option 1. Run a [local file server](create-a-file-server.html) to make the file accessible from all nodes.

- Option 2. Make the file accessible from each local node's store:
    1. Create an `extern` directory on each node's store. The pathname will differ depending on the [`--store` flag passed to `cockroach start` (if any)](start-a-node.html#general), but will look something like `/path/to/cockroach-data/extern/`.
    2. Copy the file to each node's `extern` directory.
    3. Assuming the file is called `data.sql`, you can access it in your `IMPORT` statement using the following [import file URL](#import-file-urls): `'nodelocal:///data.sql'`.

### Table users and privileges

Imported tables are treated as new tables, so you must [`GRANT`](grant.html) privileges to them.

## Performance

- All nodes are used during the import job, which means all nodes' CPU and RAM will be partially consumed by the `IMPORT` task in addition to serving normal traffic.
- To improve performance, import at least as many files as you have nodes (i.e., there is at least one file for each node to import) to increase parallelism.
- To further improve performance, order the data in the imported files by [primary key](primary-key.html) and ensure the primary keys do not overlap between files.

## Viewing and controlling import jobs

After CockroachDB successfully initiates an import, it registers the import as a job, which you can view with [`SHOW JOBS`](show-jobs.html).

After the import has been initiated, you can control it with [`PAUSE JOB`](pause-job.html), [`RESUME JOB`](resume-job.html), and [`CANCEL JOB`](cancel-job.html).

{{site.data.alerts.callout_danger}}Pausing and then resuming an <code>IMPORT</code> job will cause it to restart from the beginning.{{site.data.alerts.end}}

## Examples

### Import a table from a CSV file

To manually specify the table schema:

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('s3://acme-co/customers.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]&AWS_SESSION_TOKEN=[placeholder]')
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('gs://acme-co/customers.csv')
;
~~~

To use a file to specify the table schema:

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers
CREATE USING 's3://acme-co/customers-create-table.sql?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]'
CSV DATA ('s3://acme-co/customers.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]')
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers
CREATE USING 'azure://acme-co/customer-create-table.sql?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers
CREATE USING 'gs://acme-co/customers-create-table.sql'
CSV DATA ('gs://acme-co/customers.csv')
;
~~~

### Import a table from multiple CSV files

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA (
    's3://acme-co/customers.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]',
    's3://acme-co/customers2.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder',
    's3://acme-co/customers3.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]',
    's3://acme-co/customers4.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]',
);
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA (
    'azure://acme-co/customer-import-data1.1.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co',
    'azure://acme-co/customer-import-data1.2.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co',
    'azure://acme-co/customer-import-data1.3.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co',
    'azure://acme-co/customer-import-data1.4.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co',
    'azure://acme-co/customer-import-data1.5.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co',    
);
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA (
    'gs://acme-co/customers.csv',
    'gs://acme-co/customers2.csv',
    'gs://acme-co/customers3.csv',
    'gs://acme-co/customers4.csv',
);
~~~

### Import a table from a TSV file

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('s3://acme-co/customers.tsv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]')
WITH
	delimiter = e'\t'
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.tsv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	delimiter = e'\t'
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('gs://acme-co/customers.tsv')
WITH
	delimiter = e'\t'
;
~~~

### Skip commented lines

The `comment` option determines which Unicode character marks the rows in the data to be skipped.

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('s3://acme-co/customers.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]')
WITH
	comment = '#'
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	comment = '#'
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('gs://acme-co/customers.csv')
WITH
	comment = '#'
;
~~~

### Skip first *n* lines

The `skip` option determines the number of header rows to skip when importing a file.

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('s3://acme-co/customers.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]')
WITH
	skip = '2'
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	skip = '2'
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('gs://acme-co/customers.csv')
WITH
	skip = '2'
;
~~~

### Use blank characters as `NULL`

The `nullif` option defines which string should be converted to `NULL`.

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('s3://acme-co/employees.csv?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]')
WITH
	nullif = ''
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	nullif = ''
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('gs://acme-co/customers.csv')
WITH
	nullif = ''
;
~~~

### Import a compressed CSV file

CockroachDB chooses the decompression codec based on the filename (the common extensions `.gz` or `.bz2` and `.bz`) and uses the codec to decompress the file during import.

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('s3://acme-co/employees.csv.gz?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]')
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv.gz?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('gs://acme-co/customers.csv.gz')
;
~~~

Optionally, you can use the `decompress` option to specify the codec to be used for decompressing the file during import:

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('s3://acme-co/employees.csv.gz?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]')
WITH
	decompress = 'gzip'
;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv.gz.latest?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	decompress = 'gzip'
;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('gs://acme-co/customers.csv.gz')
WITH
	decompress = 'gzip'
;
~~~

### Import a Postgres database dump

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT PGDUMP 's3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]';
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT PGDUMP 'azure://acme-co/employees.sql?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co';
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT PGDUMP 'gs://acme-co/employees.sql';
~~~

For the commands above to succeed, you need to have created the dump file with specific flags to `pg_dump`. For more information, see [Migrate from Postgres][postgres].

### Import a table from a Postgres database dump

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees FROM PGDUMP 's3://your-external-storage/employees-full.sql?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]' WITH skip_foreign_keys;
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees FROM PGDUMP 'azure://acme-co/employees.sql?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co' WITH skip_foreign_keys;
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees FROM PGDUMP 'gs://acme-co/employees.sql' WITH skip_foreign_keys;
~~~

If the table schema specifies foreign keys into tables that do not exist yet, the `WITH skip_foreign_keys` shown may be needed. For more information, see the list of [import options](#import-options).

For the command above to succeed, you need to have created the dump file with specific flags to `pg_dump`.  For more information, see [Migrate from Postgres][postgres].

### Import a CockroachDB dump file

Cockroach dump files can be imported using the `IMPORT PGDUMP`.

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT PGDUMP 's3://your-external-storage/employees-full.sql?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]';
~~~

Azure:
{% include copy-clipboard.html %}
~~~ sql
> IMPORT PGDUMP 'azure://acme-co/employees.sql?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co';
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT PGDUMP 'gs://acme-co/employees.sql';
~~~

For more information, see [SQL Dump (Export)](sql-dump.html).

### Import a MySQL database dump

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT MYSQLDUMP 's3://your-external-storage/employees-full.sql?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]';
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT MYSQLDUMP 'azure://acme-co/employees.sql?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co';
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT MYSQLDUMP 'gs://acme-co/employees.sql';
~~~

For more detailed information about importing data from MySQL, see [Migrate from MySQL][mysql].

### Import a table from a MySQL database dump

Amazon S3:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees FROM MYSQLDUMP 's3://your-external-storage/employees-full.sql?AWS_ACCESS_KEY_ID=[placeholder]&AWS_SECRET_ACCESS_KEY=[placeholder]' WITH skip_foreign_keys
~~~

Azure:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees FROM MYSQLDUMP 'azure://acme-co/employees.sql?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co' WITH skip_foreign_keys
~~~

Google Cloud:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees FROM MYSQLDUMP 'gs://acme-co/employees.sql' WITH skip_foreign_keys
~~~

If the table schema specifies foreign keys into tables that do not exist yet, the `WITH skip_foreign_keys` shown may be needed.  For more information, see the list of [import options](#import-options).

For more detailed information about importing data from MySQL, see [Migrate from MySQL][mysql].

## Known limitation

{% include {{ page.version.version }}/known-limitations/import-high-disk-contention.md %}

## See also

- [Create a File Server](create-a-file-server.html)
- [Migration Overview](migration-overview.html)
- [Migrate from MySQL][mysql]
- [Migrate from Postgres][postgres]
- [Migrate from CSV][csv]
- [`IMPORT INTO`](import-into.html)

<!-- Reference Links -->

[postgres]: migrate-from-postgres.html
[mysql]: migrate-from-mysql.html
[csv]: migrate-from-csv.html
