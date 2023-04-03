
<p align="center">
<img width="1000" src="img.png">
</p>

In this article we create an AWS MKS Kafka resource using AWS IAM Roles 
and Policies to authenticate user access. We then create an Aeropsike 
cluster and send some sample messages to AWS Kafka via the Aerospike 
Kafka Source connector. The article is centred around using IAM to 
authenticase users and will guide the reader step by step on how to 
achieve this.

<div className="text--center">

![img_1.png](img_1.png)

</div>
<div align="center">. . . .</div>

## Aerospike Kubernetes Operator

The following Kubernetes nodes have been created using EKS. You can display the following private and public IP addresses from listing the nodes with the kubectl command.

```bash
kubectl get nodes -o wide
NAME                             STATUS   ROLES    AGE     VERSION                INTERNAL-IP      EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                 CONTAINER-RUNTIME
ip-192-168-11-132.ec2.internal   Ready    <none>   2m53s   v1.22.15-eks-fb459a0   192.168.11.132   44.201.67.177    Amazon Linux 2   5.4.219-126.411.amzn2.x86_64   docker://20.10.17
ip-192-168-31-131.ec2.internal   Ready    <none>   2m52s   v1.22.15-eks-fb459a0   192.168.31.131   44.192.83.79     Amazon Linux 2   5.4.219-126.411.amzn2.x86_64   docker://20.10.17
ip-192-168-41-140.ec2.internal   Ready    <none>   2m51s   v1.22.15-eks-fb459a0   192.168.41.140   18.208.222.35    Amazon Linux 2   5.4.219-126.411.amzn2.x86_64   docker://20.10.17
ip-192-168-41-63.ec2.internal    Ready    <none>   2m51s   v1.22.15-eks-fb459a0   192.168.41.63    54.173.138.131   Amazon Linux 2   5.4.219-126.411.amzn2.x86_64   docker://20.10.17
ip-192-168-59-220.ec2.internal   Ready    <none>   2m52s   v1.22.15-eks-fb459a0   192.168.59.220   54.227.122.222   Amazon Linux 2   5.4.219-126.411.amzn2.x86_64   docker://20.10.17
ip-192-168-6-124.ec2.internal    Ready    <none>   2m51s   v1.22.15-eks-fb459a0   192.168.6.124    35.174.60.1      Amazon Linux 2   5.4.219-126.411.amzn2.x86_64   docker://20.10.1
```

Start by getting a copy of the Aerospike git repo for the Kubernetes Operator.

```bash
git clone https://github.com/aerospike/aerospike-kubernetes-operator.git
cp features.conf aerospike-kubernetes-operator/config/samples/secrets/.
```

### Setup

Run the following commands in the order specified. Wait for the csv "Succeeded phase" to show up after running this line. Initially it might take between 30 seconds and a minute to show up.
_kubectl get csv -n operators -w_

```bash
cd aerospike-kubernetes-operator/
kubectl apply -f config/samples/storage/eks_ssd_storage_class.yaml
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.22.0/install.sh | bash -s v0.22.0
kubectl create -f https://operatorhub.io/install/aerospike-kubernetes-operator.yaml
kubectl get csv -n operators -w
cd..
git clone https://github.com/nareshmaharaj-consultant/kubernetes-anything
cd kubernetes-anything
./createNamespace.sh aerospike
cd ../aerospike-kubernetes-operator/
kubectl -n aerospike create secret generic aerospike-secret --from-file=config/samples/secrets
kubectl -n aerospike create secret generic auth-secret --from-literal=password='admin123'
```

## Destination Cluster

Use the following yaml configuration file for our **destination** cluster. Save the file and name it ssd1_xdr_dest_6.1_cluster_cr.yaml:

