# Installation Elasticsearch - One Node Cluster

# Umgebung:
*Link*:&emsp;&emsp;&emsp;https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html  
*OS:*&emsp;&emsp;&emsp;Centos Stream 9  
Virtualisierung:&emsp;&emsp;&emsp;vSphere  
Server A:&emsp;&emsp;&emsp;Elasticsearch, Kibana  
Server B:&emsp;&emsp;&emsp;Logstash, Filebeat, Python Scripts  
Voraussetzung:&emsp;&emsp;&emsp;JAVA JDK Version  

# Server A
# Elasticsearch Installation

Per Repo-Einbindung Elastic installieren:

```
cd /etc/yum.repos.d/
sudo vim elasticsearch.repo
```
**Info:** Repo sollte für alle drei Komponenten ausreichen (Elastic, Filebeat, Kibana)
```
[elasticsearch]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

```

```
Optional (nur unter RHEL9): update-crypto-policies --set DEFAULT:SHA1
sudo dnf install --enablerepo=elasticsearch elasticsearch
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
firewall-cmd --add-port=9200/tcp --permanent
firewall-cmd --reload
```

*Lesson learned* = Es wird ein *elastic* (Superuser) Passwort nach der Installation im Terminal ausgegeben. Das Passwort ist wichtig für den Zugang zu Elasticsearch. Also am besten speichern. Elasticsearch gibt in der Doku an, das Passwort in eine Umgebungsvariable zu speichern. Erscheint mir unsicher, aber für lokale Instanzen zum ausprobieren sollte es okay sein.  In Elasticsearch 8+ sind die Sicherheitseinstellungen an. Das bedeutet, dass in der elasticsearch.yml mehr Optionen in der eingetragen werden müssen als die initiale Doku zeigt.
*Lesson learned* = Entweder über eine .repo Datei oder über ein wget die Installation ermöglichen

**Konfiguration**
```
sudo vim /etc/elasticsearch/elasticsearch.yml
```

Entferne die # vor den Einträgen:
    network.host: localhost 
Zusätzlich müssen noch die Pfade für *xpack.security.http* und *xpack.security.transport* gesetzt werden.
```  
    
    #-------------------------------Node------------------
    node.name: localhost
    
    
    # Set the bind address to a specific IP (IPv4 or IPv6): 
     network.host: localhost 
    #
    # Set a custom port for HTTP: 
     http.port: 9200                                               # Muss nicht unbedingt gesetzt werden, da der Default 9300 ist.
    # elasticsearch.host: ["http://localhost:9200"]
    
    # --------------------------------- Discovery ----------------------------------
    #
    # Pass an initial list of hosts to perform discovery when this node is started:
    # The default list of hosts is ["127.0.0.1", "[::1]"]
    #
    discovery.seed_hosts: ["127.0.0.1"]
    #
    # Bootstrap the cluster using an initial set of master-eligible nodes:
    #
    #----------------------- BEGIN SECURITY AUTO CONFIGURATION -----------------------
    #
    # The following settings, TLS certificates, and keys have been automatically      
    # generated to configure Elasticsearch security features on 28-08-2023 14:32:16
    #
    # --------------------------------------------------------------------------------
    
    # Enable security features

    xpack.security.enabled: true
    xpack.security.enrollment.enabled: true
    
    # Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
    xpack.security.http.ssl:
      enabled: true
      keystore.path: /etc/elasticsearch/certs/http.p12
    
    # Enable encryption and mutual authentication between cluster nodes
    xpack.security.transport.ssl:
      enabled: true
      verification_mode: certificate
      keystore.path: /etc/elasticsearch/certs/transport.p12
      truststore.path: /etc/elasticsearch/certs/certs/transport.p12
    # Create a new cluster with the current node only
    # Additional nodes can still join the cluster later
    cluster.initial_master_nodes: ["localhost.localdomain"]


```

Das bei der Installation erschaffenene Zertifikat muss noch per ``scp`` an den Server B (auf dem Logstash installiert wird) kopiert werden und in dem Ordner hinterlegt werden, der in der ``logstash.yml`` angegeben ist. 

**Keystore** 
(Muss ich mir noch genauer Anschauen)
```
Nur wenn unter /etc/elasticsearch/ noch nicht enthalten ist:
/usr/share/elasticsearch/bin/elasticsearch-keystore create

Passwort immer neu setzen:
/usr/share/elasticsearch/bin/elasticsearch-keystore passwd

Passwort für den Http.Keystore abfragen. Dies ist für die Konfiguration von Logstash wichtig.
/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password
```

Um eine CA zu erstellen, muss folgender Befehl eingegeben werden:

```
Erstellt eine CA:
/usr/share/elasticsearch/bin/elasticsearch-certutil ca

Erstellt ein Cert(braucht man aber bei einem Server nicht, da Initial Certs mitkommen):
/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```

*Lesson learned:* In Elastic 8 oder neu wird SSL im Default aktiviert

Der Server muss dann rebootet werden.


# Kibana Installation 
Kibana Repo unter ``/etc/yum.repos.d/kibana.repo`` anlegen.


```
sudo dnf install kibana
```

**Konfiguration**
```
sudo vim /etc/kibana/kibana.yml
```

Entferne die # vor den Einträgen:
```

server.port: 5601
server.host: "localhost"                          # Alternative hier eine IP Addresse ohne "" eintragen
elasticsearch.hosts: ["https://localhost:9200"]
```

```
systemctl start kibana
systemctl enable kibana

firewall-cmd --add-port=5601/tcp --permanent
firewall-cmd --reload
```

Einen Enrollment-Token für Kibana erstellen:
```
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token --scope kibana
```
Den Token auf "localhost:5601" im Browser eingeben.
Auf dem Server einen Verifizierungs-Code erstellen und im Browser eingeben:

```
/usr/share/kibana/bin/kibana-verification-code
```
# Server A

## Install Logstash

### User anlegen:
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

### Zertifikate übertragen

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
# Install Filebeat
**Installation**
Stelle sicher, dass Kibana läuft

```
dnf install filebeat

sudo filebeat modules enable system

vim /etc/filebeat/filebeat.yml

```
Die filebeat.yml ist die Konfig-Datei für Filebeat.

**Konfiguration**
Auf jeder Maschine muss eine Filebeat/Beat Konfigurationsdatei angelegt werden. 
```
...
#----------------Filebeat Input--------------------
filebeat.inputs:
- type: log   #Hier wird ein Typ angegeben
  enabled: true
  paths:      #Hier werden die Pfade zu den Daten angeben
    - /var/log/*.log

#--------------------Logstash Output--------------------
output.logstash
  hosts: ["localhost:5044"]
```

Möchte man Module nutzen, müssen diese per ``filebeat module enable MODULNAME`` angeschaltet werden. Die Module liegen unter */etc/filebeat/modules*. Das Kommando benennt die Datei von *MODULNAME-disabled.yml* in *MODULNAME.yml*

```
/usr/share/filebeat/bin/filebeat -e -c /etc/filebeat/filebeat.yml -d "publish"
```