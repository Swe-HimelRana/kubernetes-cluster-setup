## This file describe how to setup kubernetes cluster (Tested on k3s single node)
Version: 2
Updates:
```
* In this vertion we have added Oauth2 Protection for our sensetive app like phpmyadmin, redis commander, kite dashboard. 

* I have used Github OAuth for this current config. But can be used with other oauth2 provider by changing urls and keys.

* I have used traefik IngressRoute instead of kubernetes default ingress for oauth2 protection of sensitive app. But for public app kubernetes default ingress will be enough.

* As I am using traefik extra features I have to use traefik CRD 

```
## 0. At very first install cert-manager CRD and traefik kubernetes CRD
```bash
kubectl apply -f ./cert-manager.yaml # cert-manager.io for SSL
```
verify 
```bash
root@vmi2793283:~/k3s# kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-69f748766f-xpx6q              1/1     Running   0          56s
cert-manager-cainjector-7cf6557c49-7t8fx   1/1     Running   0          56s
cert-manager-webhook-58f4cff74d-vwjzx      1/1     Running   0          56s
```
Apply traefik kubernetes crd

```bash
kubectl create --save-config -f ./traefik-kubernetes-crd.yaml  # Traefik kubernetes CRD for Oauth and traefik IngressRoute
```
Verify
```bash
root@vmi2793283:~/k3s# kubectl get crd | grep traefik.io
accesscontrolpolicies.hub.traefik.io        2025-09-22T09:56:35Z
aiservices.hub.traefik.io                   2025-09-22T09:56:35Z
apiaccesses.hub.traefik.io                  2025-09-22T09:56:35Z
apibundles.hub.traefik.io                   2025-09-22T09:56:35Z
apicatalogitems.hub.traefik.io              2025-09-22T09:56:35Z
apiplans.hub.traefik.io                     2025-09-22T09:56:35Z
apiportals.hub.traefik.io                   2025-09-22T09:56:35Z
apiratelimits.hub.traefik.io                2025-09-22T09:56:35Z
apis.hub.traefik.io                         2025-09-22T09:56:35Z
apiversions.hub.traefik.io                  2025-09-22T09:56:35Z
ingressroutes.traefik.io                    2025-09-22T09:56:35Z
ingressroutetcps.traefik.io                 2025-09-22T09:56:35Z
ingressrouteudps.traefik.io                 2025-09-22T09:56:35Z
managedsubscriptions.hub.traefik.io         2025-09-22T09:56:35Z
middlewares.traefik.io                      2025-09-22T09:56:35Z
middlewaretcps.traefik.io                   2025-09-22T09:56:35Z
serverstransports.traefik.io                2025-09-22T09:56:35Z
serverstransporttcps.traefik.io             2025-09-22T09:56:35Z
tlsoptions.traefik.io                       2025-09-22T09:56:35Z
tlsstores.traefik.io                        2025-09-22T09:56:35Z
traefikservices.traefik.io                  2025-09-22T09:56:35Z
```


## 1. Prepare Cluster Issuer for letsencrypt ssl certificate

Apply cluster issuer Resources for tells cert-manager How and where to obtain SSL Certificate 

```bash 
kubectl apply -f ./cluster-issuer.yaml
```
Verify: 

```bash
root@vmi2793283:~# kubectl get clusterissuer
NAME               READY   AGE
letsencrypt-prod   True    6m16s
```

## 2. Now Enable traefik_forward_auth for sensitive app protection
create namespace
```bash
kubectl create namespace forward-auth
```
apply files
```bash
kubectl apply -f ./traefik_forward_auth/traefik-forward-auth-secret.yaml
kubectl apply -f ./traefik_forward_auth/traefik-forward-auth-deploy.yaml
kubectl apply -f ./traefik_forward_auth/traefik-forward-auth-ingress.yaml
```
verify
```bash
root@vmi2793283:~/k3s# kubectl get service,deployment,ingress -n forward-auth
NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/cm-acme-http-solver-c6djl   NodePort    10.43.185.10   <none>        8089:30156/TCP   33s
service/traefik-forward-auth        ClusterIP   10.43.75.247   <none>        4181/TCP         62s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik-forward-auth   1/1     1            1           62s

NAME                                                     CLASS     HOSTS                       ADDRESS           PORTS     AGE
ingress.networking.k8s.io/cm-acme-http-solver-wrtxw      traefik   auth.web.himelrana.eu.org   194.163.176.149   80        32s
ingress.networking.k8s.io/traefik-forward-auth-ingress   traefik   auth.web.himelrana.eu.org   194.163.176.149   80, 443   36s
```
## 3. Prepare kite kubernetes Dashboard