```yaml
apiVersion: asdb.aerospike.com/v1beta1
kind: AerospikeCluster
metadata:
  name: aerocluster-dest-xdr
  namespace: aerospike

spec:
  size: 1
  image: aerospike/aerospike-server-enterprise:6.1.0.2

  storage:
    filesystemVolumePolicy:
      initMethod: deleteFiles
      cascadeDelete: true
    blockVolumePolicy:
      cascadeDelete: true
    volumes:
      - name: workdir
        aerospike:
          path: /opt/aerospike
        source:
          persistentVolume:
            storageClass: ssd
            volumeMode: Filesystem
            size: 1Gi
      - name: ns
        aerospike:
          path: /opt/aerospike/data/
        source:
          persistentVolume:
            storageClass: ssd
            volumeMode: Filesystem
            size: 1Gi
      - name: aerospike-config-secret
        source:
          secret:
            secretName: aerospike-secret
        aerospike:
          path: /etc/aerospike/secret

  podSpec:
    multiPodPerHost: true

  aerospikeAccessControl:
    roles:
      - name: writer
        privileges:
        - read-write
      - name: reader
        privileges:
        - read
    users:
      - name: admin
        secretName: auth-secret
        roles:
          - sys-admin
          - user-admin
          - read-write
      - name: xdr-writer
        secretName: xdr-user-auth-secret
        roles:
          - writer

  aerospikeConfig:
    service:
      feature-key-file: /etc/aerospike/secret/features.conf
    security: {}
    network:
      service:
        port: 3000
      fabric:
        port: 3001
      heartbeat:
        port: 3002
    namespaces:
      - name: test
        memory-size: 134217728
        replication-factor: 1
        storage-engine:
          type: device
          files:
            - /opt/aerospike/data/test.dat
          filesize: 1073741824
          data-in-memory: true
```

Create the following Kubernetes resources for our Aerospike destination cluster:
- xdr destination database user login credentials, as a Kubernetes secret
- destination database cluster using our yaml file named ssd1_xdr_dest_6.1_cluster_cr.yaml

```bash
export secret_auth_name=xdr-user-auth-secret
export password_secret=admin123
kubectl -n aerospike create secret generic $secret_auth_name --from-literal=password=$password_secret
kubectl create -f ssd1_xdr_dest_6.1_cluster_cr.yaml
kubectl -n aerospike get po -w
```

You should see the database pods running successfully.

```bash
kubectl get po -n aerospike -w
NAME                       READY   STATUS     RESTARTS   AGE
aerocluster-dest-xdr-0-0   0/1     Init:0/1   0          13s
aerocluster-dest-xdr-0-0   0/1     Init:0/1   0          18s
aerocluster-dest-xdr-0-0   0/1     PodInitializing   0          19s
aerocluster-dest-xdr-0-0   1/1     Running           0          24s
```

## XDR-Proxy

Next we set up the **_xdr-proxy_**. If we look back at the main title image above, you will notice that we are working from the RIGHT hand side to the LEFT hand side in that order.

### Configuration

Create the following **_xdr-proxy_** configuration file. Replace the seeds address with a FQN DNS name for the destination database pod(s) you created earlier. Multiple seed addresses may be added (recommended in production).

```yaml
cd ..
mkdir -p xdr-cfg/etc/auth
cd xdr-cfg/etc/

cat <<EOF> aerospike-xdr-proxy.yml
# Change the configuration for your use case.
# Naresh Maharaj
# Refer to https://www.aerospike.com/docs/connectors/enterprise/xdr-proxy/configuration/index.html
# for details.

# The connector's listening ports, manage service, TLS and network interface.
service:
  port: 8901
  # Aerospike Enterprise Server >= 5.0
  manage:
    address: 0.0.0.0
    port: 8902

# The destination aerospike cluster.
aerospike:
  seeds:
    - aerocluster-dest-xdr-0-0.aerospike.svc.cluster.local:
        port: 3000
  credentials:
    username: xdr-writer
    password-file: /etc/aerospike-xdr-proxy/auth/password_DC1.txt
    auth-mode: internal

# The logging config
logging:
  enable-console-logging: true
  file: /var/log/aerospike-xdr-proxy/aerospike-xdr-proxy.log
  levels:
    root: debug
    record-parser: debug
    server: debug
    com.aerospike.connect: debug
  # Ticker log interval in seconds
  ticker-interval: 3600
EOF

sudo tee auth/password_DC1.txt <<EOF
admin123
EOF
cd ..

kubectl -n aerospike create configmap xdr-proxy-cfg --from-file=etc/
kubectl -n aerospike create secret generic xdr-proxy-auth-secret --from-file=etc/auth
```

