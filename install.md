## Installation
* Get servers ready: 3 servers for the primary cluster, another 1 for the secondary cluster, 2 clients  
* Use of **cephadm-ansible** for the preflight, and **cephadm** for the actual install  
* choose among the 3 servers of primary cluster, the bootstrap node  
* `git clone https://github.com/ceph/cephadm-ansible.git`  
* `ansible-playbook -i hosts cephadm-preflight.yml` -- create the hosts files in prior
* Take care of the registry: if u have an internal registry, make sure it is accessible, and in cas it is insecure, specify it for docker in /etc/docker/daemon.json
  * on all the concerned hosts:
    ```
    [root@serverc ~]# cat /etc/docker/daemon.json 
{
    "insecure-registries" : [ "registry.lab.example.com:5000" ]
}
[root@serverc ~]# 
```

