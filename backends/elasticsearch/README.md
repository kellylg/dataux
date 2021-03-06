
MySQL access to Elasticsearch
--------------------------------
[Elasticsearch](https://www.elastic.co/) is awesome, but sometimes querying it is not, and integrating it into other apps involves custom code.  
It would be great to have mysql and all the tools that talk MySql to be able to access it.

## Example Usage
This example imports a couple hours worth of historical data
from  https://www.githubarchive.org/ into a local elasticsearch server for example querying.
It only imports `github_issues` and `github_watch`.

```sh
# from root of this repo, there is a docker-compose
# which has an elasticsearch server in it we will use for example.
cd github.com/dataux/dataux
docker-compose up

# assuming elasticsearch on localhost else --host=myeshost
# lets import the data into elasticsearch
cd backends/elasticsearch/importgithub
go build && ./importgithub

# using docker start a dataux server
docker run --rm -it --net=host -p 4000:4000 gcr.io/dataux-io/dataux:latest

# now that dataux is running use mysql-client to connect
mysql -h 127.0.0.1 -P 4000
```
now run some queries
```sql
show databases;

-- first register the schema
-- dataux will introspect the tables
-- to create schema for the tables
CREATE schema github_archive IF NOT EXISTS WITH {
  "type":"elasticsearch",
  "schema":"github_archive",
  "hosts": ["http://127.0.0.1:9200"]
};

-- now a sql tour
show databases;
use github_archive;

show tables;

describe github_watch;

select actor, repostory.url from github_watch limit 10;

select cardinality(`actor`) AS users_who_watched, min(`repository.id`) as oldest_repo from github_watch;

SELECT actor, `repository.name`, `repository.stargazers_count`, `repository.language`
FROM github_watch where `repository.language` = "Go";

select actor, repository.name from github_watch where repository.stargazers_count BETWEEN "1000" AND 1100;

SELECT actor, repository.organization AS org
FROM github_watch 
WHERE repository.created_at BETWEEN "2008-10-21T17:20:37Z" AND "2008-10-21T19:20:37Z";

select actor, repository.name from github_watch where repository.name IN ("node", "docker","d3","myicons", "bootstrap") limit 100;

select cardinality(`actor`) AS users_who_watched, count(*) as ct, min(`repository.id`) as oldest_repo
FROM github_watch
WHERE repository.description LIKE "database";


```

SQL -> Elasticsearch
----------------------------------

ES API | SQL Query  
----- | -------
Aliases                 | `show tables;`
Mapping                 | `describe mytable;`
hits.total  for filter  | `select count(*) from table WHERE exists(a);`
aggs min, max, avg, sum | `select min(year), max(year), avg(year), sum(year) from table WHERE exists(a);`
filter:   terms         | `select * from table WHERE year IN (2015,2014,2013);`
filter: gte, range      | `select * from table WHERE year BETWEEN 2012 AND 2014`


Configuration
-----------------------------
* *tables_to_load* If you do not want to load all tables.

```
CREATE schema github_archive IF NOT EXISTS WITH {
  "type":"elasticsearch",
  "schema":"github_archive",
  "tables_to_load":["indexa","indexb"],
  "hosts": ["http://127.0.0.1:9200"]
};

```