### Deployment

Now that we have our xdr-proxy config file created we can now produce the **_kubernetes deployment_** yaml file for the xdr-proxy itself. The following yaml file is used to deploy our xdr-proxy ideally in the same data centre or location where the destination databases will be hosted.

```yaml
cat <<EOF> xdr-proxy-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xdr-proxy
  namespace: aerospike
  labels:
    app: xdr-proxy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: xdr-proxy
  template:
    metadata:
      labels:
        app: xdr-proxy
    spec:
      containers:
      - name: xdr-proxy
        image: aerospike/aerospike-xdr-proxy:2.1.0
        volumeMounts:
        - name: xdr-proxy-dir
          mountPath: "/etc/aerospike-xdr-proxy/"
          readOnly: true
        - name: xdr-auth-dir
          mountPath: "/etc/aerospike-xdr-proxy/auth"
          readOnly: true
        ports:
          - name: xdr-proxy-main
            containerPort: 8901
          - name: xdr-proxy-mng
            containerPort: 8902
      volumes:
      - name: xdr-proxy-dir
        configMap:
          name: xdr-proxy-cfg
          optional: false
      - name: xdr-auth-dir
        secret:
          secretName: xdr-proxy-auth-secret
          optional: false
---
apiVersion: v1
kind: Service
metadata:
  name: xdr-proxy
  namespace: aerospike
spec:
  selector:
    app: xdr-proxy
  ports:
  - name: main
    protocol: TCP
    port: 8901
    targetPort: xdr-proxy-main
  - name: manage
    protocol: TCP
    port: 8902
    targetPort: xdr-proxy-mng
EOF

kubectl create -f xdr-proxy-deployment.yaml
kubectl get po -n aerospike -w
```

The following shows the current scheduled pods. So far, so good.

```bash
kubectl get po -n aerospike -w
NAME                         READY   STATUS    RESTARTS   AGE
aerocluster-dest-xdr-0-0     1/1     Running   0          77m
xdr-proxy-7d9fccd6c8-g5mjt   1/1     Running   0          2m26s
xdr-proxy-7d9fccd6c8-mjxp4   1/1     Running   0          2m26s
```

## Source Cluster

Create the **_source_** Aerospike cluster using the following configuration

```yaml
cd ../aerospike-kubernetes-operator/

cat <<EOF> ssd1_xdr_src_6.1_cluster_cr.yaml
apiVersion: asdb.aerospike.com/v1beta1
kind: AerospikeCluster
metadata:
  name: aerocluster-source-xdr
  namespace: aerospike

spec:
  size: 1
  image: aerospike/aerospike-server-enterprise:6.1.0.2

  storage:
    filesystemVolumePolicy:
      initMethod: deleteFiles
      cascadeDelete: true
    blockVolumePolicy:
      cascadeDelete: true
    volumes:
      - name: workdir
        aerospike:
          path: /opt/aerospike
        source:
          persistentVolume:
            storageClass: ssd
            volumeMode: Filesystem
            size: 1Gi
      - name: ns
        aerospike:
          path: /opt/aerospike/data/
        source:
          persistentVolume:
            storageClass: ssd
            volumeMode: Filesystem
            size: 1Gi
      - name: aerospike-config-secret
        source:
          secret:
            secretName: aerospike-secret
        aerospike:
          path: /etc/aerospike/secret

  podSpec:
    multiPodPerHost: true

  aerospikeAccessControl:
    roles:
      - name: writer
        privileges:
        - read-write
      - name: reader
        privileges:
        - read
    users:
      - name: admin
        secretName: auth-secret
        roles:
          - sys-admin
          - user-admin
          - read-write
      
  aerospikeConfig:
    service:
      feature-key-file: /etc/aerospike/secret/features.conf
    security: {}
    network:
      service:
        port: 3000
      fabric:
        port: 3001
      heartbeat:
        port: 3002
    xdr:
      dcs:
        - name: DC2
          connector: true
          node-address-ports:
            - xdr-proxy.aerospike.svc.cluster.local 8901
          namespaces:
            - name: test
    namespaces:
      - name: test
        memory-size: 134217728
        replication-factor: 1
        storage-engine:
          type: device
          files:
            - /opt/aerospike/data/test.dat
          filesize: 1073741824
          data-in-memory: true
EOF

kubectl create -f ssd1_xdr_src_6.1_cluster_cr.yaml
kubectl get po -n aerospike -w
```

