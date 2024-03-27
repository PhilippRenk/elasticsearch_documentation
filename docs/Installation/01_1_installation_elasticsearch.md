# Installation Elasticsearch 

| Komponente | Name |
| --- | --- |
|*Link* | https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html |
|*OS:* | Centos Stream 9 |
|Virtualisierung: | vSphere |  
|Server A: | Elasticsearch, Kibana |  
|Server B: | Logstash, Filebeat, Python Scripts | 
|Voraussetzung: | JAVA JDK Version | 


## Download

For `yum` or `dnf` create `elasticsearch.repo` file and include the repo information.

```sh title="repo filepath"
vim /etc/yum.repos.d/elasticsearch.repo
```

!!! abstract "`elasticsearch.repo`"
    ``` conf
    [elasticsearch]
    name=Elastic repository for 8.x packages
    baseurl=https://artifacts.elastic.co/packages/8.x/yum
    gpgcheck=1
    gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    autorefresh=1
    type=rpm-md
    ```
    :material-information-outline: Repo sollte für alle drei Komponenten ausreichen (Elastic, Filebeat, Kibana)


## Installation
Use the following commands on the commandline:

### dnf
=== "crypto policy :material-information-outline:{ title="Only required for RHEL9" }"

    ``` sh 
    update-crypto-policies --set DEFAULT:SHA1
    ```

=== "install"

    ``` sh
    dnf install --enablerepo=elasticsearch elasticsearch
    ```


### systemctl
Start the elasticsearch server with systemctl, enable it on startup.

=== "reload"

    ```sh
    systemctl daemon-reload
    ```

=== "enable"

    ``` sh
    systemctl enable elasticsearch
    ```

=== "start"

    ```sh
    systemctl start elasticsearch
    ```
---
### firewall-cmd
Enable the connection to the elasticsearch-server by adding a port to the firewall *(standard: `9200`)*.

=== "add-port"

    ```
    firewall-cmd --add-port=9200/tcp --permanent
    ```

=== "reload"

    ```
    firewall-cmd --reload
    ```
---

???+ warning "Password output on installation" 
    Elastic password for superuser is generated during installation and written to STDOUT
??? info "Lesson learned" 
    Entweder über eine .repo Datei oder über ein wget die Installation ermöglichen

## Configuration

```sh title="config path"
vim /etc/elasticsearch/elasticsearch.yml
```
Configure the following elements:

- **node.name**: The name which is displayed for this server at the cluster
- **network.host**: Define the IP under which the server is found
- **http.port**: Standard Port is 9200 

??? abstract "`elasticsearch.yml`"

    ```yaml hl_lines="2 6 9 27 33 34" 
    #-------------------------------Node------------------
    node.name: localhost


    # Set the bind address to a specific IP (IPv4 or IPv6): 
        network.host: localhost 
    #
    # Set a custom port for HTTP: 
        http.port: 9200  
    # elasticsearch.host: ["http://localhost:9200"]

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

## Security 
The certificate created during the installation must still be copied to server B (on which Logstash is installed) via ``scp`` and stored in the folder specified in ``logstash.yml``. 

=== "Create certificate"

    ``` sh 
    # Nur wenn unter /etc/elasticsearch/ noch nicht enthalten ist:
    /usr/share/elasticsearch/bin/elasticsearch-keystore create
    ```

=== "Reset password automatically:"

    ``` sh
    /usr/share/elasticsearch/bin/elasticsearch-keystore passwd
    ```

=== "show keystore password" 

    ``` sh
    /usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password
    ```

=== "create CA"

    ```sh
    /usr/share/elasticsearch/bin/elasticsearch-certutil ca
    ```

=== "create certificate"

    ``` sh
    /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
    ```

???+ info "Lesson learned"
    In Elastic 8 oder neu wird SSL im Default aktiviert


Reboot the server!