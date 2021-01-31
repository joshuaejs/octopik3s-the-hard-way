# octopik3s the hard way

installation notes and details

## prerequisites

since this is not using gcp, none of those steps are applicable. tmux is being used, and these were handled using an ansible playbook:

- os update
- ensure that
  - cgroups are enabled
  - swap is disabled
  - ip tables is in legacy mode

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
      "L": "Ohone/Costanoan lands",
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
      "L": "Ohone/Costanoan lands",
      "O": "system:masters",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF
```

```sh
joshuaejs@wsl2buntu in ~/repo/othw on main*
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=octopik3s admin-csr.json | cfssljson -bare admin
2021/01/30 21:34:33 [INFO] generate received request
2021/01/30 21:34:33 [INFO] received CSR
2021/01/30 21:34:33 [INFO] generating key: rsa-2048
2021/01/30 21:34:33 [INFO] encoded CSR
2021/01/30 21:34:33 [INFO] signed certificate with serial number {{ long_number}}
2021/01/30 21:34:33 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

generate the kubelet client certificates

```sh
for instance in rpi04 rpi05 rpi06 rpi07 rpi08; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Redwoood City",
      "O": "system:nodes",
      "OU": "Octopik3s The Hard Way",
      "ST": "California"
    }
  ]
}
EOF


INTERNAL_IP=$(host ${instance}.octopik3s.io | awk '{print $4}')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${INTERNAL_IP} \
  -profile=octopik3s \
  ${instance}-csr.json | cfssljson -bare ${instance}
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
      "L": "Ohone/Costanoan lands",
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
      "L": "Ohone/Costanoan lands",
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
      "L": "Ohone/Costanoan lands",
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
  -hostname=127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=octopik3s \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

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
      "L": "Ohone/Costanoan lands",
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
for instance in rpi04 rpi05 rpi06 rpi07 rpi08
do
  scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:
done
```

controller certificates

```sh
for instance in rpi01 rpi02 rpi03
do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem ${instance}:
done
```

## kubeconfigs
