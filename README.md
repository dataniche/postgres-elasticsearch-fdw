PostgreSQL Elastic Search foreign data wrapper
==============================================

This allows you to index data in Elastic Search and then search it from
PostgreSQL. You can write as well as read.

SYNOPSIS
--------

### Supported Versions

| Elastic Search | Dependency Installation Command |
|----------------|---------------------------------|
| 5 | `sudo pip install "elasticsearch>=5,<6"` |
| 6 | `sudo pip install "elasticsearch>=6,<7"` |
| 7 | `sudo pip install "elasticsearch>=7,<8"` |

| PostgreSQL | Dependency Installation Command |
|------------|---------------------------------|
| 9.4 | `sudo apt-get install postgresql-9.4-python-multicorn` |
| 9.5 | `sudo apt-get install postgresql-9.5-python-multicorn` |
| 9.6 | `sudo apt-get install postgresql-9.6-python-multicorn` |
| 10 | `sudo apt-get install postgresql-10-python-multicorn` |
| 11 | `sudo apt-get install postgresql-11-python-multicorn` |

### Installation

This requires installation on the PostgreSQL server, and has system level dependencies.
You can install the dependencies with:

```
sudo apt-get install python python-pip
```

You should install the version of multicorn that is specific to your postgres
version. See the table in _Supported Versions_ for installation commands. The
multicorn package is also only available from Ubuntu Xenial (16.04) onwards. If
you cannot install multicorn in this way then you can use
[pgxn](http://pgxnclient.projects.pgfoundry.org/) to install it.

This uses the Elastic Search client which has release versions that correspond
to the major version of the Elastic Search server. You should install the
`elasticsearch` dependency separately. See the table in _Supported Versions_
for installation commands.

Once the dependencies are installed you can install the foreign data wrapper
using pip:

```
sudo pip install pg_es_fdw
```

### Usage

A running configuration for this can be found in the `docker-compose.yml`
within this folder.

The basic steps are:

 * Load the extension
 * Create the server
 * Create the foreign table
 * Populate the foreign table
 * Query the foreign table...

#### Load extension and Create server

```sql
CREATE EXTENSION multicorn;

CREATE SERVER multicorn_es FOREIGN DATA WRAPPER multicorn
OPTIONS (
  wrapper 'pg_es_fdw.ElasticsearchFDW'
);
```

#### Create the foreign table

```sql
CREATE FOREIGN TABLE articles_es
    (
        id BIGINT,
        title TEXT,
        body TEXT,
        query TEXT,
        score NUMERIC
    )
SERVER multicorn_es
OPTIONS
    (
        host 'elasticsearch',
        port '9200',
        index 'article-index',
        type 'article',
        rowid_column 'id',
        query_column 'query',
        score_column 'score'
    )
;
```

Elastic Search 7 and greater does not require the `type` option, which
corresponds to the `doc_type` used in prior versions of Elastic Search.

This corresponds to an Elastic Search index which contains a `title` and `body`
fields. The other fields have special meaning:

 * The `id` field is mapped to the Elastic Search document id
 * The `query` field accepts Elastic Search queries to filter the rows
 * The `score` field returns the score for the document against the query

These are configured using the `rowid_column`, `query_column` and
`score_column` options. All of these are optional.

#### Populate the foreign table

```sql
INSERT INTO articles_es
    (
        id,
        title,
        body
    )
VALUES
    (
        1,
        'foo',
        'spike'
    );
```

It is possible to write documents to Elastic Search using the foreign data
wrapper. This feature was introduced in PostgreSQL 9.3.

#### Query the foreign table

To select all documents:

```sql
SELECT
    id,
    title,
    body
FROM
    articles_es
;
```

To filter the documents using a query:

```sql
SELECT
    id,
    title,
    body,
    score
FROM
    articles_es
WHERE
    query = 'body:chess'
;
```

This uses the [URI Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-uri-request.html) from Elastic Search.

Caveats
-------

Elastic Search does not support transactions, so the elasticsearch index
is not guaranteed to be synchronized with the canonical version in PostgreSQL.
Unfortunately this is the case even for serializable isolation level transactions.
It would however be possible to check against Elastic Search version field and locking.

Rollback is currently not supported.

Tests
-----

There are end to end tests that use docker to create a PostgreSQL and Elastic
Search database. These are then populated with data and tests are run against
them.

These require docker and docker-compose. These also require python packages
which you can install with:

```bash
pip install -r tests/requirements.txt
```

You can then run the tests using `tests/run.py`.  This takes the PostgreSQL
version(s) to test using the `--pg` argument and the Elastic Search versions to
test with the `--es` argument.  The currently supported versions of PostgreSQL
are 9.4 through to 11. The currently supported versions of Elastic Search are 5
and 6. You can pass multiple versions to test it against all of them:

```bash
➜ pipenv run ./tests/run.py --pg 9.4 9.5 9.6 10 11 --es 5 6 7
Testing PostgreSQL 9.4 with Elasticsearch 5
PostgreSQL 9.4 with Elasticsearch 5: Test read - PASS
PostgreSQL 9.4 with Elasticsearch 5: Test query - PASS
Testing PostgreSQL 9.4 with Elasticsearch 6
PostgreSQL 9.4 with Elasticsearch 6: Test read - PASS
PostgreSQL 9.4 with Elasticsearch 6: Test query - PASS
Testing PostgreSQL 9.4 with Elasticsearch 7
PostgreSQL 9.4 with Elasticsearch 7: Test read - PASS
PostgreSQL 9.4 with Elasticsearch 7: Test query - PASS
Testing PostgreSQL 9.5 with Elasticsearch 5
PostgreSQL 9.5 with Elasticsearch 5: Test read - PASS
PostgreSQL 9.5 with Elasticsearch 5: Test query - PASS
Testing PostgreSQL 9.5 with Elasticsearch 6
PostgreSQL 9.5 with Elasticsearch 6: Test read - PASS
PostgreSQL 9.5 with Elasticsearch 6: Test query - PASS
Testing PostgreSQL 9.5 with Elasticsearch 7
PostgreSQL 9.5 with Elasticsearch 7: Test read - PASS
PostgreSQL 9.5 with Elasticsearch 7: Test query - PASS
Testing PostgreSQL 9.6 with Elasticsearch 5
PostgreSQL 9.6 with Elasticsearch 5: Test read - PASS
PostgreSQL 9.6 with Elasticsearch 5: Test query - PASS
Testing PostgreSQL 9.6 with Elasticsearch 6
PostgreSQL 9.6 with Elasticsearch 6: Test read - PASS
PostgreSQL 9.6 with Elasticsearch 6: Test query - PASS
Testing PostgreSQL 9.6 with Elasticsearch 7
PostgreSQL 9.6 with Elasticsearch 7: Test read - PASS
PostgreSQL 9.6 with Elasticsearch 7: Test query - PASS
Testing PostgreSQL 10 with Elasticsearch 5
PostgreSQL 10 with Elasticsearch 5: Test read - PASS
PostgreSQL 10 with Elasticsearch 5: Test query - PASS
Testing PostgreSQL 10 with Elasticsearch 6
PostgreSQL 10 with Elasticsearch 6: Test read - PASS
PostgreSQL 10 with Elasticsearch 6: Test query - PASS
Testing PostgreSQL 10 with Elasticsearch 7
PostgreSQL 10 with Elasticsearch 7: Test read - PASS
PostgreSQL 10 with Elasticsearch 7: Test query - PASS
Testing PostgreSQL 11 with Elasticsearch 5
PostgreSQL 11 with Elasticsearch 5: Test read - PASS
PostgreSQL 11 with Elasticsearch 5: Test query - PASS
Testing PostgreSQL 11 with Elasticsearch 6
PostgreSQL 11 with Elasticsearch 6: Test read - PASS
PostgreSQL 11 with Elasticsearch 6: Test query - PASS
Testing PostgreSQL 11 with Elasticsearch 7
PostgreSQL 11 with Elasticsearch 7: Test read - PASS
PostgreSQL 11 with Elasticsearch 7: Test query - PASS
PASS
```

### Test Failure Messages

```
Error starting userland proxy: listen tcp 0.0.0.0:5432: bind: address already in use
```
You are already running something that listens to 5432.
Try stopping your running postgres server:
```
sudo /etc/init.d/postgresql stop
```

```
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```
Your system does not have the appropriate limits in place to run a production ready instance of elasticsearch.
Try increasing it:
```
sudo sysctl -w vm.max_map_count=262144
```
This setting will revert after a reboot.
