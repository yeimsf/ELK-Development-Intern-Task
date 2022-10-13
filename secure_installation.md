# Secure Installation

## Elastic Stack V8.4.2 [Single-Cluster]

### Elasticsearch

#### Installation

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

### Kibana

#### Installation

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.4.2-x86_64.rpm;
rpm -i kibana-8.4.2-x86_64.rpm;
```

#### Configuration

- `/usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system`
- Uncomment the line elasticsearch.hosts = ['localhost:9200']
- server.host: 0.0.0.0
- elasticsearch.hosts: ["https://<Node_IP_Address>:9200"]
- elasticsearch.username: "kibana_system"
- elasticsearch.password: "KyXKdRSqxtMiUr42Ebl+"
- elasticsearch.ssl.certificateAuthorities: [ "/path/of/elasticsearch/cert/http_ca.crt" ]

### Logstash

#### Installation

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

## Elastic Stack V7.11.2 [Single-Cluster]

### Elasticsearch

#### Installation

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.2-x86_64.rpm;
rpm -i elasticsearch-7.11.2-x86_64.rpm
```

### Kibana

#### Installation

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.11.2-x86_64.rpm;
rpm -i kibana-7.11.2-x86_64.rpm;
```

### Logstash

#### Installation

```bash
wget https://artifacts.elastic.co/downloads/kibana/logstash-7.11.2-x86_64.rpm;
rpm -i logstash-7.11.2-x86_64.rpm;
```

### Security Configuration

#### Setup Minimal Security First

- add these lines to your elasticsearch.yml

```yml
xpack.security.enabled: true
discovery.type: single-node
```

- Start elasticsearch service
- setup passwords for all built-in users:
  - `/usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto #For Auto Generation of Passwords` Or `./bin/elasticsearch-setup-passwords interactive #For Manual Password Setups`
- Uncomment This Line in Kibana:
  - `elasticsearch.username: "kibana_system"`
- Create a keystore to store kibana_system Password:
  - `/usr/share/kibana/bin/kibana-keystore create` PS: if keystore already exists, overwrite it
- Add kibana_system user password to the keystore when prompted by the next command:
  - `/usr/share/kibana/bin/kibana-keystore add elasticsearch.password`
- start kibana and check for connection via browser

#### Setup Basic Security Using SSL/TLS

- Create a certificate authority and enter password for it:
  - `/usr/share/elasticsearch/bin/elasticsearch-certutil ca`, Output: `elastic-stack-ca.p12`
- Generate a certificate for elasticsearch:
  - when prompted enter password for certificate authority used, then you'll be prompted to choose name of output file and after that enter a new password for the certificate itself
  - `/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12`, Output: `elastic-certificates.p12`
- Add These Lines to your elasticsearch.yml:

  ```yml
        cluster.name: my-cluster
        node.name: node-1
        xpack.security.transport.ssl.enabled: true
        xpack.security.transport.ssl.verification_mode: certificate 
        xpack.security.transport.ssl.client_authentication: required
        xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
        xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
  ```

- Add certificate password in keystore and truststore:
  
  ```bash
  ./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password;
  ./bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password;
  ```

- Genearate an SSL certificate for HTTPS:
  - `/usr/share/elasticsearch/bin/elasticsearch-certutil http`
  - When asked if you want to generate a CSR, enter n.
  - When asked if you want to use an existing CA, enter y.
  - Enter the path to your CA. This is the absolute path to the elastic-stack-ca.p12 file that you generated for your cluster.
  - Enter the password for your CA.
  - Enter an expiration value for your certificate. You can enter the validity period in years, months, or days. For example, enter 90D for 90 days.
  - When asked if you want to generate one certificate per node, enter n.
  - List every hostname and variant used to connect to your cluster over HTTPS.
  - Enter the IP addresses that clients can use to connect to your node
  - check if data is correct and enter N to not modify them and output the zip file
- Output is a zip file that contains 2 directories one for elasticsearch and the other for kibana to successfully communicate with elasticsearch

```bash
/elasticsearch
|_ README.txt
|_ http.p12
|_ sample-elasticsearch.yml
/kibana
|_ README.txt
|_ elasticsearch-ca.pem
|_ sample-kibana.yml
```

- add these lines in elasticsearch.yml
  
