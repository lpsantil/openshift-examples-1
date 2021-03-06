# etcd Version 2.3.7 - from single node to ha cluster
# Usefull aliases:
<pre>
alias etcdctl="etcdctl --cert-file /etc/etcd/peer.crt  --key-file /etc/etcd/peer.key --ca-file  /etc/etcd/ca.crt"
alias curl="curl --cacert /etc/etcd/ca.crt --cert /etc/etcd/peer.crt   --key /etc/etcd/peer.key"
</pre>

Usefull etcd stats: <code>curl https://$IP$:2379/v2/stats/self |  python -m json.tool<code>


## On etcd ca server (first etcd node)
### Prepare first etcd node
#### Adjust /etc/etcd/etcd.conf
<pre>
--- etcd.conf.backup    2016-10-21 10:42:26.786155551 +0200
+++ etcd.conf   2016-10-21 13:15:26.781088044 +0200
@@ -1,4 +1,6 @@
-ETCD_NAME=default
+
+ETCD_NAME=master1
+ETCD_LISTEN_PEER_URLS=https://10.132.0.6:2380
 ETCD_DATA_DIR=/var/lib/etcd/
 #ETCD_SNAPSHOT_COUNTER=10000
 ETCD_HEARTBEAT_INTERVAL=500
@@ -7,6 +9,12 @@
 #ETCD_MAX_SNAPSHOTS=5
 #ETCD_MAX_WALS=5
 #ETCD_CORS=
+ETCD_DEBUG=true
+#[cluster]
+ETCD_INITIAL_ADVERTISE_PEER_URLS=https://10.132.0.6:2380
+ETCD_INITIAL_CLUSTER="master1=https://10.132.0.6:2380"
+ETCD_INITIAL_CLUSTER_STATE=new
+ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-1

 ETCD_ADVERTISE_CLIENT_URLS=https://10.132.0.6:2379
</pre>

#### Adjust peering adress
<pre>
$ etcdctl -C https://10.132.0.6:2380 member list
57e6b62e0801: name=master1 peerURLs=http://localhost:7001,http://localhost:2380 clientURLs=https://10.132.0.6:2379 isLeader=true
$ curl https://10.132.0.6:2379/v2/members/57e6b62e0801 -XPUT -H "Content-Type: application/json" -d '{"peerURLs":["https://10.132.0.6:2380"]}'
$ etcdctl -C https://10.132.0.6:2380 member list
57e6b62e0801: name=master1 peerURLs=https://10.132.0.6:2380 clientURLs=https://10.132.0.6:2379 isLeader=true
</pre>

### Create certificates for new nodes
<pre>
cd /etc/etcd/ca/
# Set/export IP of know etcd node
export SAN="IP:10.132.0.8"
# Set/export hostname of new etcd node
export PREFIX="master2"

# server request
openssl req -new -keyout ${PREFIX}-server.key -config /etc/etcd/ca/openssl.cnf  -out ${PREFIX}-server.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=${PREFIX}

# server signing
openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out ${PREFIX}-server.crt -in ${PREFIX}-server.csr -extensions etcd_v3_ca_server -batch

# peer request
openssl req -new -keyout ${PREFIX}-peer.key -config /etc/etcd/ca/openssl.cnf -out ${PREFIX}-peer.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=${PREFIX}

# peer signing
openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out ${PREFIX}-peer.crt -in ${PREFIX}-peer.csr -extensions etcd_v3_ca_peer -batch

# Copy certs to know etcd node
tar czvf /tmp/${PREFIX}.certs.tgz ${PREFIX}* ca.crt
....
</pre>

### Add new node to etcd and fetch config
<pre>
$ etcdctl member add master2 https://10.132.0.8:2380
Added member named master2 with ID 4af1b5c984ce9ef2 to cluster

ETCD_NAME="master2"
ETCD_INITIAL_CLUSTER="master2=https://10.132.0.8:2380,master1=http://localhost:2380,master1=http://localhost:7001"
ETCD_INITIAL_CLUSTER_STATE="existing"

</pre>

## On master2 (one of new etcd node)
### Adjust os firewall
Added following lines into /etc/sysconfig/iptables/
<pre>
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 2379 -j ACCEPT
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 2380 -j ACCEPT
</pre>