From the source cluster, confirm the xdr component has made a connection to the xdr-proxy by filtering the Kubernetes log file as shown in the following kubectl command.

```bash
kubectl -n aerospike logs aerocluster-source-xdr-0-0 -c aerospike-server | grep xdr | grep conn
Dec 08 2022 13:49:21 GMT: INFO (xdr): (dc.c:581) DC DC2 connectedOct 10 2022 13:57:53 GMT: INFO (xdr): (dc.c:581) DC DC2 connected
```

### Simple Message Test

Add some data to the source database and confirm it is being received in the destination cluster. Start by getting the source database service address and connect using the Aerospike's command line tool called aql.

```bash
kubectl get svc -n aerospike
NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
aerocluster-dest-xdr         ClusterIP   None             <none>        3000/TCP            3h38m
aerocluster-dest-xdr-0-0     NodePort    10.100.226.179   <none>        3000:30168/TCP      3h38m
aerocluster-source-xdr       ClusterIP   None             <none>        3000/TCP            33m
aerocluster-source-xdr-0-0   NodePort    10.100.116.173   <none>        3000:31999/TCP      33m
xdr-proxy                    ClusterIP   10.100.44.96     <none>        8901/TCP,8902/TCP   41m
```

```bash
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- aql -U admin -P admin123 -h aerocluster-source-xdr-0-0
Insert a source record using the following command in aql
insert into test.a1 (PK,a,b,c,d) values(1,"A","B","C","D")
OK, 1 record affected.

aql> select * from test
+----+-----+-----+-----+-----+
| PK | a   | b   | c   | d   |
+----+-----+-----+-----+-----+
| 1  | "A" | "B" | "C" | "D" |
+----+-----+-----+-----+-----+
1 row in set (0.023 secs)
```

Connect to the destination cluster in the same way and confirm data has successfully arrived.

```bash
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- aql -U admin -P admin123 -h aerocluster-dest-xdr-0-0
Run the following select query in the destination cluster.
aql> select * from test
+----+-----+-----+-----+-----+
| PK | a   | b   | c   | d   |
+----+-----+-----+-----+-----+
| 1  | "A" | "B" | "C" | "D" |
+----+-----+-----+-----+-----+
1 row in set (0.031 secs)

OK
```

## Interim summary

So far, at this point it's confirmed that xdr-proxy is doing exactly what we need it to do.

If you now review the logs for the initial 2 xdr-proxies that were scheduled you should indeed see userKey=1.

```bash
kubectl logs xdr-proxy-7d9fccd6c8-5q7gn -n aerospike | grep record-parser
2022-12-08 14:53:50.607 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=a1, digest=[120, 48, -23, -90, 110, 126, 84, -1, 114, -116, -9, -21, 28, 75, 126, -68, -51, 83, 31, -117], userKey=1, lastUpdateTimeMs=1670511230565, userKeyString=1, digestString='eDDppm5+VP9yjPfrHEt+vM1TH4s=')

kubectl logs xdr-proxy-7d9fccd6c8-f5zkt -n aerospike | grep record-parser
(none)
```

## Scaling the XDR-Proxies

Go ahead and scale up the xdr-proxy to 6 pods by editing the file xdr-proxy-deployment.yaml and then applying the changes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: xdr-proxy
  namespace: aerospike
  labels:
    app: xdr-proxy
