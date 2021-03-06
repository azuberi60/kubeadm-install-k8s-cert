# Multi Master Demo Using Kubernetes the Hard Way 

##  Pre-requisite 

*     Master Node 1 - 2 cpu x 4 GB 
*     Master Node 2 - 2 cpu x 4 GB 
*     Worker Node 1 - 2 cpu x 4 GB 
*     Worker Node 2 - 2 cpu x 4 GB 
*     LoadBalancer  - 1 cpu x 2 GB

---

## Initial Setup 

* Setup ssh keys from loadbalancer node to all other nodes 
* Ensure /etc/hosts file is populated and all servers are able to ping each other 

# Overview of SSL/TLS certificates

##  What are SSL certificates ?

> SSL certificate enables encrypted transfer of sensitive information between a client and a server. The purpose of encryption is to make sure that only the intended recipient will be able to view the information. SSL certificates are used to enable https connection between browser and websites.

##  How to generate SSL certificates ?

> There are multiple toolkits available in the market to create self signed SSL certificates. Most notable of them are - 

*  openssl
*  cfssl
*  easyrsa 

> **Self signed certificates** are useful when you want to enable SSL/TLS envryption for applications that run within your organization. These certificates are not recognized by browsers as the certificate is internal to your organization itself. In order to enable communication with any system outside your organization, you will have to set up MSSL/2 way SSL. 

> There are multiple **third party SSL certificate providers** like Verisign, Symantec, Intouch, Comodo etc. Their Certificate public key is embedded with all major browsers like chrome, IE, safari, mozilla. This enables any external user to connect to your server using a secure HTTPS connection that is recognized by the browser.  

#  Components of SSL certificate 

##  Certificate Authority (CA)

> **CA** are third party trusted entities that issues a **trusted SSL certificate**. Trusted certificate are used to create a secure connection (https) from browser/client to a server that accepts the incoming request. When you create a self-signed certificate for your organization, __**YOU**__ become the CA. 

##  Private/Public key(CSR) & Certificate

 SSL uses the concept of **private/public key pair** to authenticate, secure and manage connection between client and server. They work together to ensure TLS handshake takes place, creating a secure connection (https)

 **Private key** creates your digital signature which will eventually be trusted by any client that tries to connect to your server. With help of private key, you generate a **CSR (certificate signing request)**. Private key is kept on the server and the security of the private key is the sole responsibility of your organization. The private key should never leave your organization. 

 In contrast to private key, a **Public Key** can be distributed to multiple clients. Public Key or CSR is usually submitted to a CA like Comodo/Verisign/Entrust etc, and the CSR (formerly created by your private key) is then signed by the CA. This process generates a SSL/TLS certificate that can now be distributed to any client application. Since this certificate is signed by a trusted CA, your end users can now connect securely to your server (which contains the private key) using their browser. 

 Some third party CA also takes care of generating the private/public key pair for you. This, sometimes, is a good option in case you lose your private key or your private key is compromised. The CA provider takes care of re-keying your certificate with a new private key, and the new private key is then handed over to you. 

 When dealing with self signed certificate, its usually the organization that generates the root CA certificate and acts as the sole CA provider. Any subsequent CSR will be then signed by the root CA. This enables organizations to ensure TLS communication for applications which runs internal to them. 

##  Steps to generate a self signed certificate 

*     Choose a toolkit of your choice (openssl / easyrsa / cfssl ) -- We will use cfssl 
*     Generate root CA private key 
*     Generate a root certificate and self-sign it using the CA private key 
*     Distribute the root CA certificate on ALL the machines who wants to trust you
*     For each application/machine create a new private key 
*     Use the private key to generate a public key (CSR)
*     Ensure the Common Name Field (CN) is set accurately as per your IP address / service name or DNS
*     Sign the CSR with root CA private key and root CA certificate to generate the client certificate
*     Distribute the Client certificate to the corresponding application 

##  What certificates do we need to generate for Kubernetes ?

*   Client certificates for the **kubelet** to authenticate to the **API server**
*   Server certificate for the **apiServer endpoint**
*   Client certificates for **administrators** of the cluster to authenticate to the API server
*   Client certificates for the **apiServer** to talk to the **kubelets (nodes)**
*   Client certificate for the **apiServer** to talk to **etcd**
*   Client certificate/kubeconfig for the **controller manager** to talk to the **apiServer**
*   Client certificate/kubeconfig for the **scheduler** to talk to the **apiServer**
*   **ETCD** client/server certificates for authentication between **each other** and **apiServer**
*   Client certificate for **kube-proxy** to talk to **apiServer**


--- 

## Configuration on LOADBALANCER VIRTUAL MACHINE 

* **Download cfssl binaries**

` curl -s -L -o /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64`

` curl -s -L -o /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64`

` curl -s -L -o /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64`

` chmod +x /bin/cfssl*`

