# octopik3s the hard way

installation notes and details

## prerequisites

since this is not using gcp, none of those steps are applicable. tmux is being used, and provisioning compute resources was handled with ansible playbooks, to ensure that:

- os is updated
- ensure that
  - cgroups are enabled
  - extraneous services are disabled
  - swap is disabled
  - ip tables is in legacy mode
- required packages are installed on each host

## client tools

ubuntu 20.04 on wsl2 has an older version of cfssl and cfssljson (1.2). removed the apt package, built the binaries from source.

```sh
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ sudo apt-get --purge remove golang-cfssl
[sudo] password for jejs:
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following packages will be REMOVED:
  golang-cfssl*
0 upgraded, 0 newly installed, 1 to remove and 0 not upgraded.
After this operation, 37.0 MB disk space will be freed.
Do you want to continue? [Y/n] y
(Reading database ... 54917 files and directories currently installed.)
Removing golang-cfssl (1.2.0+git20160825.89.7fb22c8-3) ...
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ go get -u github.com/cloudflare/cfssl/cmd/cfssl
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ go/bin/cfssl version
Version: dev
Runtime: go1.13.8
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ go get -u github.com/cloudflare/cfssl/cmd/cfssljson
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ go/bin/cfssljson --version
Version: dev
Runtime: go1.13.8
```

verify that `kubectl` is version 1.18.6 or higher

```sh
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
```

## provisioning compute resources

