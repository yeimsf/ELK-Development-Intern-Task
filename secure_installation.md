# Secure Installation

## Elasticsearch

### Installation

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.2-x86_64.rpm;
rpm -i elasticsearch-8.4.2-x86_64.rpm;
```

- Superuser password will be presented at stdout of elasticsearch installation
- username = elastic [default], password = <Generated_SuperUser_Password>
- open browser and check on `https://<Node_IP_Address>:9200` and provide the username and password of elasticsearch
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

## Kibana

### Installation

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.4.2-x86_64.rpm;
rpm -i kibana-8.4.2-x86_64.rpm;
```

### Configuration

- `/usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system`
- Uncomment the line elasticsearch.hosts = ['localhost:9200']
- server.host: 0.0.0.0
- elasticsearch.hosts: ["https://<Node_IP_Address>:9200"]
- elasticsearch.username: "kibana_system"
- elasticsearch.password: "KyXKdRSqxtMiUr42Ebl+"
- elasticsearch.ssl.certificateAuthorities: [ "/path/of/elasticsearch/cert/http_ca.crt" ]

## Logstash

### Installation

```bash
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.4.2-x86_64.rpm;
rpm -i logstash-8.4.2-x86_64.rpm
```

- create a file with stdin named test.conf to test the connection and add the following in the output

```conf
input {
        stdin { }
}
output{
        elasticsearch {
                hosts => ['https://192.168.0.113:9200']
                cacert => '/etc/kibana/certs/http_ca.crt'
                user => 'elastic'
                password => '3X3o=RrDH7872TAIjmao'
                ssl => true
                index => 'finally'
        }
}
```

- `/usr/share/logstash/bin/logstash -f /usr/share/logstash/test.conf`