---

* **Download kubectl binary** 

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/linux/amd64/kubectl
chmod +x kubectl
mv kubectl /usr/local/bin
kubectl version --client

```
---

* **Install Loadbalancer (HAPROXY)** 

```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y haproxy
```

---

* **Configure HAPROXY to distribute traffic between the two masters** 

Export the below variables in loadbalancer virtual machine 

```
mastera_ip=x.x.x.x # INSERT MASTERA IP ADDRESS HERE
masterb_ip=x.x.x.x # INSERT MASTERB IP ADDRESS HERE
loadbalancer_ip=x.x.x.x # INSERT LOADBALANCER IP ADDRESS HERE
```

Configure the HAPROXY configuration by adding a frontend and backing configuration in `/etc/haproxy/haproxy.cfg` file

```
cat << EOF | sudo tee -a /etc/haproxy/haproxy.cfg

frontend kubernetes
bind ${loadbalancer_ip}:6443
option tcplog
mode tcp
default_backend kubernetes-master-nodes


backend kubernetes-master-nodes
mode tcp
balance roundrobin
option tcp-check
server mastera ${mastera_ip}:6443 check fall 3 rise 2
server masterb ${masterb_ip}:6443 check fall 3 rise 2
EOF

```

Once configuration is complete - restart HAPROXY services

```
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

---

* **Create CA certificate** 

Create a directory where all certs can be created 

```
mkdir ~/certs
cd certs
```

Create the CA config and CA CSR files that we will leverage to create CA certificate 

```
cat << EOF | sudo tee ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat << EOF | sudo tee ca-csr.json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "CA",
    "ST": "Cork Co."
  }
 ]
}

EOF

```

Create the CA private key and the corresponding certificate that can be distributed to master and worker nodes 

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

ls -ltra
total 28
drwx------ 7 root root 4096 May 29 15:52 ..
-rw-r--r-- 1 root root  232 May 29 15:54 ca-config.json
-rw-r--r-- 1 root root  195 May 29 15:54 ca-csr.json
-rw-r--r-- 1 root root 1363 May 29 15:56 ca.pem
-rw-r--r-- 1 root root 1001 May 29 15:56 ca.csr
-rw------- 1 root root 1675 May 29 15:56 ca-key.pem

```

---

* **Create certificate for ETCD cluster** 


```
cat << EOF | tee kubernetes-csr.json

{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "Kubernetes",
    "ST": "Cork Co."
  }
 ]
}
EOF

```

Export the following variables - 

```
mastera_ip=x.x.x.x # INSERT MASTERA IP ADDRESS HERE
masterb_ip=x.x.x.x # INSERT MASTERB IP ADDRESS HERE
loadbalancer_ip=x.x.x.x # INSERT LOADBALANCER IP ADDRESS HERE
mastera_host=xxxxxx # INSERT MASTER A HOSTNAME HERE
masterb_host=xxxxxx # INSERT MASTER B HOSTNAME HERE
loadbalancer_host=xxxxxx # INSERT Loadbalancer HOSTNAME HERE
```

Create the ETCD certificate using the below cfssl command - 

```
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-hostname=$mastera_host,$masterb_host,$loadbalancer_host,$mastera_ip,$masterb_ip,$loadbalancer_ip,127.0.0.1,kubernetes.default \
-profile=kubernetes kubernetes-csr.json | \
cfssljson -bare kubernetes
```

Verify the certificate DNS by checking the DNS section in the kubernetes.pem file 

```
openssl x509 -in kubernetes.pem -text -noout
```

You should notice something like - 

```
 X509v3 Subject Alternative Name:
                DNS:mastera, DNS:masterb, DNS:loadbalancer, DNS:, DNS:kubernetes.default, IP Address:10.0.1.5, IP Address:10.0.2.4, IP Address:127.0.0.1

```

Copy the CA certificate, Kubernetes certificate & private key to all nodes 

```
scp ca.pem kubernetes.pem kubernetes-key.pem mastera:~/
scp ca.pem kubernetes.pem kubernetes-key.pem masterb:~/
scp ca.pem kubernetes.pem kubernetes-key.pem workera:~/
scp ca.pem kubernetes.pem kubernetes-key.pem workerb:~/

```

### The configuration section of Loadbalancer node ends here. The following section shows I&C on master and worker nodes. 

---


## Install kubelet, docker, kubeadm and kubectl on all master and worker nodes 

The below steps must be performed on both master and worker nodes 

* **Install Docker** 

```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
systemctl enable docker
usermod -aG docker YOUR_ACCOUNT_NAME_HERE
```

---

* **Install kubeadm, kubelet and kubectl (optional) on all nodes**

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb http://apt.kubernetes.io kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list
apt-get update
```

I am interested specifically in the version 1.17.0 of kubernetes at the moment. In order to get the correct suffix for installation, you can get the versions by executing - 