```yml
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

- add the password of the private key generated above to keystore:
  - `/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password`

- add these lines to kibana.yml:

```yml
elasticsearch.ssl.certificateAuthorities: $KBN_PATH_CONF/elasticsearch-ca.pem
elasticsearch.hosts: https://<your_elasticsearch_host>:9200
```

- enable HTTPS for kibana itself:
  - `/usr/share/elasticsearch/bin/elasticsearch-certutil cert -pem -ca /path/to/elastic-stack-ca.p12 -name kibana-server`
  - enter password of the certificate authority
  - Output is a zip file containing a directory with two files, a certificate and a private key

```bash
/kibana-server
|_ kibana-server.crt
|_ kibana-server.key
```

- add the following lines in kibana.yml:

```yml
server.ssl.enabled: true
server.ssl.certificate: $KBN_PATH_CONF/kibana-server.crt
server.ssl.key: $KBN_PATH_CONF/kibana-server.key
```

- Restart Both Elasticsearch and Kibana

## Elastic Stack V7.11.2 [Multi-Cluster]

### Elasticsearch

#### Installation

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.2-x86_64.rpm;
rpm -i elasticsearch-7.11.2-x86_64.rpm
```

### Kibana

#### Installation

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.11.2-x86_64.rpm;
rpm -i kibana-7.11.2-x86_64.rpm;
```

### Logstash

#### Installation

```bash
wget https://artifacts.elastic.co/downloads/kibana/logstash-7.11.2-x86_64.rpm;
rpm -i logstash-7.11.2-x86_64.rpm;
```

### Security Configuration

#### Setup Minimal Security First

- add this line to your elasticsearch.yml

```yml
xpack.security.enabled: true
```

- Start elasticsearch service
- setup passwords for all built-in users:
  - `/usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto #For Auto Generation of Passwords` Or `./bin/elasticsearch-setup-passwords interactive #For Manual Password Setups`
- Uncomment This Line in Kibana:
  - `elasticsearch.username: "kibana_system"`
- Create a keystore to store kibana_system Password:
  - `/usr/share/kibana/bin/kibana-keystore create` PS: if keystore already exists, overwrite it
- Add kibana_system user password to the keystore when prompted by the next command:
  - `/usr/share/kibana/bin/kibana-keystore add elasticsearch.password`
- start kibana and check for connection via browser

#### Setup Basic Security Using SSL/TLS

- Create a certificate authority and enter password for it:
  - `/usr/share/elasticsearch/bin/elasticsearch-certutil ca`, Output: `elastic-stack-ca.p12`
- Generate a certificate for elasticsearch:
  - `/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12`, Output: `elastic-certificates.p12`
- Add These Lines to your elasticsearch.yml:

  ```yml
        cluster.name: my-cluster
        node.name: node-1
        xpack.security.transport.ssl.enabled: true
        xpack.security.transport.ssl.verification_mode: certificate 
        xpack.security.transport.ssl.client_authentication: required
        xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
        xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
  ```

- Modify `/etc/hosts` on all nodes to include a pattern of nodes DNS as follows:
  - Node1:
    ```shell
    192.168.0.113 node1.elastic.com node1
    ```
  - Node2:
    ```shell
    192.168.0.112 node2.elastic.com node2
    ```
- On master node or `node1` generate HTTPS certificate with all nodes DNS and IPs as follows:
  - `/usr/share/elasticsearch/bin/elasticsearch-certutil http`
  - When asked if you want to generate a CSR, enter n.
  - When asked if you want to use an existing CA, enter y.
  - Enter the path to your CA. This is the absolute path to the elastic-stack-ca.p12 file that you generated for your cluster.
  - Enter an expiration value for your certificate. You can enter the validity period in years, months, or days. For example, enter 90D for 90 days and 1Y for 1 year, etc.
  - When asked if you want to generate one certificate per node, enter n.
  - List every hostname and variant used to connect to your cluster over HTTPS, you can use wildcards if the hostnames have patterns.
  - Enter the IP addresses of these hostnames
  - check if data is correct and enter N to not modify them and output the zip file

- Output is a zip file that contains 2 directories one for elasticsearch and the other for kibana to successfully communicate with elasticsearch

```bash
/elasticsearch
|_ README.txt
|_ http.p12
|_ sample-elasticsearch.yml
/kibana
|_ README.txt
|_ elasticsearch-ca.pem
|_ sample-kibana.yml
```

- copy http.p12 to all nodes in the cluster

- add these lines in elasticsearch.yml in all nodes
  
```yml
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

- add the password of the private key generated above to keystore:
  - `/usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password`

- add these lines to kibana.yml:

```yml
elasticsearch.ssl.certificateAuthorities: $KBN_PATH_CONF/elasticsearch-ca.pem
elasticsearch.hosts: https://<your_elasticsearch_host>:9200
```

> Some Exceptions Can Occur Due To elasticsearch and kibana groups not having read/write permissions on certificates and other files, if they occur just change the group of files resulted in the exception to whichever component will use them and change permissions to read and write to them