Deploy kite 

```bash
kubectl apply -f ./kitedash/kite-deploy.yaml
```
Apply ingress

```bash
kubectl apply -f ./kitedash/kite-ingress.yaml
```

Verify 

```bash
root@vmi2793283:~# kubectl get deployments -n kube-system
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
coredns                  1/1     1            1           17h
kite                     1/1     1            1           5h22m
local-path-provisioner   1/1     1            1           17h
metrics-server           1/1     1            1           17h
traefik                  1/1     1            1           17h
root@vmi2793283:~# kubectl get ingress -n kube-system
NAME               CLASS     HOSTS                           ADDRESS           PORTS     AGE
kitedash-ingress   traefik   kitedash.web.himelrana.eu.org   194.163.176.149   80, 443   4h59m
root@vmi2793283:~# 
```

Verify ssl Certificate issued or not 
```bash
root@vmi2793283:~# kubectl describe certificate kite-tls -n kube-system
Name:         kite-tls
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2025-09-19T12:29:56Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  kitedash-ingress
    UID:                   d117ce8b-723f-4e46-aed8-f831d11f1eaa
  Resource Version:        22610
  UID:                     58b99e19-5568-4885-bb1d-b60af2a3d5e0
Spec:
  Dns Names:
    kitedash.web.himelrana.eu.org
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  kite-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2025-09-19T17:15:15Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-12-18T16:16:43Z
  Not Before:              2025-09-19T16:16:44Z
  Renewal Time:            2025-11-18T16:16:43Z
  Revision:                1
Events:
  Type    Reason   Age   From                               Message
  ----    ------   ----  ----                               -------
  Normal  Issuing  10m   cert-manager-certificates-issuing  The certificate has been successfully issued
  ```

# 3 Setup MYSQL with Phpmyadmin
Note: mysql is accessible through port: `serverip:30306` for external use or for local `mysql.mysql.svc.cluster.local` with default port 3306
create mysql namespace 
```bash
kubectl create namespace mysql
```
Apply mysql folder for setting up pv, pvc, secret, configmap, deployment, service and ingress

```bash
kubectl apply -f ./mysql/
```
Verify deployment
```bash
root@vmi2793283:~/k3scluster# kubectl get deployment -n mysql
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
mysql        1/1     1            1           16s
phpmyadmin   1/1     1            1           16s
```
verify service
```bash
root@vmi2793283:~/k3scluster# kubectl get service -n mysql
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
mysql        NodePort    10.43.103.226   <none>        3306:30306/TCP   12m
phpmyadmin   ClusterIP   10.43.237.233   <none>        80/TCP           12m
```
Verify pv, pvc, configmap, secrets

```bash
root@vmi2793283:~/k3scluster# kubectl get pv,pvc,configmap,secrets -n mysql
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/mysql-pv   1Gi        RWO            Retain           Bound    mysql/mysql-pvc   manual         <unset>                          7m23s

NAME                              STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mysql-pvc   Bound    mysql-pv   1Gi        RWO            manual         <unset>                 7m23s

NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      30m
configmap/mysql-config       1      7m23s

NAME                    TYPE                DATA   AGE
secret/mysql-secret     Opaque              1      7m23s
secret/phpmyadmin-tls   kubernetes.io/tls   2      7m19s
```
Verify phpmyadmin ingress
```bash
root@vmi2793283:~/k3scluster# kubectl get ingress -n mysql
NAME                 CLASS     HOSTS                             ADDRESS           PORTS     AGE
phpmyadmin-ingress   traefik   phpmyadmin.web.himelrana.eu.org   194.163.176.149   80, 443   8m47s
```
verify phpmyadmin certificate