```
curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'

```

The version that we will select is `1.17.0-00`

```
apt-get install -y kubelet=1.17.0-00 kubeadm=1.17.0-00 kubectl=1.17.0-00
```

Verify the version - 

```
root@mastera:~# kubelet --version
Kubernetes v1.17.0

```

---

* **Turn SWAP OFF** 

```
swapoff -a
```

For non cloud users - the below command might be required to disable the swap FS 

```
sed -i '/ swap / s/^/#/' /etc/fstab
```

---

## Install and configure ETCD on the MASTER NODES

* Step 1 : Login to mastera 

* Step 2 : Export some environment variables in the below format 

` export ETCD_NAME=HOSTNAME_OF_THE_NODE` 

` export INTERNAL_IP=IP_ADDRESS_OF_NODE` 

` export INITIAL_CLUSTER=MASTER1_NAME=https://MASTER1_PRIVATE_IP:2380,MASTER2_NAME=https://MASTER2_PRIVATE_IP` 

For mastera - the configuration would look like - 

```
export ETCD_NAME=mastera
export INTERNAL_IP=10.0.1.5
export INITIAL_CLUSTER=mastera=https://10.0.1.5:2380,masterb=https://10.0.2.4:2380
```

* Step 3 : Login to masterb

* Step 4 : Export some environment variables in the below format 

` export ETCD_NAME=HOSTNAME_OF_THE_NODE` 

` export INTERNAL_IP=IP_ADDRESS_OF_NODE` 

` export INITIAL_CLUSTER=MASTER1_NAME=https://MASTER1_PRIVATE_IP:2380,MASTER2_NAME=https://MASTER2_PRIVATE_IP` 

For masterb - the configuration would look like - 

```
export ETCD_NAME=masterb
export INTERNAL_IP=10.0.2.4
export INITIAL_CLUSTER=mastera=https://10.0.1.5:2380,masterb=https://10.0.2.4:2380
```

* Step 5 - Execute the following commands on both the machines (mastera and masterb)

  * Create directory structure 
  
  ```
  sudo mkdir /etc/etcd /var/lib/etcd
  ```
  
  * Copy the CA certificate and ETCD certificates to /etc/etcd location
  
  ```
  sudo mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd
  ```
  
  * Install etcd binaries 
  
  ```
  wget https://github.com/etcd-io/etcd/releases/download/v3.3.13/etcd-v3.3.13-linux-amd64.tar.gz
  tar xvzf etcd-v3.3.13-linux-amd64.tar.gz
  sudo mv etcd-v3.3.13-linux-amd64/etcd* /usr/local/bin/
  chmod +x /usr/local/bin/etcd*
  ```
  
  * Verify ETCD version 
  
  ```
   etcdctl --version
  ```
  
  * **ENSURE STEP 1-4 are completed and verified. All environment variables must be set correctly to correctly bring up etcd. Any discrepancies will lead failure in etcd.**
  
  * Create the etcd systemd unit file 
  
  The below file now utilizes the environment variables set on both the master nodes
  
  ```
  cat << EOF | sudo tee /etc/systemd/system/etcd.service
        [Unit]
        Description=etcd
        Documentation=https://github.com/coreos

        [Service]
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
          --initial-cluster ${INITIAL_CLUSTER} \\
          --initial-cluster-state new \\
          --data-dir=/var/lib/etcd
        Restart=on-failure
        RestartSec=5

        [Install]
        WantedBy=multi-user.target
  EOF
  ```
  
  * Start ETCD
  
  ```
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
  ```
  
  * Verify ETCD
  
  ```
  sudo ETCDCTL_API=3 etcdctl member list \
          --endpoints=https://127.0.0.1:2379 \
          --cacert=/etc/etcd/ca.pem \
          --cert=/etc/etcd/kubernetes.pem \
          --key=/etc/etcd/kubernetes-key.pem
          
  
  ```
  
---

## Initialize master nodes

* STEP 1 - Export the below variables on all the **master nodes**

export mastera_ip=IP OF MASTERA 
export masterb_ip=IP OF MASTERB
export loadbalancer_ip=IP OF LOADBALANCER

The configuration should resemble - 

```
mastera_ip=10.0.1.5
masterb_ip=10.0.2.4
loadbalancer_ip=10.0.1.7
```

* Create a kubeadm cluster configuration file on **both the master nodes**

```
cat << EOF | tee config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
apiServerCertSANs:
- ${loadbalancer_ip}
controlPlaneEndpoint: "${loadbalancer_ip}:6443"
etcd:
  external:
    endpoints:
    - https://${mastera_ip}:2379
    - https://${masterb_ip}:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/kubernetes.pem
    keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 192.168.0.0/16
apiServerExtraArgs:
  apiserver-count: "3"

EOF

```

