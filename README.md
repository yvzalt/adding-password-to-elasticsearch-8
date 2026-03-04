# Securing a 3-Node Elasticsearch Cluster with TLS and Authentication


This repository provides a step-by-step guide to setting up a 3-node Elasticsearch cluster using Vagrant, configuring node discovery, generating/distributing SSL/TLS certificates for the Transport and HTTP layers, and enabling basic password authentication.

## Prerequisites
* Vagrant and a virtualization provider (e.g., VirtualBox) installed.
* A `Vagrantfile` configured to spin up 3 nodes (`es-node-1`, `es-node-2`, `es-node-3`) without initial Elasticsearch configurations.

---

## Phase 1: Initializing and Configuring the Cluster

First, we set up 3 Elasticsearch nodes without connecting them to each other using the Vagrantfile.
Run the following command to start the nodes:
```bash
vagrant up
```

1.Configure the Master Node

Connect to the first node (vagrant ssh es-node-1) and edit the Elasticsearch configuration file:

```bash
sudo vi /etc/elasticsearch/elasticsearch.yml
```
Add the following configuration:

```yaml
cluster.name: es-cluster
node.name: es-node-1
network.host: 192.168.56.101
http.port: 9200
discovery.seed_hosts: ["192.168.56.101", "192.168.56.102", "192.168.56.103"]
xpack.security.enabled: false
cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
```
2.Configure the Data Nodes

Connect to and configure the remaining data nodes with the following configurations.

For es-node-2:

```yaml
cluster.name: es-cluster
node.name: es-node-2
network.host: 192.168.56.102
http.port: 9200
discovery.seed_hosts: ["192.168.56.101", "192.168.56.102", "192.168.56.103"]
xpack.security.enabled: false
cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
```

For es-node-3:

```yaml
cluster.name: es-cluster
node.name: es-node-3
network.host: 192.168.56.103
http.port: 9200
discovery.seed_hosts: ["192.168.56.101", "192.168.56.102", "192.168.56.103"]
xpack.security.enabled: false
cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
```

3.Start Services and Verify

After the configurations are completed, start the Elasticsearch service on each node:

```bash
sudo systemctl start elasticsearch
```

Run a curl command on any node to verify that all nodes have successfully joined the cluster:

```bash
[vagrant@es-node-1 ~]$ curl -X GET "http://192.168.56.101:9200/_cat/nodes?v"
ip             heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
192.168.56.103           51          97  11    0.09    0.38     0.37 cdfhilmrstw *      es-node-3
192.168.56.101           33          96   4    0.23    0.57     0.46 cdfhilmrstw -      es-node-1
192.168.56.102           23          96  12    0.08    0.44     0.46 cdfhilmrstw -      es-node-2
```

Also, check the cluster health:

```bash
[vagrant@es-node-1 ~]$ curl -X GET "http://192.168.56.101:9200/_cluster/health?pretty"
{
  "cluster_name" : "es-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 1,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

---

## Phase 2: Securing the Cluster (Transport Layer TLS)

To activate Elasticsearch password authentication, we must first encrypt the communication between the nodes (Transport Layer).

1.Generate Certificates

On es-node-1, create the Certificate Authority (CA):

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca --out /tmp/elastic-stack-ca.p12 --pass ""
```
Next, create the node certificate using the CA. This .p12 certificate includes the private key and the CA certificate:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca /tmp/elastic-stack-ca.p12 --ca-pass "" --out /tmp/elastic-certificates.p12 --pass ""
```

2.Distribute the Certificates

Send the generated certificate to the other nodes using scp (the default password for the vagrant user is vagrant):

```bash
scp /tmp/elastic-certificates.p12 vagrant@192.168.56.102:/tmp/
scp /tmp/elastic-certificates.p12 vagrant@192.168.56.103:/tmp/
```

On all nodes, copy the certificate to the Elasticsearch directory and change ownership:
```bash
sudo cp /tmp/elastic-certificates.p12 /etc/elasticsearch/certs/
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
```

3.Enable Security in Configuration

Update the /etc/elasticsearch/elasticsearch.yml file on all nodes with the following security settings:

```yaml
xpack.security.enabled: true
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/elastic-certificates.p12
  truststore.path: certs/elastic-certificates.p12

# Optional: Disable HTTP SSL for now
xpack.security.http.ssl:
  enabled: false
```

4.Configure Keystore

Run the following commands on all nodes to set the certificate passwords. Since we created the certificates with an empty password (""), simply press Enter when prompted:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

Restart the Elasticsearch service on all nodes:
```bash
sudo systemctl restart elasticsearch
```

Test the security. It should throw an authentication error (401) as expected:

```bash
[vagrant@es-node-1 ~]$ curl -X GET "http://192.168.56.101:9200/_cat/nodes?v"
{"error":{"root_cause":[{"type":"security_exception","reason":"missing authentication credentials for REST request [/_cat/nodes?v]","header":{"WWW-Authenticate":["Basic realm=\"security\" charset=\"UTF-8\"","ApiKey"]}}],"type":"security_exception","reason":"missing authentication credentials for REST request [/_cat/nodes?v]","header":{"WWW-Authenticate":["Basic realm=\"security\" charset=\"UTF-8\"","ApiKey"]}},"status":401}
```

---

## Phase 3: Setting Passwords

With the transport layer secured, we can now set the passwords. Run this command on one node (e.g., es-node-1):

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -i --url "http://192.168.56.101:9200"
```
(For testing purposes, we set the password to elastic123)

Verify the setup by running the curl command with the new credentials:
```bash
[vagrant@es-node-1 ~]$ curl -u elastic:elastic123 -X GET "http://192.168.56.101:9200/_cat/nodes?v"
ip             heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
192.168.56.101           56          93   2    0.16    0.08     0.07 cdfhilmrstw -      es-node-1
192.168.56.103           31          95   2    0.04    0.06     0.09 cdfhilmrstw -      es-node-3
192.168.56.102           27          96   1    0.19    0.18     0.16 cdfhilmrstw *      es-node-2
```
---

## Phase 4: Activating HTTP SSL (HTTPS)

To secure the HTTP layer, you can reuse the same transport SSL certificate. Edit /etc/elasticsearch/elasticsearch.yml on all nodes:
```yaml
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/elastic-certificates.p12
```
Add the passwords to the HTTP Keystore on all nodes (press Enter for empty passwords):
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.truststore.secure_password
```

Restart the services on all nodes:
```bash
sudo systemctl restart elasticsearch
```
### Final HTTPS Verification

Access Elasticsearch over HTTPS.

```bash
[root@es-node-1 elasticsearch]# curl -k -u elastic:elastic123 -X GET "https://192.168.56.101:9200/_cat/nodes?v"
ip             heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
192.168.56.102           33          95  19    0.94    0.43     0.22 cdfhilmrstw *      es-node-2
192.168.56.103           30          94  19    0.81    0.39     0.18 cdfhilmrstw -      es-node-3
192.168.56.101           15          95  16    0.56    0.32     0.17 cdfhilmrstw -      es-node-1

```


Note: The -k (insecure) flag is used here to ignore the non-trusted certificate warning.
