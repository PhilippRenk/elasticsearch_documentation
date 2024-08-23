# Useful commands

:octicons-file-code-16:

## `systemctl`
=== "restart elastic"

    ```sh
    systemctl restart elasticsearch.service
    ```

=== "status elastic"

    ```sh
    systemctl status elasticsearch.service
    ```


## `journalctl`
=== "elastic logs"

    ```bash
    journalctl -u elasticsearch.service
    ```


## `curl`
=== "elastic with login"

    ```bash
    curl --cacert config/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200
    ```


# Certificates

The Fingerprint of the http certificate can be obtained with the command:
```bash
openssl x509 -fingerprint -sha256 -in /etc/elasticsearch/certshttp_ca.crt
```

The password for the http.p12 keystore can be obtained with:
```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.http.ssl.keystore.secure_password
```

for the transport.p12:
```bash
/usr/share/elasticsearch/bin/elasticsearch-keystore show xpack.security.transport.ssl.keystore.secure_password
```

# nmcli

## Device information
=== "show device"

    ```sh
    nmcli device show
    ```
=== "show device (pretty)"
  
    ```sh
    nmcli -p device show
    ```

## Individual changes
=== "change ip"

    ```sh
    nmcli con mod enps03 ipv4.addresses "192.168.2.20/24"
    ```
=== "change gateway"

    ```sh
    nmcli con mod enps03 ipv4.gateway "192.168.2.1"
    ```
=== "change dns"

    ```sh
    nmcli con mod enps03 ipv4.dns "8.8.8.8"
    ```
=== "change DHCP to static"

    ```sh
    nmcli con mod enps03 ipv4.method manual
    ```

**commit changes**

??? info inline end
    config is written to `/etc/sysconfig/network-scripts/ifcfg-enps03`

```sh
nmcli con up enps03
```

## nmcli shell script 

<details><summary>shell script to update network connection</summary>

```bash
#!/bin/bash

# Replace these values with your actual IP address, DNS, and connection method
#!/bin/bash

# Set your variables
connection_name="YourConnectionName"
new_ip_address="192.168.1.2"
new_dns="8.8.8.8"
new_method="manual"

# Check if NetworkManager is installed
if ! command -v nmcli &> /dev/null; then
    echo "NetworkManager (nmcli) not found. Please install it."
    exit 1
fi

# Check if the connection exists
if ! nmcli connection show --active | grep -q "$connection_name"; then
    echo "Connection '$connection_name' not found or not active."
    exit 1
fi

# Update the connection settings
nmcli connection modify "$connection_name" ipv4.address "$new_ip_address" 
nmcli connection modify "$connection_name" ipv4.dns "$new_dns" 
nmcli connection modify "$connection_name" ipv4.method "$new_method"

# Restart the connection for changes to take effect
nmcli connection down "$connection_name"
nmcli connection up "$connection_name"

echo "Connection '$connection_name' updated successfully."

```
</details>