* Create a directory - `/etc/kubernetes/pki`

```
mkdir -p /etc/kubernetes/pki
```

Some ***words of wisdom*** regarding kubeadm certificates - 

kubeadm manages and installs certificates for kubernetes components without any manual intervention. It also possible to use kubeadm to renew certificates if required. The location where kubeadm places the certificates is - `/etc/kubernetes/pki `

If a given certificate and private key pair both exist, and its content is evaluated compliant with the above specs, the existing files will be used and the generation phase for the given certificate skipped. This means the user can, for example, copy an existing CA to `/etc/kubernetes/pki/ca.{crt,key}`, and then kubeadm will use those files for signing the rest of the certs.

Only for the CA, it is possible to provide the `ca.crt` file but not the `ca.key` file, if all other certificates and kubeconfig files already are in place kubeadm recognize this condition and activates the `ExternalCA` , which also implies the `csrsignercontroller` in controller-manager won’t be started


We will now create a CA authority for kubernetes that will be used to sign all certificates required for components to work - 

```
cd /etc/kubernetes/pki
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=Kubernetes" -days 10000 -out ca.crt

ls -ltra
-rw------- 1 root root 1679 May 29 17:52 ca.key
-rw-r--r-- 1 root root 1115 May 29 17:54 ca.crt

```

* Initialize mastera 

```
cd 
kubeadm init --config=config.yaml

```

Your output should look somewhat like -

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.0.1.7:6443 --token 18w9cj.4wewp36o0rugdfss \
    --discovery-token-ca-cert-hash sha256:f957544240cda3ed215b2dbc301c7127a0528e6d388170ca68c20b60768e4166 \
    --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.1.7:6443 --token 18w9cj.4wewp36o0rugdfss \
    --discovery-token-ca-cert-hash sha256:f957544240cda3ed215b2dbc301c7127a0528e6d388170ca68c20b60768e4166
```

Preserve this output for now 

* Copy `/etc/kubernetes/pki/*` from mastera to masterb at the location `/etc/kubernetes/pki` 

Since our ssh is setup from loadbalancer node - Log in to loadbalancer node first and copy certificates from mastera to loadbalancer

On **loadbalancer node**

```
mkdir pki 
cd pki
scp mastera:/etc/kubernetes/pki/* .
```

Remove apiserver certificates 

```
rm apiserver.*
```

Copy these certificates from **Loadbalancer** to **masterb** 

```
cd ~/pki
scp * masterb:/etc/kubernetes/pki
```

* Initialize masterb

Log on to masterb node 

Verify certificates 

```
ls -ltra /etc/kubernetes/pki

```

Initialize kubeadm

```
kubeadm init --config=config.yaml
```

Preserve the output

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.0.1.7:6443 --token lmdhso.86dtbrqhokg1z5n9 \
    --discovery-token-ca-cert-hash sha256:f957544240cda3ed215b2dbc301c7127a0528e6d388170ca68c20b60768e4166 \
    --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.1.7:6443 --token lmdhso.86dtbrqhokg1z5n9 \
    --discovery-token-ca-cert-hash sha256:f957544240cda3ed215b2dbc301c7127a0528e6d388170ca68c20b60768e4166

```

Take a look at the above output and only select the **join** command for the worker nodes 

```
kubeadm join 10.0.1.7:6443 --token lmdhso.86dtbrqhokg1z5n9 \
    --discovery-token-ca-cert-hash sha256:f957544240cda3ed215b2dbc301c7127a0528e6d388170ca68c20b60768e4166

```

---

## Initialize the worker node 

To initialize the worker node - use the command from above and execute it on both the workers 

```
kubeadm join 10.0.1.7:6443 --token lmdhso.86dtbrqhokg1z5n9 \
    --discovery-token-ca-cert-hash sha256:f957544240cda3ed215b2dbc301c7127a0528e6d388170ca68c20b60768e4166
```

your output should be as below - 

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```


## Set up kubectl on control / loadbalancer node 

Log on to loadbalancer node 

Notice the output from the master initialization once again - 

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

To initialize the loadbalancer node - 

Firstly,

```
  mkdir -p $HOME/.kube
```

Secondly, copy the file /etc/kubernetes/admin.conf from either mastera or masterb to loadbalancer node 

```
scp  mastera:/etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```


Run you first kubectl command - 

```
kubectl get nodes 

kubectl get pods --all-namespaces
```

Notice the nodes in NotReady status - 

```
kubectl get nodes
NAME      STATUS     ROLES    AGE     VERSION
mastera   NotReady   master   45m     v1.17.0
masterb   NotReady   master   7m4s    v1.17.0
workera   NotReady   <none>   4m17s   v1.17.0
workerb   NotReady   <none>   4m18s   v1.17.0

```

---

## CNI installation 

We will use calico for our demo, however there is no restriction to which CNI you prefer 

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```

