spec:
  replicas: 6
  selector:
    matchLabels:
      app: xdr-proxy
...
kubectl apply -f xdr-proxy-deployment.yaml
```

You can also achieve the same by running the following command

```bash
kubectl scale deploy xdr-proxy -n aerospike --replicas=6
```

You should now have 6 instances of the xdr-proxies running.

```bash
kubectl get po -n aerospike -w
NAME                         READY   STATUS    RESTARTS   AGE
aerocluster-dest-xdr-0-0     1/1     Running   0          3h50m
aerocluster-source-xdr-0-0   1/1     Running   0          75m
xdr-proxy-7d9fccd6c8-49ttl   1/1     Running   0          7s
xdr-proxy-7d9fccd6c8-5q7gn   1/1     Running   0          83m
xdr-proxy-7d9fccd6c8-c4j7k   1/1     Running   0          7s
xdr-proxy-7d9fccd6c8-f5zkt   1/1     Running   0          83m
xdr-proxy-7d9fccd6c8-lscbg   1/1     Running   0          7s
xdr-proxy-7d9fccd6c8-r56vs   1/1     Running   0          7s
```

Lets add some sample messages from our source xdr cluster with primary keys 5,6,7

```bash
aql> insert into test (PK,a,b) values (5,"A","B")
OK, 1 record affected.
aql> insert into test (PK,a,b) values (6,"A","B")
OK, 1 record affected.
aql> insert into test (PK,a,b) values (7,"A","B")
OK, 1 record affected.
```

Notice how we have userKey=5, userKey=6 and userKey=7 across the newly scaled xdr-proxies.

```bash
kubectl logs xdr-proxy-7d9fccd6c8-5q7gn -n aerospike | grep record-parser
2022-12-08 14:53:50.607 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=a1, digest=[120, 48, -23, -90, 110, 126, 84, -1, 114, -116, -9, -21, 28, 75, 126, -68, -51, 83, 31, -117], userKey=1, lastUpdateTimeMs=1670511230565, userKeyString=1, digestString='eDDppm5+VP9yjPfrHEt+vM1TH4s=')

kubectl logs xdr-proxy-7d9fccd6c8-f5zkt -n aerospike | grep record-parser
(none)

kubectl logs xdr-proxy-7d9fccd6c8-49ttl -n aerospike | grep record-parser
(none)

kubectl logs xdr-proxy-7d9fccd6c8-c4j7k -n aerospike | grep record-parser
2022-12-08 15:05:37.511 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=null, digest=[-88, 104, 104, -114, 19, -44, -19, 29, -15, 18, 118, 72, -117, -106, -28, 21, -48, 50, 26, 113], userKey=7, lastUpdateTimeMs=1670511937250, userKeyString=7, digestString='qGhojhPU7R3xEnZIi5bkFdAyGnE=')

kubectl logs xdr-proxy-7d9fccd6c8-lscbg -n aerospike | grep record-parser
2022-12-08 15:05:27.548 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=null, digest=[68, 4, -94, -44, -75, 112, -102, 73, -120, 41, -101, -120, 33, -111, 15, -114, -85, 46, -2, 80], userKey=5, lastUpdateTimeMs=1670511927465, userKeyString=5, digestString='RASi1LVwmkmIKZuIIZEPjqsu/lA=')
2022-12-08 15:05:32.300 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=null, digest=[33, -100, 127, 120, 17, 45, -79, 115, -40, 53, -70, -57, 120, 73, 20, -50, -99, -98, -104, 85], userKey=6, lastUpdateTimeMs=1670511932288, userKeyString=6, digestString='IZx/eBEtsXPYNbrHeEkUzp2emFU=')

