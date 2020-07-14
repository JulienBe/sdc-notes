# What to use to inject previous behavior

When the container is executed for the first time, it will execute the files with extensions .sh, .cql or .cql.gz located at /docker-entrypoint-initdb.d in sort'ed order by filename. This behavior can be skipped by setting the environment variable CASSANDRA_IGNORE_INITDB_SCRIPTS to a value other than yes or true.

In order to have your custom files inside the docker image you can mount them as a volume.

```
docker run -v /path/to/init-scripts:/docker-entrypoint-initdb.d bitnami/cassandra
```

3 operations were perfomed before:
1. create the sdc cassandra user if not present
2. create the schema
