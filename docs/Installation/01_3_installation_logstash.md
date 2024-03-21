
# Install Logstash

## User anlegen:
In Kibana (auf Server A) muss unter Management -> Dev Tools per API ein User ``logstash_writer`` und die Rolle ``logstash_writer_role`` erstellt werden.

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

Mit `/user/share/elasticsearch/bin/elasticsearch_reset_password -u logstash_writer` muss das Passwort neu gesetzt werden und dann per Dev Tools in Kibana gesetzt werden.

## Zertifikate übertragen

Übertragung des Zertifikats per
```bash
scp path/to/http_ca.crt user@IP:/home/user
```

Übertragung des SSL-Keys per 
```bash
scp path/to/http.p12 user@IP:/home/user
```

# Server B
**Installation**
Installation von Logstash auf dem Server.

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

# Erstellung des Ordners für die Zertifikate und der Kopiervorgang
mkdir /etc/logstash/certs
cp /home/user/http* /etc/logstash/certs/

# Änderung der Rechte
chown -R logstash:logstash /etc/logstash/

```
**Konfiguration**
Die Konfigurations-Datei von Logstash bearbeiten
```
vim /etc/logstash/logstash.yml
```
Entscheidend sind die X-Pack bzw SSL -Einstellungen:
(Für den logstash_system User muss entweder das Passwort auf dem Elasticserver neu generiert werden oder man kennt es schon)
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

Es muss eine Konfigurationsdatei für die Pipeline von Filebeat -> Logstash -> Elasticsearch konfiguriert werden.
```
vim /etc/logstash/conf.d/logstash.conf
```
*Lesson learned:* Die Conf-Dateien müssen unter conf.d stehen
*Lesson learned*: SSL ist bei Logstash auch ein "Problem".
```

input {
  beats {
     port => 5044
     }
}

filter {    #<---- Hier kann der Filter definiert werden
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
    cacert => '/etc/logstash/certs/http_ca_crt'    #<--- Kann natürlich Variable sein
    user => "logstash_writer"
    password => "PASSWORD"
    ssl => true
  }
}
```

Konfig testen:
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf --config.test_and_exit