kubectl logs xdr-proxy-7d9fccd6c8-r56vs -n aerospike | grep record-parser
(none)
```

When data is actively flowing between source and destination clusters, the existing cached list of xdr-proxy connections from the source cluster is not refreshed, so the newly scaled xdr-proxies we have just scheduled will not be utilized immediately.

To demonstrate, let's add data to the source cluster using Aerospike's benchmark tool. At the same time, we will scale the xdr-proxies on the destination side and observe the results. Before we start, we reduce the xdr-proxy server count to 1 to make the observations clear.

I use my own local machine which has the benchmark tool already installed to send data to the source EC2 instances. In order to do this, we need to get the external IP address of the service.

```bash
kubectl get AerospikeCluster aerocluster-source-xdr -n aerospike  -o yaml

...
pods:
    aerocluster-source-xdr-0-0:
      aerospike:
        accessEndpoints:
        - 192.168.41.63:31999
        alternateAccessEndpoints:
        - 54.173.138.131:31999
        clusterName: aerocluster-source-xdr
        nodeID: 0a0
        tlsAccessEndpoints: []
        tlsAlternateAccessEndpoints: []
        tlsName: ""
      aerospikeConfigHash: 4aacb9809beaa01d99a9f00293c9f7dc141845f8
      hostExternalIP: 54.173.138.131
      hostInternalIP: 192.168.41.63
      image: aerospike/aerospike-server-enterprise:6.1.0.2
      initializedVolumes:
      - workdir
      - ns
      networkPolicyHash: acbbfab3668e1fceeed201139d1173f00095667e
      podIP: 192.168.50.203
      podPort: 3000
      podSpecHash: 972dc2a779fe9ab407212b547d54d3a72ecef259
      servicePort: 31999
...
```

Add the AWS firewall rule to allow traffic into the Kubernetes service. Connect the `asbenchmark` tool to start writing traffic using the public IP address for the NodePort Service.

```bash
asbenchmark -h 54.173.138.131:31999 -Uadmin -Padmin123 -z 10 -servicesAlternate -w RU,0 -o B256
```

Scale up the xdr-proxy servers from 1 to 2, and check the logs of both proxies to see what messages they received. In a production environment, you should disable the logging.

Notice that no data has passed through the new xdr-proxy instance `xdr-proxy-7d9fccd6c8-s2tzt`.

```bash
kubectl -n aerospike logs xdr-proxy-7d9fccd6c8-s2tzt | grep record-parser
(none)