```bash
root@vmi2793283:~/k3scluster# kubectl get certificate -n mysql
NAME             READY   SECRET           AGE
phpmyadmin-tls   True    phpmyadmin-tls   9m39s
```
```bash
root@vmi2793283:~/k3scluster# kubectl describe certificate phpmyadmin-tls -n mysql
Name:         phpmyadmin-tls
Namespace:    mysql
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2025-09-19T20:28:06Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  phpmyadmin-ingress
    UID:                   f7066cfa-6932-404c-bc21-c2f9ee444a3d
  Resource Version:        28486
  UID:                     0f3e7826-db43-4b7d-872f-4a5baf63a618
Spec:
  Dns Names:
    phpmyadmin.web.himelrana.eu.org
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  phpmyadmin-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2025-09-19T20:28:10Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-12-18T19:29:38Z
  Not Before:              2025-09-19T19:29:39Z
  Renewal Time:            2025-11-18T19:29:38Z
  Revision:                1
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    11m   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  11m   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "phpmyadmin-tls-hxnb9"
  Normal  Requested  11m   cert-manager-certificates-request-manager  Created new CertificateRequest resource "phpmyadmin-tls-1"
  Normal  Issuing    11m   cert-manager-certificates-issuing          The certificate has been successfully issued
```

# 4 Setup Redis with Commander

Apply redis commander foler (if needed create namespace redis. It should automatically created)
address: `redis.redis.svc.cluster.local` or for external user `serverip:30307` commander and redis both are protected using password (secret)
```bash
kubectl apply -f ./redis/
```
Verify deployment

```bash
root@vmi2793283:~/k3scluster# kubectl get deployment -n redis
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
redis             1/1     1            1           67m
redis-commander   1/1     1            1           67m
```

Verify Service

```bash
root@vmi2793283:~/k3scluster# kubectl get service -n redis
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
redis             NodePort    10.43.228.213   <none>        6379:30307/TCP   69m
redis-commander   ClusterIP   10.43.4.141     <none>        80/TCP           69m
```

Verify pv,pvc,secret,configmap

```bash
root@vmi2793283:~/k3scluster# kubectl get pv,pvc,secret,configmap -n redis
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/mysql-pv   1Gi        RWO            Retain           Bound    mysql/mysql-pvc   manual         <unset>                          94m
persistentvolume/redis-pv   2Gi        RWO            Retain           Bound    redis/redis-pvc   manual         <unset>                          70m

NAME                              STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/redis-pvc   Bound    redis-pv   2Gi        RWO            manual         <unset>                 70m

NAME                          TYPE                DATA   AGE
secret/redis-commander-auth   Opaque              2      54m
secret/redis-commander-tls    kubernetes.io/tls   2      70m
secret/redis-secret           Opaque              3      70m

NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      71m
```

Verify Ingresss of commander

```bash
root@vmi2793283:~/k3scluster# kubectl get ingress -n redis
NAME                      CLASS     HOSTS                        ADDRESS           PORTS     AGE
redis-commander-ingress   traefik   redis.web.himelrana.eu.org   194.163.176.149   80, 443   71m
```

Verify SSL Certificate

```bash
root@vmi2793283:~/k3scluster# kubectl get certificate -n redis
NAME                  READY   SECRET                AGE
redis-commander-tls   True    redis-commander-tls   73m
root@vmi2793283:~/k3scluster# kubectl describe certificate redis-commander-tls -n redis
Name:         redis-commander-tls
Namespace:    redis
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2025-09-19T20:51:32Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  redis-commander-ingress
    UID:                   69a71bbf-02df-4e74-a3a3-9495336cb97c
  Resource Version:        29290
  UID:                     5ec5601e-7269-407b-b024-7f5a65ab28c7
Spec:
  Dns Names:
    redis.web.himelrana.eu.org
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  redis-commander-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2025-09-19T20:52:03Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-12-18T19:53:31Z
  Not Before:              2025-09-19T19:53:32Z
  Renewal Time:            2025-11-18T19:53:31Z
  Revision:                1
Events:                    <none>
```

