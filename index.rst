:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml


Abstract
========

A walkthrough of how data from a request gets into a backend postgres database to allow it to be compared to how QServ might have to do it.

In short, you have to do what you imagine, and what pretty much any SQL server requires.  Once the TAP server
gets the request, it needs to send a CREATE TABLE to the backend SQL server (and figure out what that
statement should look like, i.e. what columns and the types of those columns), then use INSERT to upload
the rows, and then finally call DROP TABLE.

Let's go through the call stack if you will, starting at the request coming in, and at the end
see how the CADC server constructs the CREATE TABLE and the INSERTs.

Call flow
=========

First, how the IVOA TAP 1.1 spec defines UPLOAD:

Page 19:
https://www.ivoa.net/documents/TAP/20190927/REC-TAP-1.1.pdf

The UPLOAD verb works by doing a
HTTP POST to the UWS Job URL

HTTP POST http://example.com/tap/async/42
UPLOAD=mytable,http://otherplace.com/path/votable.xml

The upload of the table must be in the votable
format.  This standardizes columns, datatypes, that
sort of thing.  The server retrieves the votable.xml
table.

I believe the table can also be uploaded inline
without a URL between the two.  Both of these paths
are merged into the UploadManager code which
implements the UWSInlineContentHandler.

The Rubin postgres code takes an input stream
which contains the votable xml file and streams
it into the async-results GCS bucket to wait
for being read when the query is executed.

https://github.com/lsst-sqre/tap-postgres/blob/master/tap/src/main/java/ca/nrc/cadc/sample/UploadManager.java

Most of the interesting things happen in
the QueryRunner:

https://github.com/lsst-sqre/tap-postgres/blob/master/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java

The QueryRunner uses a few different
connection pools to execute queries.

These are pools to QServ or the backend,
as well as connection pools to the
database containing the tap_schema data.
These pools can be found in:

https://github.com/lsst-sqre/tap-postgres/blob/master/tap/src/main/webapp/META-INF/context.xml

I will outline parts of the code
that are used in upload.

First: 
https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L143

This is the name of the connection pool
to use for uploads.  Pat has specficially
reused the same pool as the one handling
queries.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L235

This is where the name of the connection
pool is used to make a connection to the
DB.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L269

Here is where the query begins to
get executed.  Note both async and
sync queries follow this same path.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L297

Here is where the upload connection
is retrieved from the pool.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L313

Here is where the TAP_SCHEMA stored
in the database is read via a connection
to that database.  It uses a pool like
the other database queries.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L318

This is where the uploaded table
starts to come into play.

On the next two lines the table is
uploaded to the database using the 
uploadDataSource, which is a member
of the same pool as the connection
pool running non upload queries.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L325

Here is where the tap schema information
for the uploaded table is combined with
the tap schema data.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L328

Here is where the data is combined for
tap schema.  

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L340

Here is where the TAP_SCHEMA data
on uploaded tables is used and pushed
into the query.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L382

Here we take the SQL provided from
the request, and run it against
both the uploaded data and the rest
of the data in the database.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L396

Writing out results for a sync request.

https://github.com/lsst-sqre/tap-postgres/blob/808cb63798258ff3bbcd48bd3c7af38d77f37a28/tap/src/main/java/ca/nrc/cadc/sample/QueryRunner.java#L411

Writing out results for an async request.


Okay now let's look more at the UploadManager.

Here's the interface:

https://github.com/opencadc/tap/blob/8cd3b6bdb7d3ac60b8d5cea38e46c4c87331ac41/cadc-tap-server/src/main/java/ca/nrc/cadc/tap/UploadManager.java#L87

This has the interface that allows for
uploading tables.  For example, all
tables are uploaded to TAP_UPLOAD.
It takes the datasource (SQL
connection) to do the upload to the
database.

setDatabaseDataType is what negotiates
between the votable datatypes and
the backend datatypes (SQL datatypes).

Upload handles the actual uploading.


https://github.com/opencadc/tap/blob/master/cadc-tap-server/src/main/java/ca/nrc/cadc/tap/BasicUploadManager.java#L101

Here's the basicuploadmanager.
This is where the uploading
starts to happen.  Note: params
are each a table.  That's how
they are referred to in the TAP
VO spec.

https://github.com/opencadc/tap/blob/8cd3b6bdb7d3ac60b8d5cea38e46c4c87331ac41/cadc-tap-server/src/main/java/ca/nrc/cadc/tap/BasicUploadManager.java#L316

Here is where the name of the uploaded
table is created.

https://github.com/opencadc/tap/blob/master/cadc-tap-schema/src/main/java/ca/nrc/cadc/tap/db/TableCreator.java#L87

Here's where tables are created and dropped.

https://github.com/opencadc/tap/blob/8cd3b6bdb7d3ac60b8d5cea38e46c4c87331ac41/cadc-tap-schema/src/main/java/ca/nrc/cadc/tap/db/TableCreator.java#L290

Here's where the string building to create
table is.

https://github.com/opencadc/tap/blob/8cd3b6bdb7d3ac60b8d5cea38e46c4c87331ac41/cadc-tap-schema/src/main/java/ca/nrc/cadc/tap/db/TableCreator.java#L310

Create a unique index on the table.

https://github.com/opencadc/tap/blob/8cd3b6bdb7d3ac60b8d5cea38e46c4c87331ac41/cadc-tap-schema/src/main/java/ca/nrc/cadc/tap/db/TableCreator.java#L187

Here's the DROP table.

https://github.com/opencadc/tap/blob/master/cadc-tap/src/main/java/ca/nrc/cadc/tap/schema/ColumnDesc.java#L100

Here's where each ColumnDescription,
including most importantly the name
and datatype are stored.

Then after the table is created with the right
columns, use the TableLoader to load the data.

https://github.com/opencadc/tap/blob/master/cadc-tap-schema/src/main/java/ca/nrc/cadc/tap/db/TableLoader.java

Here's where it's invoked after the TableCreator:

https://github.com/opencadc/tap/blob/8cd3b6bdb7d3ac60b8d5cea38e46c4c87331ac41/cadc-tap-server/src/main/java/ca/nrc/cadc/tap/BasicUploadManager.java#L235

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
