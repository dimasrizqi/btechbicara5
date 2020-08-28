# Demo

step

* setup os machine as a router.
 
1. enable ipv4 packet forwarding

```
vi /etc/sysctl.conf
...
net.ipv4.ip_forward=1
...

sysctl -p
```

2. add Source NAT rule for Internet Access on vm-router

```
iptables -t nat -I POSTROUTING -o ens5 -j MASQUERADE
```

* deploy cluster k8s.

1. execute on cluster node

```
apt update; sudo apt upgrade -y; sudo apt autoremove -y
apt install -y docker.io; sudo docker version
apt install -y apt-transport-https; curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

vi /etc/apt/sources.list.d/kubernetes.list 
...
deb http://apt.kubernetes.io/ kubernetes-xenial main
...

apt update; sudo apt install -y kubectl kubelet kubeadm
```

2. execute on master node

```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

3. install CNI.

```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
wget https://docs.projectcalico.org/manifests/custom-resources.yaml
vi custom-resources.yaml
...
|- 192.168.0.0/16 |+ 10.244.0.0/16
...

kubectl apply -f custom-resources.yaml
```

* setup BGP on vm-router

1. setup BGP.
```
add-apt-repository ppa:cz.nic-labs/bird
apt install bird2
mv /etc/bird/bird.conf /etc/bird/bird.conf.bak
vi /etc/bird/bird.conf 
...
protocol kernel {
    ipv4 {
   	 import none;        # Default is import all
   	 export all;         # Default is export none
    };
   	 scan time 60;       # Scan kernel routing table every 20 seconds
  	 merge paths on;     # Enable ECMP, this parameter requires at least bird 1.6
}

protocol device {
    scan time 10;       # Scan interfaces every 10 seconds
}

protocol static {
	ipv4;
}

protocol bgp mymaster {   
    ipv4 {
	import all;
	export all;
        add paths on;
    };
    description "10.10.10.253";                  # local ip
    local as 65001;                             # local as.It must be different from the as of the port-manager
    neighbor 10.10.10.10 port 17900 as 65000;   # Master node IP and AS number
    source address 10.10.10.253;                 # Router IP 
    #import all; 
    #export all;
    enable route refresh off;
}
...

systemctl restart bird
```

* setup Porter on k8s master.

```
helm repo add test https://charts.kubesphere.io/test
helm repo update
helm install porter test/porter
```

```
vi bgp.yaml
...
apiVersion: network.kubesphere.io/v1alpha1
kind: BgpConf
metadata:
  name: bgpconf-sample
spec:
  # the as of porter
  as : 65000
  routerID : 10.10.10.10
  port: 17900
---
apiVersion: network.kubesphere.io/v1alpha1
kind: BgpPeer
metadata:
  name: bgppeer-sample
spec:
  # the as of the Router
  config:
    peerAs : 65001
    neighborAddress: 10.10.10.253
  addPaths:
    sendMax: 10
...

kubectl apply -f bgp.yaml
```

*setup Eip.

```
vi eip.yaml
...
apiVersion: network.kubesphere.io/v1alpha1
kind: Eip
metadata:
    name: eip-sample-pool
spec:
    # Modify the ip address segment to the ip address segment of the actual environment.
    address: 172.123.123.124
    protocol: bgp
    disable: false
...
```

* test

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    lb.kubesphere.io/v1alpha1: porter
    protocol.porter.kubesphere.io/v1alpha1: bgp
  name: nginx-service
spec:
  selector:
    app: nginx
  type:  LoadBalancer 
  ports:
    - name: http
      port: 8088
      targetPort: 80
```

references: 
- https://github.com/kubesphere/porter/blob/master/doc/simulate_with_bird.md
- https://github.com/kubesphere/porter/blob/master/doc/porter-chart.md
- https://www.youtube.com/watch?v=EjU1yAVxXYQ
