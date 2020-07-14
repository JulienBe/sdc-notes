unwrapping this

~~~Dockerfile
FROM onap/base_sdc-cqlsh:1.6.1

COPY --chown=sdc:sdc chef-solo /home/sdc/chef-solo/

COPY --chown=sdc:sdc chef-repo/cookbooks /home/sdc/chef-solo/cookbooks/

COPY --chown=sdc:sdc startup.sh /home/sdc/

RUN chmod 770 /home/sdc/startup.sh

ENTRYPOINT [ "/home/sdc/startup.sh" ]
~~~

# unpacks to 

~~~Dockerfile
FROM openjdk:8-jdk-alpine

RUN addgroup -g 1000 sdc && adduser -S -u 1000 -G sdc -s /bin/sh sdc
USER sdc
RUN mkdir ~/.cassandra/ && \
    echo  '[cql]' > ~/.cassandra/cqlshrc  && \
    echo  'version=3.4.4' >> ~/.cassandra/cqlshrc 
USER root

RUN apk add --no-cache py-pip && \
    pip install cqlsh==5.0.4 && \
    mkdir ~/.cassandra/ && \
    echo  '[cql]' > ~/.cassandra/cqlshrc  && \
    echo  'version=3.4.4' >> ~/.cassandra/cqlshrc  && \
    set -ex && \
    pip install cqlsh && \
    apk add --no-cache \
    bash \
    build-base \
    ruby=2.5.8-r0 \
    ruby-dev \
    libffi-dev \
    libxml2-dev && \
    gem install chef:13.8.5 berkshelf:6.3.1 io-console:0.4.6 etc webrick --no-document && \
    apk update && \
    apk add binutils \
    libtasn1
USER sdc


COPY --chown=sdc:sdc chef-solo /home/sdc/chef-solo/

COPY --chown=sdc:sdc chef-repo/cookbooks /home/sdc/chef-solo/cookbooks/

COPY --chown=sdc:sdc startup.sh /home/sdc/

RUN chmod 770 /home/sdc/startup.sh

ENTRYPOINT [ "/home/sdc/startup.sh" ]
~~~

# Overview 

startup: just run `chef`

run list
~~~
    "recipe[cassandra-actions::01-createCsUser]",
    "recipe[cassandra-actions::03-schemaCreation]",
    "recipe[cassandra-actions::04-importConformance]"
~~~

createCsUser

~~~ruby
template "/tmp/create_cassandra_user.sh" do
  source "create_cassandra_user.sh.erb"
  sensitive true
  mode 0755
  variables({
     :cassandra_ip      => node['Nodes']['CS'].first,
     :cassandra_port    => node['cassandra']['cassandra_port'],
     :cassandra_pwd     => ENV['CS_PASSWORD'],
     :sdc_usr           => ENV['SDC_USER'],
     :sdc_pwd           => ENV['SDC_PASSWORD']
  })
end


bash "create-sdc-user" do
   code <<-EOH
     cd /tmp ; /tmp/create_cassandra_user.sh
   EOH
end
~~~

schemaCreation
~~~ruby
cookbook_file "/tmp/sdctool.tar" do
  source "sdctool.tar"
  mode 0755
end

## extract sdctool.tar
bash "install tar" do
  cwd "/tmp"
  code <<-EOH
     /bin/tar xf /tmp/sdctool.tar -C /tmp
  EOH
end


template "janusgraph.properties" do
  sensitive true
  path "/tmp/sdctool/config/janusgraph.properties"
  source "janusgraph.properties.erb"
  mode "0755"
  variables({
     :DC_NAME                       => node['cassandra']['datacenter_name'],
     :cassandra_ip                  => node['Nodes']['CS'].first,
     :cassandra_port_num            => node['cassandra'][:cassandra_port],
     :janus_connection_timeout      => node['cassandra'][:janusgraph_connection_timeout],
     :cassandra_pwd                 => node['cassandra'][:cassandra_password],
     :cassandra_usr                 => node['cassandra'][:cassandra_user],
     :replication_factor            => node['cassandra']['replication_factor']
  })
end