Verify redis external connection (If you don't want to expose redis externally just do not allow port 30307 from firewall. Also you can disable nodeport)
```bash
docker run -it --rm redis redis-cli -h 194.163.176.149 -p 30307 -a RedisPass123 ping 

Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
PONG
```

# 5 Setup MongoDB with express web client

can be access at `mongodb.mongodb.svc.cluster.local` with default port `27017` or `serverip:30308` for external client. (express can be access through ingress domain)

create namespace (mongodb)

```bash
kubectl create namespace mongodb
```

deploy mongodb with express

```bash
kubectl apply -f ./mongodb/
```

verify deployment

```bash
oot@vmi2793283:~/k3scluster# kubectl get deployment -n mongodb
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
mongo-express   1/1     1            1           29m
mongodb         1/1     1            1           29m
```
verify service

```bash
root@vmi2793283:~/k3scluster# kubectl get service -n mongodb
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
mongo-express   ClusterIP   10.43.98.239    <none>        80/TCP            33m
mongodb         NodePort    10.43.104.222   <none>        27017:30308/TCP   33m
```

verify pv,pvc,secrets,configmap

```bash
root@vmi2793283:~/k3scluster# kubectl get pv,pvc,secret,configmap -n mongodb
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/mongodb-pv   5Gi        RWO            Retain           Bound    mongodb/mongodb-pvc   manual         <unset>                          33m
persistentvolume/mysql-pv     1Gi        RWO            Retain           Bound    mysql/mysql-pvc       manual         <unset>                          134m
persistentvolume/redis-pv     2Gi        RWO            Retain           Bound    redis/redis-pvc       manual         <unset>                          110m

NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mongodb-pvc   Bound    mongodb-pv   5Gi        RWO            manual         <unset>                 33m

NAME                        TYPE                DATA   AGE
secret/mongo-express-auth   Opaque              2      32m
secret/mongo-express-tls    kubernetes.io/tls   2      31m
secret/mongodb-secret       Opaque              3      33m

NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      33m
configmap/mongodb-config     1      
```

Verify ingress

```bash
root@vmi2793283:~/k3scluster# kubectl get ingress -n mongodb
NAME              CLASS     HOSTS                          ADDRESS           PORTS     AGE
express-ingress   traefik   express.web.himelrana.eu.org   194.163.176.149   80, 443   34m
```

Verify Certificate

```bash
root@vmi2793283:~/k3scluster# kubectl get certificate -n mongodb
NAME                READY   SECRET              AGE
mongo-express-tls   True    mongo-express-tls   34m
root@vmi2793283:~/k3scluster# kubectl describe certificate mongo-express-tls -n mongodb
Name:         mongo-express-tls
Namespace:    mongodb
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2025-09-19T22:09:44Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  express-ingress
    UID:                   66d628a2-f199-45cf-88ee-94b34f470653
  Resource Version:        33765
  UID:                     be882ead-2018-4205-90a4-c2d0b78c1984
Spec:
  Dns Names:
    express.web.himelrana.eu.org
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  mongo-express-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2025-09-19T22:11:21Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2025-12-18T21:12:49Z
  Not Before:              2025-09-19T21:12:50Z
  Renewal Time:            2025-11-18T21:12:49Z
  Revision:                1
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    35m   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  35m   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "mongo-express-tls-mztcm"
  Normal  Requested  35m   cert-manager-certificates-request-manager  Created new CertificateRequest resource "mongo-express-tls-1"
  Normal  Issuing    33m   cert-manager-certificates-issuing          The certificate has been successfully issued
  ```

  Note: You can check mongodb remote connection through remote client using nodeport: 30308 (Tested on Studio 3T)


  This is the first setup lot more coming (external secret, helm repo with those config, for x86_64 and ARM  CPU's)
  Written by : Himel (contact@himelrana.com)
  Happy Hacking