kubectl -n aerospike logs xdr-proxy-7d9fccd6c8-5q7gn | grep record-parser
...
2022-12-08 18:21:36.158 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[51, -120, 114, -17, -44, 72, 123, 125, 50, 92, 3, 110, -21, -38, 74, 25, 42, 35, 117, 72], userKey=null, lastUpdateTimeMs=1670523696059, userKeyString=null, digestString='M4hy79RIe30yXANu69pKGSojdUg=')
2022-12-08 18:21:36.158 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[124, -6, -70, -23, 44, 41, -19, 40, -11, -16, 126, 120, 81, -113, -112, -79, 66, 77, -99, -6], userKey=null, lastUpdateTimeMs=1670523696059, userKeyString=null, digestString='fPq66Swp7Sj18H54UY+QsUJNnfo=')
2022-12-08 18:21:36.158 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-127, -118, 63, -32, -74, 60, -60, 86, 31, -119, -1, -105, -108, -59, 111, 48, -34, -61, -108, -5], userKey=null, lastUpdateTimeMs=1670523696105, userKeyString=null, digestString='gYo/4LY8xFYfif+XlMVvMN7DlPs=')
2022-12-08 18:21:36.256 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[99, 1, 80, 2, 76, -43, 125, 77, 47, 8, 6, 35, 49, 117, -35, 54, 120, -29, 118, -72], userKey=null, lastUpdateTimeMs=1670523696178, userKeyString=null, digestString='YwFQAkzVfU0vCAYjMXXdNnjjdrg=')
2022-12-08 18:21:36.257 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-88, -46, -48, -33, 77, -120, 123, -101, -70, -20, -96, -104, -51, -90, 28, -15, 70, 11, 118, 83], userKey=null, lastUpdateTimeMs=1670523696202, userKeyString=null, digestString='qNLQ302Ie5u67KCYzaYc8UYLdlM=')
2022-12-08 18:21:36.257 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-97, 18, 28, -43, 75, 42, -22, -126, -61, -108, -36, 118, -86, -105, -52, 119, -39, -33, -127, -76], userKey=null, lastUpdateTimeMs=1670523696175, userKeyString=null, digestString='nxIc1Usq6oLDlNx2qpfMd9nfgbQ=')
2022-12-08 18:21:36.257 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-120, 103, -51, 57, -71, -106, 13, -48, 100, 28, 59, -3, -39, -56, -67, -103, 29, 36, 75, 119], userKey=null, lastUpdateTimeMs=1670523696191, userKeyString=null, digestString='iGfNObmWDdBkHDv92ci9mR0kS3c=')
2022-12-08 18:21:36.257 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-47, 88, -13, 13, -35, 77, 24, 22, -40, -61, -118, -115, 82, 13, 127, -125, 53, 66, -22, -8], userKey=null, lastUpdateTimeMs=1670523696233, userKeyString=null, digestString='0VjzDd1NGBbYw4qNUg1/gzVC6vg=')
2022-12-08 18:21:36.258 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-63, -11, 93, -90, 47, 29, -63, 36, 12, 53, -86, 84, 57, -125, 16, 43, -18, 93, 56, 9], userKey=null, lastUpdateTimeMs=1670523696186, userKeyString=null, digestString='wfVdpi8dwSQMNapUOYMQK+5dOAk=')
2022-12-08 18:21:36.258 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-23, 101, -114, -87, -52, 107, 36, 113, 101, 33, -16, 82, -95, 97, 34, -121, 82, -97, 40, 59], userKey=null, lastUpdateTimeMs=1670523696145, userKeyString=null, digestString='6WWOqcxrJHFlIfBSoWEih1KfKDs=')
2022-12-08 18:21:36.258 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-77, -118, 49, 4, -75, 123, 81, 2, -103, -73, 42, -70, -54, 95, 98, 23, 73, 66, -86, 7], userKey=null, lastUpdateTimeMs=1670523696230, userKeyString=null, digestString='s4oxBLV7UQKZtyq6yl9iF0lCqgc=')
2022-12-08 18:21:36.258 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[50, -38, -31, -54, -122, -113, -38, 88, 15, 7, 96, 51, -92, -25, 60, -104, 26, 113, -117, -82], userKey=null, lastUpdateTimeMs=1670523696157, userKeyString=null, digestString='MtrhyoaP2lgPB2AzpOc8mBpxi64=')
2022-12-08 18:21:36.258 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[-77, 31, 67, -18, -52, -114, 42, -18, 36, -111, 89, 62, 109, 114, -54, 54, -121, -110, -88, -108], userKey=null, lastUpdateTimeMs=1670523696206, userKeyString=null, digestString='sx9D7syOKu4kkVk+bXLKNoeSqJQ=')
2022-12-08 18:21:36.258 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[73, 11, 98, -50, 32, 12, 0, -50, 22, -101, -108, 18, 38, 7, -65, 6, -58, 60, -6, -33], userKey=null, lastUpdateTimeMs=1670523696171, userKeyString=null, digestString='SQtiziAMAM4Wm5QSJge/BsY8+t8=')
2022-12-08 18:21:36.258 GMT DEBUG record-parser - parsed message fields: key=Key(namespace='test', set=testset, digest=[109, -82, 24, 53, 35, 89, -72, -117, -22, 79, 119, -89, 56, -5, 0, -103, -54, 51, 25, 126], userKey=null, lastUpdateTimeMs=1670523696146, userKeyString=null, digestString='ba4YNSNZuIvqT3enOPsAmcozGX4=')
...
```

## Conclusion
Dependable resiliency is a core part of the Aerospike value proposition. Aerospike's XDR feature ensures that users can mitigate the risk of a cluster becoming unavailable by replicating data asynchronously from one data center to another. We show here how you can achieve this in a Kubernetes environment by using the xdr-proxy, allowing you to avoid network complications. All this has been achieved with minimum effort thanks to the Aerospike Kubernetes Operator. 