see [raspberry pik3s](https://github.com/joshuaejs/raspberry-pik3s)

## provisioning a ca and generating tls certificates

ca configuration files:

```txt
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "octopik3s": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "octopik3s",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Ohlone lands",
      "O": "Octopik3s",
      "OU": "CA",
      "ST": "California"
    }
  ]
}
EOF
```

generate the certificates

```sh
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2021/01/30 21:13:28 [INFO] generating a new CA key and certificate from CSR
2021/01/30 21:13:28 [INFO] generate received request
2021/01/30 21:13:28 [INFO] received CSR
2021/01/30 21:13:28 [INFO] generating key: rsa-2048
2021/01/30 21:13:28 [INFO] encoded CSR
2021/01/30 21:13:28 [INFO] signed certificate with serial number {{ long_number }}
```

### client and server certificates

generate admin certificate and key

```txt
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Ohlone lands",
      "O": "system:masters",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=octopik3s \
  admin-csr.json | cfssljson -bare admin
```

generate the kubelet client certificates

```sh
for pi in rpi04 rpi05 rpi06 rpi07 rpi08
do
  cat > ${pi}-csr.json <<EOF
{
  "CN": "system:node:${pi}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Ohlone lands",
      "O": "system:nodes",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

INTERNAL_IP=$(host ${pi}.octopik3s.io | awk '{print $4}')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${pi},${INTERNAL_IP} \
  -profile=octopik3s \
  ${pi}-csr.json | cfssljson -bare ${pi}
done
```

generate the `kube-controller-manager` client certificate and private key

```txt
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "California",
      "O": "system:kube-controller-manager",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=octopik3s \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

generate the `kube-proxy` client certificate and private key

```txt
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Ohlone lands",
      "O": "system:node-proxier",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=octopik3s \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

generate the `kube-scheduler` client certificate and private key

```txt
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Ohlone lands",
      "O": "system:kube-scheduler",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=octopik3s \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

generate the api server certificate and private key

```txt
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Ohlone lands",
      "O": "Octopik3s",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.240.0.1,192.168.50.192,192.168.50.193,192.168.50.194,192.168.50.195,127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=octopik3s \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

> the api server is automagically assigned the `kubernetes` internal dns name, which resolves to the first ip (10.240.0.1) from the range 10.240.0.0/24, used for internal cluster services

generate the `service-account` certificate and private key

```txt
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Ohlone lands",
      "O": "Octopik3s",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=octopik3s \
  service-account-csr.json | cfssljson -bare service-account
  ```

### distribute the certificates

```sh
for pi in rpi04 rpi05 rpi06 rpi07 rpi08
do
  scp ca.pem ${pi}-key.pem ${pi}.pem ${pi}:
done
```

controller certificates

```sh
for pi in rpi01 rpi02 rpi03
do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem ${pi}:
done
```

## generating kubernetes configuration files for authentication

generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients, as well as the `admin` user.

there is no external load-balancer, an internal vip (192.168.50.192) will be created using [keepalived](https://www.keepalived.org) and [haproxy](http://www.haproxy.org/)

!IMG

the `kubelets` and `kubectl` will connect to the apiserver on `192.168.50.192:8443`. `keepalived` will monitor/manage vrrp, and `haproxy` will relay all client connections.

generate `kubeconfigs` for each worker node

```sh
for pi in rpi04 rpi05 rpi06 rpi07 rpi08; do
  kubectl config set-cluster octopik3s-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://192.168.50.192:8443 \
    --kubeconfig=${pi}.kubeconfig

  kubectl config set-credentials system:node:${pi} \
    --client-certificate=${pi}.pem \
    --client-key=${pi}-key.pem \
    --embed-certs=true \
    --kubeconfig=${pi}.kubeconfig

  kubectl config set-context default \
    --cluster=octopik3s-the-hard-way \
    --user=system:node:${pi} \
    --kubeconfig=${pi}.kubeconfig

  kubectl config use-context default --kubeconfig=${pi}.kubeconfig
done
```

generate kubeconfig for the `kube-proxy` service

```txt
kubectl config set-cluster octopik3s-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.50.192:8443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=octopik3s-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

the controllers all connect on `127.0.0.1:6443`

generate kubeconfig for the `kube-controller-manager` service

```txt
kubectl config set-cluster octopik3s-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=octopik3s-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

generate kubeconfig for the `kube-scheduler` service

```txt
kubectl config set-cluster octopik3s-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=octopik3s-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

generate kubeconfig for the `admin` user

```txt
kubectl config set-cluster octopik3s-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=octopik3s-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

distribute the configuration files

```sh
for pi in rpi04 rpi05 rpi06 rpi07 rpi08
do
  scp ${pi}.kubeconfig kube-proxy.kubeconfig ${pi}:
done
```

```sh
for pi in rpi01 rpi02 rpi03
do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${pi}:
done
```

## generating the data encryption config and key

```sh
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

copy `encryption-config.yaml` to each controller

```sh
for pi in rpi01 rpi02 rpi03
do
  scp encryption-config.yaml ${pi}:
done
```

## bootstrapping the etcd cluster

get the current release from [github.com/etcd-io/etcd](https://github.com/etcd-io/etcd/releases)

```sh
wget -q --show-progress --https-only --timestamping "https://github.com/etcd-io/etcd/releases/download/v3.4.14/etcd-v3.4.14-linux-arm64.tar.gz"

tar -xvf etcd-v3.4.14-linux-arm64.tar.gz
sudo mv etcd-v3.4.14-linux-arm64/etcd* /usr/local/bin/

sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

create `/etc/etcd/etcd.conf`

```txt
ETCD_UNSUPPORTED_ARCH=arm64
```

configure etcd service, including workaround for 'unsupported arch arm64'

```txt
ETCD_NAME=$(hostname -s)
INTERNAL_IP=$(host $(hostname -s).octopik3s.io | awk '{print $4}')

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
EnvironmentFile=/etc/etcd/etcd.conf
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster rpi01=https://192.168.50.193:2380,rpi02=https://192.168.50.194:2380,rpi03=https://192.168.50.195:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

restart systemd and start etcd

```txt
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

if there are no errors, verify the etcd cluster members

```txt
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem

601fa20fbb620f8a, started, rpi02, https://192.168.50.194:2380, https://192.168.50.194:2379, false
729d5d0a6cd23e32, started, rpi01, https://192.168.50.193:2380, https://192.168.50.193:2379, false
c6b9c056ddc35df5, started, rpi03, https://192.168.50.195:2380, https://192.168.50.195:2379, false
```

## bootstrapping the kubernetes control plane

this will bootstrap and configure the kubernetes control plane for high availability on each controller, as well as install `kube-apiserver`, `kube-controller-manager`, and `kube-scheduler`.

(using tmux)

```txt
sudo mkdir -p /etc/kubernetes/config
```

download the latest official release [binaries](https://kubernetes.io/docs/setup/release/notes/)

```txt
wget -q --show-progress --https-only --timestamping "https://dl.k8s.io/v1.20.0/kubernetes-server-linux-arm64.tar.gz"

tar -zxf kubernetes-server-linux-arm64.tar.gz
sudo mv kubernetes/server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /usr/local/bin/
```

### configure the api server

```txt
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

create the kube-apiserver.service systemd file

```txt
INTERNAL_IP=$(host $(hostname -s).octopik3s.io | awk '{print $4}')

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.50.193:2379,https://192.168.50.194:2379,https://192.168.50.195:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config='api/all=true' \\
  --service-account-issuer=api \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.240.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### configure the controller manager

```txt
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.240.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### configure the scheduler

*note*: the apiVersion for kubescheduler must be updated to `v1beta1`, this will be going away in 1.23

```txt
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --leader-elect=true \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### start the controller services

```txt
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

### rbac for kubelet authorization

*note*: the apiVersion must be updated to `v1`

on one controller

```txt
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

```txt
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

### add a load-balancer

as i am not using gcp, i have to setup something as an external load-balancer...`keepalived` and `haproxy` are a common solution.

generate `keepalived.conf`

```txt
VRRP_KEY=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 24 | head -n 1)

cat > keepalived.conf <<EOF
vrrp_instance VI_1 {
    state BACKUP
    priority 100
    interface eth0
    virtual_router_id 42
    advert_int 1
    nopreempt
    authentication {
        auth_type AH
        auth_pass ${VRRP_KEY}
    }
    virtual_ipaddress {
        192.168.50.192
    }
}
EOF
```

copy to controllers

`for pi in rpi01 rpi02 rpi03 ; do scp keepalived.conf ${pi}: ; done`

on the controller nodes

```txt
sudo apt-get update && sudo apt-get -y install keepalived haproxy
sudo mkdir -p /etc/keepalived
sudo mv keepalived.conf /etc/keepalived/
```

append the following to `/etc/haproxy/haproxy.cfg`

```txt
frontend stats
  bind *:8080
  mode http
  stats uri /haproxy?stats

frontend octopik3s
  bind 192.168.50.192:8443
  bind 127.0.0.1:8443
  mode tcp
  option tcplog
  timeout client 4h
  default_backend octopik3s

backend octopik3s
  option httpchck GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  timeout server 4h
  balance roundrobin
  server octopik3s-1 192.168.50.193:6443 check
  server octopik3s-2 192.168.50.194:6443 check
  server octopik3s-3 192.168.50.195:6443 check
```

start the services

```txt
sudo systemctl daemon-reload
sudo systemctl enable keepalived haproxy
sudo systemctl start keeplived haproxy
```

!IMG

## bootstrapping the worker nodes

(done via tmux to all workers)

### download and install worker binaries

- [cni-plugins](https://github.com/containernetworking/plugins/releases/)
  - the `overlay` kernel module doesn't exist/won't load which using these binaries
  - using the apt package `containernetworking-plugins`
- [runc](https://github.com/opencontainers/runc/releases)
  - there is no arm64 binary; will have to use apt
- [containerd](https://github.com/containerd/containerd/releases)
  - there is no arm64 binary, will have to use apt
- [kubernetes binaries](ttps://kubernetes.io/docs/setup/release/notes/)
- [cri-tools](https://github.com/kubernetes-sigs/cri-tools/releases/)

```txt
sudo mkdir -p \
  /etc/cni/net.d \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes


wget -q --show-progress --https-only --timestamping \
  https://dl.k8s.io/v1.20.0/kubernetes-node-linux-arm64.tar.gz \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.20.0/crictl-v1.20.0-linux-arm64.tar.gz


sudo tar -zxf crictl-v1.20.0-linux-arm64.tar.gz -C /usr/local/bin/
tar -xvf kubernetes-node-linux-arm64-tar.gz
rm -f kubernetes/node/bin/kubeadm
sudo mv kubernetes/node/bin/k* /usr/local/bin/
rm -rf kubernetes
```

### configure cni networking

create `bridge` network configuration

```txt
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.200.`(hostname -s|cut -c5)`.0/24"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

create `loopback` network configuration

```txt
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### configure containerd

some prerequisites as documented on [container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)

```txt
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

Ensure the arm64 binaries are installed and up-to-date: `sudo apt-get udpate && sudo apt-get install containerd runc`

Create the `containerd` configuration. update the `runtime_engine` with the correct path for the apt installed `runc`

- references
  - [migrating k8s from docker to containerd](https://josebiro.medium.com/migrating-k8s-from-docker-to-containerd-484aaf6baf40)

```txt
sudo mkdir -p /etc/containerd/
sudo containerd config default | sudo tee /etc/containerd/config
```

the systemd `containerd.service` is created when installed by apt.

### configure the kubelet

```txt
sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

create `kubelet-config.yaml`

```txt
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.240.0.10"
podCIDR: "10.200.0.0/16"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

create the `kubelet` service

```txt
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --cni-bin-dir="/usr/lib/cni"
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### configure kube-proxy

create `kube-proxy-config.yaml`

```txt
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

create `kube-proxy` service

```txt
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

verify

```txt
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ kubectl get nodes -o wide
NAME    STATUS     ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
rpi04   NotReady   <none>   2m21s   v1.20.0   192.168.50.196   <none>        Ubuntu 20.04.2 LTS   5.4.0-1015-raspi   containerd://1.3.3-0ubuntu2.2
rpi05   NotReady   <none>   2m21s   v1.20.0   192.168.50.197   <none>        Ubuntu 20.04.2 LTS   5.4.0-1015-raspi   containerd://1.3.3-0ubuntu2.2
rpi06   NotReady   <none>   2m21s   v1.20.0   192.168.50.198   <none>        Ubuntu 20.04.2 LTS   5.4.0-1015-raspi   containerd://1.3.3-0ubuntu2.2
rpi07   NotReady   <none>   2m21s   v1.20.0   192.168.50.199   <none>        Ubuntu 20.04.2 LTS   5.4.0-1015-raspi   containerd://1.3.3-0ubuntu2.2
rpi08   NotReady   <none>   2m20s   v1.20.0   192.168.50.200   <none>        Ubuntu 20.04.2 LTS   5.4.0-1015-raspi   containerd://1.3.3-0ubuntu2.2
```

## configuring kubectl for remote access

done

## provisioning routes

to be completed once kube-proxy/conntrack/kernel modules are working/correct

## deploying the dns cluster add-on

to be completed once kube-proxy/conntrack/kernel modules are working/correct
