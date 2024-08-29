## Installation
* Get servers ready: 3 servers for the primary cluster, another 1 for the secondary cluster, 2 clients  
* Use of **cephadm-ansible** for the preflight, and **cephadm** for the actual install  
* choose among the 3 servers of primary cluster, the bootstrap node  
* `git clone https://github.com/ceph/cephadm-ansible.git`  
* `ansible-playbook -i hosts cephadm-preflight.yml` -- create the hosts files in prior
  * here, I skip the -e ceph-origin parameter, because by default, it is set to community | in prod if u have the subscription, u should set it to **-e ceph_origin=rhcs**
  * the playbook will among other things install ceph repos & pkgs  
* Take care of the registry: if u have an internal registry, make sure it is accessible, and in cas it is insecure, specify it for docker in /etc/docker/daemon.json
  * on all the concerned hosts:
    ```bash
    [root@serverc ~]# cat /etc/docker/daemon.json 
     {
       "insecure-registries" : [ "registry.lab.example.com:5000" ]
     }
    [root@serverc ~]# 
    ```  
  * in my case, i skip the registry parameter to let it go straight to quay.io
* Create a ceph directory under root, and inside it the bootstrap file: **initial-config-primary-cluster.yaml** 
* add some disks to be used in the data pool:  vda, vdb, vdc, 
* `cephadm bootstrap --mon-ip=172.25.250.12 --apply-spec=initial-config-primary-cluster.yaml --initial-dashboard-password=redhat --dashboard-password-noupdate --allow-fqdn-hostname --allow-overwrite`
  * In case there was an existing cluster and u need to delete to have a clean state:
    ```bash
    systemctl stop ceph.target
    cephadm rm-cluster --fsid $(sudo cephadm ls | grep -m1 fsid | awk '{print $2}') --force
    rm -rf /etc/ceph/*
    rm -rf /var/lib/ceph/*
    ```
  if all goes well: we should have the message: BOOTSTRAP COMPLETE