template "/tmp/sdctool/config/configuration.yaml" do
  sensitive true
  source "configuration.yaml.erb"
  mode 0755
  variables({
      :host_ip                => node['Nodes']['BE'],
      :catalog_port           => node['BE'][:http_port],
      :ssl_port               => node['BE'][:https_port],
      :cassandra_ip           => node['Nodes']['CS'].first,
      :cassandra_port         => node['cassandra']['cassandra_port'],
      :rep_factor             => node['cassandra']['replication_factor'],
      :DC_NAME                => node['cassandra']['datacenter_name'],
      :janusgraph_Path        => "/tmp/sdctool/config/",
      :socket_connect_timeout => node['cassandra']['socket_connect_timeout'],
      :socket_read_timeout    => node['cassandra']['socket_read_timeout'],
      :cassandra_pwd          => node['cassandra'][:cassandra_password],
      :cassandra_usr          => node['cassandra'][:cassandra_user]
  })
end



bash "executing-schema-creation" do
   code <<-EOH
     cd /tmp
     chmod +x /tmp/sdctool/scripts/schemaCreation.sh
     /tmp/sdctool/scripts/schemaCreation.sh /tmp/sdctool/config
   EOH
end

bash "executing-janusGraphSchemaCreation.sh" do
  code <<-EOH
     chmod +x /tmp/sdctool/scripts/janusGraphSchemaCreation.sh
     /tmp/sdctool/scripts/janusGraphSchemaCreation.sh /tmp/sdctool/config
   EOH
end
~~~

importConformance
~~~ruby
working_directory =  "/tmp"
cl_release=node['version'].split('.')[0..2].join('.').split('-')[0]
printf("\033[33mcl_release=[%s]\n\033[0m", cl_release)



bash "import-Conformance" do
  cwd "#{working_directory}"
  code <<-EOH
    conf_dir=/tmp/sdctool/config
    tosca_dir=/tmp/sdctool/tosca

    cl_version=`grep 'toscaConformanceLevel:' $conf_dir/configuration.yaml |awk '{print $2}'`

    cd /tmp/sdctool/scripts
    /bin/chmod +x sdcSchemaFileImport.sh
    echo "execute /tmp/sdctool/scripts/sdcSchemaFileImport.sh ${tosca_dir} #{cl_release} ${cl_version} ${conf_dir} onap"
    ./sdcSchemaFileImport.sh ${tosca_dir} #{cl_release} ${cl_version} ${conf_dir} onap
  EOH
end
~~~

# 1 createCsUser

~~~ruby
template "/tmp/create_cassandra_user.sh" do
  source "create_cassandra_user.sh.erb"
  sensitive true
  mode 0755
  variables({
     :cassandra_ip      => node['Nodes']['CS'].first,
     :cassandra_port    => node['cassandra']['cassandra_port'],
     :cassandra_pwd     => ENV['CS_PASSWORD'],
     :sdc_usr           => ENV['SDC_USER'],
     :sdc_pwd           => ENV['SDC_PASSWORD']
  })
end


bash "create-sdc-user" do
   code <<-EOH
     cd /tmp ; /tmp/create_cassandra_user.sh
   EOH
end
~~~

create_cassandra_user.sh.erb

~~~bash
#!/bin/bash

CASSANDRA_IP=<%= @cassandra_ip %>
CASSANDRA_PORT=<%= @cassandra_port %>
CS_PASSWORD=<%= @cassandra_pwd %>
SDC_USER=<%= @sdc_usr %>
SDC_PASSWORD=<%= @sdc_pwd %>


pass_changed=99
retry_num=1
is_up=0
while [ $is_up -eq 0 -a $retry_num -le 100 ]; do

   echo "exit" | cqlsh -u cassandra -p $CS_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT  > /dev/null 2>&1
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

cassandra_user_exist=`echo "list users;" | cqlsh -u cassandra -p $CS_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT | grep -c $SDC_USER`
        if [ $cassandra_user_exist -eq 1 ] ; then
                echo "cassandra user $SDC_USER already exist"
        else
                echo "Going to create $SDC_USER"
                echo "create user $SDC_USER with password '$SDC_PASSWORD' nosuperuser;" | cqlsh -u cassandra -p $CS_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT
        fi
~~~

Used variables:
- cassandra_ip
- cassandra_port
- cassandra_pwd
- sdc_usr
- sdc_pwd

tries to :
```
# check if it's up
cqlsh -u cassandra -p $CS_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT
# check if the $SDC_USER is present
cassandra_user_exist=`echo "list users;" | cqlsh -u cassandra -p $CS_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT | grep -c $SDC_USER`
# create user if not present
echo "create user $SDC_USER with password '$SDC_PASSWORD' nosuperuser;" | cqlsh -u cassandra -p $CS_PASSWORD $CASSANDRA_IP $CASSANDRA_PORT
```
