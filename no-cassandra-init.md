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
#!/bin/bash

pass_changed=99
retry_num=1
is_up=0
while [ $is_up -eq 0 -a $retry_num -le 100 ]; do

   echo "exit" | cqlsh -u cassandra -p $CASSANDRA_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT  > /dev/null 2>&1
   res1=$?

   if [ $res1 -eq 0 ]; then
      echo "`date` --- cqlsh is enabled to connect."
      is_up=1
   else
      echo "`date` --- cqlsh is NOT enabled to connect yet. sleep 5"
      sleep 5
   fi
   let "retry_num++"
done

cassandra_user_exist=`echo "list users;" | cqlsh -u cassandra -p $CASSANDRA_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT | grep -c $SDC_USER`
if [ $cassandra_user_exist -eq 1 ] ; then
        echo "cassandra user $SDC_USER already exist"
else
        echo "Going to create $SDC_USER"
        echo "create user $SDC_USER with password '$SDC_PASSWORD' nosuperuser;" | cqlsh -u cassandra -p $CASSANDRA_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT
fi
~~~

Picked up existing script. `SC_PASSWORD` changed to `CASSANDRA_PASSWORD` for consistency
