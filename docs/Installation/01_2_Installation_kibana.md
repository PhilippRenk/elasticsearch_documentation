# Installation Kibana 

Remember to include repo on rpm - same as [elastic installation](01_1_installation_elasticsearch.md#download){ data-preview }

```sh title="Install"
sudo dnf install kibana
```

## Konfiguration

```sh 
sudo vim /etc/kibana/kibana.yml
```

```yaml title="change following values"
server.port: 5601
server.host: "localhost"
elasticsearch.hosts: ["https://localhost:9200"]
```

### systemctl
Start the kibana-server with systemctl, enable it on startup.

=== "reload"

    ```sh
    systemctl daemon-reload
    ```

=== "enable"

    ``` sh
    systemctl enable kibana.service
    ```

=== "start"

    ```sh
    systemctl start kibana.service
    ```
---
### firewall-cmd
Enable the connection to the elasticsearch-server by adding a port to the firewall *(standard: `9200`)*.

=== "add-port"

    ```
    firewall-cmd --add-port=5601/tcp --permanent
    ```

=== "reload"

    ```
    firewall-cmd --reload
    ```
---

## Enroll

To enroll kibana automatically generate an enrollment token on the elastic server. You have to utilize the `elasticsearch-create-enrollment-token` binary. 

```sh
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token --scope kibana
```

### Browser
Log into the server with a browser under port 5601 e.g. (http://localhost:5601) and pass the enrollment token. After that a verification code ist requestet. Generate it on the elastic Server with the `kibana-verification-code` binary.

```
/usr/share/kibana/bin/kibana-verification-code
```

### Detatched
You can run a browser detatched enrollment with the `kibana-setup` binary.

```sh
/usr/share/kibana/bin/kibana-setup --enrollment-token
```