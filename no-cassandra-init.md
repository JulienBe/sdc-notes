# What to use to inject previous behavior

When the container is executed for the first time, it will execute the files with extensions .sh, .cql or .cql.gz located at /docker-entrypoint-initdb.d in sort'ed order by filename. This behavior can be skipped by setting the environment variable CASSANDRA_IGNORE_INITDB_SCRIPTS to a value other than yes or true.

In order to have your custom files inside the docker image you can mount them as a volume.

```
docker run -v /path/to/init-scripts:/docker-entrypoint-initdb.d bitnami/cassandra
```

3 operations were perfomed before:
1. create the sdc cassandra user if not present
2. create the schema
3. ?

# 1. Create the user if not present

Env variables :

- $SDC_PASSWORD
- $SDC_USER
- $CASSANDRA_PASSWORD
- $CASSANDRA_IP
- $CASSANDRA_PORT

~~~bash
# Load libraries
. /opt/bitnami/scripts/libbitnami.sh
. /opt/bitnami/scripts/libcassandra.sh
. /opt/bitnami/scripts/libos.sh

# wait for cassandra to be available
wait_for_nodetool_up "100" "5"
cqlsh -u cassandra -p $CASSANDRA_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT -e "CREATE USER IF NOT EXISTS  $SDC_USER WITH PASSWORD '$SDC_PASSWORD' NOSUPERUSER;"
~~~

~Picked up existing script~. Simplified the script, and there was a race condition so had to account for cassandra being accessible but not ready. Using built in wait_for_nodetool_up
`SC_PASSWORD` changed to `CASSANDRA_PASSWORD` for consistency