Restart iptables & openshift-node service! <code>systemctl restart iptables.service atomic-openshift-node.service</code>

### Install etcd & certificates
<pre>
yum install etcd
cd /etc/etcd/
tar xzvf /tmp/master2.certs.tgz
for i in master2-* ; do mv ${i} ${i##master2-}; done;
systemctl enable etcd
</pre>

### Create etcd.conf
<pre>
ETCD_NAME="master2"
ETCD_LISTEN_PEER_URLS=https://10.132.0.8:2380
ETCD_DATA_DIR=/var/lib/etcd/
#ETCD_SNAPSHOT_COUNTER=10000
ETCD_HEARTBEAT_INTERVAL=500
ETCD_ELECTION_TIMEOUT=2500
ETCD_LISTEN_CLIENT_URLS=https://10.132.0.8:2379
#ETCD_MAX_SNAPSHOTS=5
#ETCD_MAX_WALS=5
#ETCD_CORS=
#ETCD_DEBUG=true

#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://10.132.0.8:2380
ETCD_INITIAL_CLUSTER="master1=https://10.132.0.6:2380,master2=https://10.132.0.8:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-1

ETCD_ADVERTISE_CLIENT_URLS=https://10.132.0.8:2379

#[proxy]
#ETCD_PROXY=off

#[security]
ETCD_CA_FILE=/etc/etcd/ca.crt
ETCD_CERT_FILE=/etc/etcd/server.crt
ETCD_KEY_FILE=/etc/etcd/server.key
ETCD_PEER_CA_FILE=/etc/etcd/ca.crt
ETCD_PEER_CERT_FILE=/etc/etcd/peer.crt
ETCD_PEER_KEY_FILE=/etc/etcd/peer.key
</pre>

### Start etcd
<code>systemctl start etcd</code>


### Check member list & cluster-health
<pre>
$ etcdctl -C https://10.132.0.8:2379 cluster-health
member 57e6b62e0801 is healthy: got healthy result from https://10.132.0.6:2379
member 38cd9e653a9061b1 is healthy: got healthy result from https://10.132.0.8:2379
cluster is healthy
$ etcdctl -C https://10.132.0.8:2379 member list
57e6b62e0801: name=master1 peerURLs=https://10.132.0.6:2380 clientURLs=https://10.132.0.6:2379 isLeader=false
38cd9e653a9061b1: name=master2 peerURLs=https://10.132.0.8:2380 clientURLs=https://10.132.0.8:2379 isLeader=true
</pre>

If it is healty add more nodes :-)

### After all nodes added and cluster is healty
#### Adjust etcd.config
Added full list of cluster nodes to etcd.conf:
<pre>
ETCD_INITIAL_CLUSTER="master1=https://10.132.0.6:2380,master2=https://10.132.0.8:2380,master3=https://10.132.0.9:2380"
</pre>

and restart etcd.


# Adjust openshift master

After setup etcd cluster and check cluster-healthy: Adjust <code>/etc/origin/master/master-config.yaml</code> and restart services <code>systemctl restart atomic-openshift-master-controllers.service atomic-openshift-master-api.service</code>


# Just for informations: Restore etcd node from backup
<pre>
$ systemctl stop etcd
$ rm -rf /var/lib/etcd
$ cp -rp /root/etcd_backup /var/lib/etcd
$ etcd -data-dir=/var/lib/etcd/ -force-new-cluster
$ chown -R etcd: /var/lib/etcd
$ systemctl start etcd
$ etcdctl -C https://10.132.0.6:2380 member list
57e6b62e0801: name=master1 peerURLs=http://localhost:7001,http://localhost:2380 clientURLs=https://10.132.0.6:2379 isLeader=true
$ curl https://10.132.0.6:2379/v2/members/57e6b62e0801 -XPUT -H "Content-Type: application/json" -d '{"peerURLs":["https://10.132.0.6:2380"]}'
$ etcdctl member list
57e6b62e0801: name=master1 peerURLs=https://10.132.0.6:2380 clientURLs=https://10.132.0.6:2379 isLeader=true
</pre>
