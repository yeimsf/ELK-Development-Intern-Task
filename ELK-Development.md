## RPM Machine

### Elasticsearch

- Installation is assumed to be online: 
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.2-x86_64.rpm;
rpm -i elasticsearch-8.4.2-x86_64.rpm;
```

- Superuser password will be presented at stdout of elasticsearch installation
- username = elastic [default], password = <Generated_SuperUser_Password>
- After The installation is done:
  - Disabling SSL [Mandatory]:
    - Modify /etc/elasticsearch/elasticsearch.yml:
      - change anyhting under xpack from true to false to disable the security features
      - uncomment the network.host and http.host and make them = 0.0.0.0 in order to allow localhost and the ip given to the node both access elasticsearch
      - uncomment the transport.host and make it localhost
- Reload the daemon and enable/start elasticsearch
        `systemctl daemon-reload; systemctl enable elasticsearch.service; systemctl start elasticsearch.service`
- After Service Done
- open the browser and query http://localhost:9200 and provide the username and password from installation
- Should all the above steps execute successfully, the response should be like this:
    ```json
    {
    "name" : "4V4L4NCH3",
    "cluster_name" : "elasticsearch",
    "cluster_uuid" : "Hp-6X4YhR3q3F-2fN63jsA",
    "version" : {
    "number" : "8.4.2",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "89f8c6d8429db93b816403ee75e5c270b43a940a",
    "build_date" : "2022-09-14T16:26:04.382547801Z",
    "build_snapshot" : false,
    "lucene_version" : "9.3.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
     },
    "tagline" : "You Know, for Search"
    }
    ```

### Kibana

#### Installation

```
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.4.2-x86_64.rpm;
rpm -i kibana-8.4.2-x86_64.rpm;
```

#### Configuration

- Uncomment the line elasticsearch.hosts = ['localhost:9200']
- server.host: Node IP Address
- elasticsearch.hosts: ["http://<Node_IP_Address>:9200"]

#### Testing

- After That Open Kibana on http://localhost:5601

### Logstash

##### Installation

```
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.4.2-x86_64.rpm;
rpm -i logstash-8.4.2-x86_64.rpm
```

##### Testing

1. Simple Test:
        `/usr/share/logstash/bin/logstash -e 'input { stdin {  }  } output { elasticsearch { hosts => ["127.0.0.1"] } }'`
   1. Enter any text in stdin
   2. Open Kibana -> Analytics -> Discover -> every input you entered should appear
2. Log File Stream Test:
        `/usr/share/logstash/bin/logstash -e 'input { file { path => "/var/log/secure" start_position => "beginning" } } output { elasticsearch { hosts => ["127.0.0.1"] } }'`
   1. Assuming /var/log/secure is a valid file path and contains data
   2. Events should also appear in Discover Section in Analytics App
   3. The Results were like the following:

![Sessions_Log](./screenshots/ELKRemoteSecureLogFile.jpg)

##### Firewall Bypass

- To Enable Remote Access To ELK Stack Permenantly:
    - Best To Use Firewall-cmd
    - `firewall-cmd --zone=public --add-port=9200/tcp --permanent`
    - `firewall-cmd --zone=public --add-port=5601/tcp --permanent`
    - `firewall-cmd --zone=public --add-port=5044/tcp --permanent` 

###### Offline Installation Of ELK Stack

- for transfering and installation of ELK Stack creation of ansible script can be helpful but requires extra steps to be done like:
  - Packages: Download them using the commands above in the installation section of each component and store them into a folder named files
  - ssh-copy-id: to copy the public key of the host to the node that will be running ELK
  - ansible.cfg and inventory files for node specification and connection configuration
  - playbook that contains all tasks that will be performed for installation
> NOTE: It's Better To Manually Configure ELK On Host Manually, Since Errors/Tests Should Be Performed After Each Configuration To Ensure Everything Is In Sync 


### Filebeat

#### Installation

```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.4.2-amd64.deb
sudo dpkg -i filebeat-8.4.2-amd64.deb
```

#### Configuration

- Modify /etc/filebeats/filebeats.yml:
    - Kibana Connection:
    ```
        # =================================== Kibana ===================================

        # Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
        # This requires a Kibana endpoint configuration.
        setup.kibana:

        # Kibana Host
        # Scheme and port can be left out and will be set to the default (http and 5601)
        # In case you specify and additional path, the scheme is required: http://localhost:5601/path
        # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
                host: "<ELK_NODE_IP_ADDRESS>:5601"
    ```
    - Indexing Options:
      - Elasticsearch Indexing:
      ```
        # ---------------------------- Elasticsearch Output ----------------------------
        output.elasticsearch:
        # Array of hosts to connect to.
                hosts: ["<ELK_NODE_IP_ADDRESS>:9200"]
      ``` 
      - Logstash Indexing:
      ```
        # ------------------------------ Logstash Output -------------------------------
        output.logstash:
        # The Logstash hosts
                hosts: ["<ELK_NODE_IP_ADDRESS>:5044"]
      ```   
- List filebeat modules: `/usr/bin/filebeats modules list`
- Enable Modules: `/usr/bin/filebeats modules enable <modules you want to enable>`
- Modify Enabled Modules: `/etc/filebeats/modules.d/<names of modules enabled>`:
  - Enable Them: `enabled: true`
  - Logs Paths: `var.paths: [<PATH_TO_LOG_SOURCE>]`
  ```
        # Module: system
        # Docs: https://www.elastic.co/guide/en/beats/filebeat/8.4/filebeat-module-system.html
        - module: system
        # Syslog
          syslog:
                enabled: true
                var.paths: ["/var/log/syslog*"]
        # Set custom paths for the log files. If left empty,
        # Filebeat will choose the paths depending on your OS.
        #var.paths:
        # Authorization logs
          auth:
                enabled: true
                var.paths: ["/var/log/auth.log*"]
        # Set custom paths for the log files. If left empty,
        # Filebeat will choose the paths depending on your OS.
        #var.paths:
  ```

- Restart Filebeat service and wait for Elasticsearch to collect and display them in Analytics -> Discover

### Case 1: as.log

#### Case Specification

> This Case is made to test whether an endpoint that contains filebeat will be able to send this custom log to logstash and be able to extract fields from it

- Filebeat.yml Configuration:
  - Enable Logstash as the indexer
  - Enable filebeat.inputs as follows:
  ```
  filebeat.inputs:
        # Each - is an input. Most options can be set at the input level, so
        # you can use different inputs for various configurations.
        # Below are the input specific configurations.

        # filestream is an input for collecting log messages from files.
        - type: filestream

        # Unique ID among all inputs, an ID is required.
        #id: my-filestream-id
        # Change to true to enable this input configuration.
          enabled: true
        # Paths that should be crawled and fetched. Glob based paths.
          paths:
                - /var/log/as.log
  ```

- Logstash Test Without Filebeat:
  ```
        input {
                generator {
                        message => "2020-05-09 08:23:17,549 UTC+0200 WARN  [localhost-startStop-1] com.opentext.ecm.admin.core.EclipseLinkCommonsLoggingAdapter -- metadata_warning_ignore_inheritance_subclass_cach"
                        count => 1
                }
        }
        filter {
                grok {
                        match => {
                                "message" => "^%{TIMESTAMP_ISO8601:timestamp}\s*%{TZ:timezone_type}%{INT:timezone_time}\s*%{LOGLEVEL:log_type}\s*\[%{HOSTNAME:hostname}\]\s*%{GREEDYDATA:log_message}"
                        }
                }
        }
        output {
                elasticsearch {
                        hosts => ["127.0.0.1:9200"]
                        index => "tomcat-index"
                }
        }
  ```

- Filebeat-Logstash Conf File For Pipeline:
  ```
        input {
                beats {
                        port => 5044
                }
        }
        filter {
                grok {
                        match => {
                                "message" => "^%{TIMESTAMP_ISO8601:timestamp}\s*%{TZ:timezone_type}%{INT:timezone_time}\s*%{LOGLEVEL:log_type}\s*\[%{HOSTNAME:hostname}\]\s*%{GREEDYDATA:log_message}"
                        }
                }
        }
        output {
                elasticsearch {
                        hosts => ["127.0.0.1:9200"]
                        index => "tomcat-index"
                }
        }
  ```

1. Go into Stack management -> Index Management, and ensure there's an index with the name tomcat-index
2. Go to Stack Management -> Data Views -> Create Data View, you'll see the tomcat-index in the list -> enter name and index pattern and save, 
3. Go to Analytics -> Discover, and specify the data view
4. you'll see the logs being processed and displayed like in the following:

![as.log](./screenshots/Screenshot%20from%202022-10-01%2012-12-54.png)

> All predefined patterns are here in this path: /usr/share/logstash/vendor/bundle/jruby/2.6.0/gems/logstash-patterns-core-4.3.4/patterns/legacy
