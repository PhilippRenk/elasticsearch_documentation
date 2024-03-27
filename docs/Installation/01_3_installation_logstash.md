
# Install Logstash

## Create User:
In Kibana (on Server A), a user ``logstash_writer`` and the role ``logstash_writer_role`` must be created via API under Management -> Dev Tools.

```
POST /_security/role/logstash_write_role
{
    "cluster": [
      "monitor",
      "manage_index_templates"
    ],
    "indices": [
      {
        "names": [
           "filebeat*"
        ],
        "privileges": [
          "write",
          "create_index"
        ],
        "field_security": {
          "grant": [
            "*"
          ]
        }
      }
    ],
    "run_as": [],
    "metadata": {},
    "transient_metadata": {
      "enabled": true
    }
}

POST /_security/user/logstash_writer
{
  "username": "logstash_writer",
  "roles": [
    "logstash_write_role"
  ],
  "full_name": null,
  "email": null,
  "password": "PASSWORD",
  "enabled": true
}
```

The password must be reset with `/user/share/elasticsearch/bin/elasticsearch_reset_password -u logstash_writer` and then set via Dev Tools in Kibana.

## Create certificate

Transfer of the certificate via
```bash
scp path/to/http_ca.crt user@IP:/home/user
```

Transmission of the SSL key via
```bash
scp path/to/http.p12 user@IP:/home/user
```

# Server B
**Installation**
Installation of Logstash on the server.

```
vim /etc/yum.repos.d/logstash.repo
```
Repo-Datei: logstash.repo
```
[logstash-8.x]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
Logstash über die Kommandozeile installieren und Zertifikate in ein Verzeichnis ablegen. Zusätzlich müssen dann die User/Group Berechtigungen von root auf logstash geändert werden, damit Logstash die Zertifikate finden und nutzen kann.
```
# Installation
dnf install logstash

# Creation of the folder for the certificates and the copying process
mkdir /etc/logstash/certs
cp /home/user/http* /etc/logstash/certs/

# Change of rights
chown -R logstash:logstash /etc/logstash/

```
**Konfiguration**
Edit the Logstash configuration file
```
vim /etc/logstash/logstash.yml
```
The X-Pack or SSL settings are decisive:
(For the logstash_system user, either the password must be regenerated on the Elasticserver or you already know it)
```
# X-Pack Management
xpack.management.enabled: true
xpack.management.elasticsearch.username: logstash_system
xpack.management.elasticsearch.password: 'PASSWORD'
xpack.management.elasticsearch.hosts: "https://IP:9200"
xpack.management.elasticsearch.ssl.certificate_authority: "/etc/logstash/certs/http_ca.crt"
xpack.management.elasticsearch.ssl.certificate: "/etc/logstash/certs/http_ca.crt"
xpack.management.elasticsearch.ssl.key: "/etc/logstash/certs/http.p12"
```

A configuration file must be configured for the pipeline from Filebeat -> Logstash -> Elasticsearch.
```
vim /etc/logstash/conf.d/logstash.conf
```
*Lesson learned:* The conf files must be located under conf.d
*Lesson learned*: SSL is also a "problem" with Logstash.
```

input {
  beats {
     port => 5044
     }
}

filter {    #<---- The filter can be defined here
  grok {
    match => {}
  }
  date {
    match => []
    target => ""
  }
}

output {
  elasticsearch {
    hosts => ["https://IP:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}"
    cacert => '/etc/logstash/certs/http_ca_crt'    #<--- Can be variable
    user => "logstash_writer"
    password => "PASSWORD"
    ssl => true
  }
}
```

Testing config:
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf --config.test_and_exit