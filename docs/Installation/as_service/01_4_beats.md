
# Install Filebeat
**Installation**
Stelle sicher, dass Kibana läuft.

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