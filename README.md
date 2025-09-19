# my-home-k8s
This is my ansible plyaybook for home k8s one node cluster. 
Of course, it can be expanded with additional nodes, just perform a `join` on the workers. 

##### k8s cluster specification
- k8s version: 1.34.1
- runtime: containerd
- installation method: kubeadm
- cgroup driver: systemd

##### Requirements, or rather what I used, because you may have a different vision:
- Raspberry 5 (8GB) with SSD disk (SD cards apparently have a tendency to die after a while, although I haven't experienced that yet.)
- ansible
- internet connection ;)

##### What needs to be changed? 
In files:
- `ansible.cfg`, 
- `group_vars/all.yml`
- `inventory/hosts.ini`  
you need to specify:  
- `<RPI_USER>` - user you will use on the target host(host with k8s cluster)
- `<RPI_IP_ADDRESS>` - your target host ip address
- `<RPI_HOST_NAME>` - hostname for you target host

##### To do:  
- prepare Raspberry  
So, I used a Raspberry Pi 5 8GB K8S for the installation (it runs quite well, even though there's still some resources left to use). The playbook is designed so that it can be expanded in the future with worker sections and/or more control planes.
Before launching the playbook, you need to install the operating system on your Raspberry Pi 5. If the firmware is up-to-date, all you need to do is connect the power supply and a network cable. The Raspberry Pi will boot and allow you to download and install your favorite operating system. In my case, it's Ubuntu Server 24.04.3 LTS.  
Create user, configure sudo, e.t.c
- Install ansible on your computer (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)  
- Configure connection to your RPI with ssh keys, I used `id_ed25519`
- clone this repository and get into it
- Install collections: `ansible-galaxy collection install -r collections/requirements.yml`
- run playbook: `ansible-playbook site.yml -b -K -v`  

##### Actions to take after
- Remove taint  
Because it's one node cluster you should remove taint from node, because you want to have conrol plana and worker on one node:  
`kubectl taint nodes --all node-role.kubernetes.io/control-plane-`
- Install network plugin (cni)  
In my case it's `calico`,  so just run the command (ver. 3.30.3):  
`kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml`
- Install load balancer  
in my case it's `MetalLB` (ver. 0.15.2), but you choose what suits you:  
`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml`  
- Configure MetalLB(if you chose it)  

  ip pool:
  ```
  apiVersion: metallb.io/v1beta1
  kind: IPAddressPool
  metadata:
    name: home-pool
    namespace: metallb-system
  spec:
    addresses:
    - 192.168.x.xxx-192.168.x.xxx
  ```
  L2Advertisement:  
  ```
  apiVersion: metallb.io/v1beta1
  kind: L2Advertisement
  metadata:
    name: home-l2
    namespace: metallb-system
  spec: {}
  ```
  and beacause I can't use gratuitous ARP, I'm forced to use good old routing:  
  `sudo ip route add <IP_LB_POOL>/<mask> via <RPI_IP_ADDRESS> dev <YOUR_HOME_NET_INTERFACE>` (manual)  
  or  
  `sudo nmcli connection modify <YOUR_HOME_NET_INTERFACE> +ipv4.routes "<IP_LB_POOL>/<mask> <RPI_IP_ADDRESS>"` (less manual)


##### If everything was done correctly, and it should be ;)

You should get pods:
```
kubectl get pods -A
NAMESPACE        NAME                                       READY   STATUS    RESTARTS   AGE
kube-system      calico-kube-controllers-59556d9b4c-knhc4   1/1     Running   0          11h
kube-system      calico-node-kvtcs                          1/1     Running   0          11h
kube-system      coredns-66bc5c9577-ckdk8                   1/1     Running   0          24h
kube-system      coredns-66bc5c9577-v9m5z                   1/1     Running   0          24h
kube-system      etcd-rpi5-node1                            1/1     Running   1          24h
kube-system      kube-apiserver-rpi5-node1                  1/1     Running   1          24h
kube-system      kube-controller-manager-rpi5-node1         1/1     Running   1          24h
kube-system      kube-proxy-dl2p8                           1/1     Running   0          9h
kube-system      kube-scheduler-rpi5-node1                  1/1     Running   1          24h
metallb-system   controller-6599cd9c46-85kvm                1/1     Running   0          10h
metallb-system   speaker-jq8rt                              1/1     Running   0          10h
```
And resource usage (without limits) similar to:  
```
CONTAINER           NAME                      CPU %               MEM                 DISK                INODES              SWAP
1bf6b8b235fef       coredns                   0.30                15.63MB             36.86kB             11                  0B
4f5e5e823e4bd       kube-proxy                0.00                13.63MB             65.54kB             24                  0B
604e486df8872       calico-node               1.10                155.4MB             335.9kB             105                 0B
763f0484fc5a4       speaker                   0.37                17.89MB             40.96kB             12                  0B
7c114e2d179c1       coredns                   0.27                15.18MB             36.86kB             11                  0B
8cda3296c6f33       kube-controller-manager   1.32                54.25MB             69.63kB             20                  0B
caa0712be7905       controller                0.05                19.94MB             45.06kB             13                  0B
d911c78df37ec       calico-kube-controllers   2.36                17.04MB             40.96kB             11                  0B
e677a78e0d1fc       kube-scheduler            1.42                24.31MB             12.29kB             7                   0B
f883f784ce64a       kube-apiserver            5.67                305.6MB             49.15kB             14                  0B
ffef4275bed32       etcd                      3.30                52.92MB             36.86kB             11                  0B
```

##### License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
