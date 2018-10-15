---
layout: post
title: '"Insert Overwrite Into Table" with Amazon Athena'
---

For a long time, Amazon Athena does not support `INSERT` or CTAS (`Create Table As Select`) statements.
To be sure, the results of a query are automatically saved.
But the saved files are always in CSV format, and in obscure locations.
You want to save the results as an Athena table, or *insert* them into an existing table?
Either process the auto-saved CSV file, or process the query result in memory,
in both cases using some engine *other than* Athena, because, well, Athena can't write!
This leaves Athena as basically a read-only *query* tool for quick investigations and analytics,
which is rather crippling to the usefulness of the tool.

This situation changed three days ago.
On October 11, [Amazon Athena announced support for CTAS statements](https://aws.amazon.com/about-aws/whats-new/2018/10/athena_ctas_support/).
One can create a new table to hold the results of a query, and the new table is immediately usable
in subsequent queries.
The table can be written in columnar formats like Parquet or ORC, with compression,
and can be partitioned.

This is a **huge** step forward.

On the surface, CTAS allows us to create a new table dedicated to the results of a query.
This is not `INSERT`---we still can not use Athena queries to grow existing tables in an ETL fashion.
It is still rather limited.

It turns out this limitation is not hard to overcome.
Crucially, CTAS supports writting data out in a few formats, especially Parquet and ORC with compression,
and the resultant table can be partitioned. These capabilities are basically all we need for a "regular" table.
Another key point is that CTAS lets us specify the location of the resultant data.
(After all, Athena is not a storage engine. Its table definition and data storage are always separate things.)

With this, a strategy emerges: create a temporary table using a query's results, but put the data in a calculated
location on the file path of a partitioned "regular" table; then let the regular table take over the data,
and discard the meta data of the temporary table.

In this post, we will implement this approach. Along the way we need to create a few supporting utilities.
We will only show what we need to explain the approach, hence the functionalities may not be complete
for serious applications.


The CTAS statement
==================

The basic form of the supported CTAS statement is like this

```sql
CREATE table_name
WITH (
    external_location = 's3://some-location/',
    format = 'ORC',
    orc_compression = 'ZLIB',
    partitioned_by = ARRAY['col_name,...']
)
AS
SELECT
...
```

If `format` is 'PARQUET', the compression is specified by a `parquet_compression` option.
When `partitioned_by` is present, the partition columns must be the last ones in the list of columns
in the `SELECT` statement.
Other details can be found [here](https://docs.aws.amazon.com/athena/latest/ug/create-table-as.html#ctas-table-properties).


Utility preparations
====================

We need to detour a little bit and build a couple utilities.
The first is a class representing Athena table meta data.
The class is listed below.


```python
import pyathena


def athena_write(sql):
    conn = pyathena.connect()
    cursor = conn.cursor()
    cursor.execute(sql.rstrip(';') + ';')


class Table:
    def __init__(self, 
                 db_name: str, 
                 tb_name: str, 
                 location: str, 
                 columns: List[Tuple[str,str]], 
                 partitions: List[Tuple[str,str]]=None) -> None:
        '''
        `columns` and `partitions`: list of (col_name, col_type).
        '''
        self.db_name = db_name
        self.tb_name = tb_name
        assert location.startswith('s3://')
        self.location = location.rstrip('/') + '/'
        if columns is not None:
            self.columns = [(name, type_.upper()) for (name, type_) in columns]
        if partitions:
            self.partitions = [(name, type_.upper()) for (name, type_) in partitions]
        else:
            self.partitions = []
        self.compression = 'ZLIB'
    
        # We fix the writing format to be always ORC.

    @property
    def full_name(self):
        return self.db_name + '.' + self.tb_name

    def create(self, drop_if_exists: bool=False) -> None:
        def collapse(spec):
            return ', '.join(name + ' ' + type_ for (name, type_) in spec)

        if self.partitions:
            partitions = f"PARTITIONED BY ({collapse(self.partitions)})"
        else:
            partitions = ''
        sql = f'''
            CREATE EXTERNAL TABLE {self.db_name}.{self.tb_name}
            ({collapse(self.columns)})
            {partitions}
            STORED AS ORC
            LOCATION '{self.location}'
            TBLPROPERTIES ('orc.compress' = '{self.compression}');
        '''
        if drop_if_exists:
            self.drop()
        athena_write(sql)
        self.repair()

    def repair(self) -> None:
        athena_write(f'MSCK REPAIR TABLE {self.full_name}')

    def drop(self) -> None:
        athena_write(f'DROP TABLE IF EXISTS {self.full_name}')
```

This defines some basic functions, including creating and dropping a table.
It does not deal with CTAS yet. For that, we need some utilities to handle AWS S3 data,
in particular, *deleting* S3 objects, because we intend to implement the `INSERT OVERWRITE INTO TABLE` behavior
(note the "overwrite" part).

We create a utility class as listed below. It lacks `upload` and `download` methods
because they are not needed in this post.

```python
import boto3

# This module requires a directory `.aws/` containing credentials in the home directory.
# Or environment variables `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY`.


class S3Bucket:
    def __init__(self, bucket):
        self._bucket = boto3.resource('s3').Bucket(bucket)
        self._s3 = boto3.session.Session().client('s3')

    def ls(self, key, recursive: bool=False):
        # List object names directly or recursively named like `key*`.
        # If `key` is `abc/def/`,
        # then `abc/def/123/45` will return as `123/45`
        #
        # If `key` is `abc/def`,
        # then `abc/defgh/45` will return as `defgh/45`;
        # `abc/def/gh` will return as `/gh`.
        #
        # So if you know `key` is a `directory`, then it's a good idea to
        # include the trailing `/` in `key`.

        z = self._bucket.objects.filter(Prefix=key)

        if key.endswith('/'):
            key_len = len(key)
        else:
            key_len = key.rfind('/') + 1

        if recursive:
            return (v.key[key_len :] for v in z)  
            # this is a generator, b/c there can be many, many elements
        else:
            keys = set()
            for v in z:
                vv = v.key[key_len :]
                idx = vv.find('/')
                if idx >= 0:
                    vv = vv[: idx]
                keys.add(vv)
            return sorted(list(keys))

    def delete(self, key: str) -> None:
        self._s3.delete_object(Bucket=self._bucket.name, Key=key)

    def delete_tree(self, s3_path: str) -> int:
        '''
        Return the number of objects deleted.
        After this operation, the 'folder' `s3_path` is also gone.
        
        TODO: this is not the fastest way to do it.
        '''
        assert s3_path.endswith('/')
        n = 0
        for k in self.ls(s3_path, recursive=True):
            kk = s3_path + k
            self.delete(kk)
            n += 1
        return n
```

`insert overwrite into table`
=============================

Now we are ready to take on the core task: implement "insert overwrite into table" via CTAS.

First, we add a method to the class `Table` that deletes the data of a specified partition.

```python
class Table:
    # ...

    def partition_path(self, partition_values: List[str]):
        assert self.partitions
        assert 0 < len(partition_values) <= len(self.partitions)
        partition_names = [k for k,v in self.partitions[:len(partition_values)]]
        path = self.location + '/'.join(f'{k}={v}' for k,v in zip(partition_names, partition_values)) + '/'
        return path

    def purge_data(self, partition_values: List[str]=None) -> int:
        z = self.location[len('s3://') :]
        # After 's3://'.

        assert '/' in z
        bucket = S3Bucket(z[: z.find('/')])

        path = z[(z.find('/') + 1) :]
        # After bucket key.

        if partition_values:
            p = self.partition_path(partition_values)
            n = len(self.location) - len(path)
            path = p[n:]

        return bucket.delete_tree(path)
```

Next, we add a method to do the real thing:

```python
import random

TEMP_DB = 'tmp'  # Assume we have a temporary database called 'tmp'.

class Table:
    # ...

    def insert_overwrite_partition(self, partition_values: List[str], sql: str):
        tmp_tb = str(random.random()).replace('.', '')
        tmp_tb = f'{TEMP_DB}.{tmp_tb}'

        ppath = self.partition_parth(partition_values)
        
        if len(self.partitions) > len(partition_values):
            parts = ', '.join([f"'{k}'" for k,v in self.partitions[len(partition_values): ])
            parts = f'partitioned_by = ARRAY[{parts}],'
            # Be sure to verify that the last columns in `sql` match these partition fields.
        else:
            parts = ''

        sql = f'''
            CREATE TABLE {tmp_tb}
            WITH (
                external_location = '{ppath}',
                format = 'ORC',
                {parts}
                orc_compression = '{self.compression}'
            )
            AS
            {sql}
        '''

        athena_write(f'DROP TABLE IF EXISTS {tmp_tb}')
        self.purge_data(partition_values)
        athena_write(sql)
        self.repair()
        athena_write(f'DROP TABLE IF EXISTS {tmp_tb}')
```

That does the trick.