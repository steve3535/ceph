### Installation
* Ref. https://docs.ceph.com/projects/cephadm-ansible/en/latest/ 
* Get servers ready: 3 servers for the primary cluster, another 1 for the secondary cluster, and another 1 for a client
* Clone the project `git clone https://github.com/ceph/cephadm-ansible.git`  
* You'll need ansible with a hosts inventory:
  * install one through a python venv for e.g.
  * create a hosts inventory like the following:
    ```bash
    [myceph]
    ceph3 ansible_host=10.104.0.134 labels="['mon', 'mgr', 'osd']"
    ceph1 ansible_host=10.104.0.145 labels="['_admin', 'mon', 'mgr', 'osd']"
    ceph2 ansible_host=10.104.0.193 labels="['mon', 'mgr', 'osd']"
    [admin]
    ceph1
    [clients]
    gitlab.example.com ansible_host=10.104.0.179
    ```
* choose among the 3 servers of primary cluster, the bootstrap node - e.g. ceph1 
* **Run the preflight:** `ansible-playbook -i hosts cephadm-preflight.yml`
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
* **Bootstrap:**
  * before running bootstrap, it is stated in the documentation taht we need to ensure ssh passwordless for root from the admin nodes to the others
  * but in fact, it does not work because ceph will try to use a very specific key that is available only after the boot stap on the admin node at */etc/ceph/ceph.pub*
  * Create a playbook like this:
    ```
    #cat site.yml
    ---
    - name: bootstrap the cluster
      hosts: ceph1
      become: true
      gather_facts: false
      tasks:
        - name: bootstrap initial cluster
          cephadm_bootstrap:
           mon_ip: "10.104.0.145"

    - name: add more hosts
      hosts: all
      become: true
      gather_facts: true
      tasks:
        - name: add hosts to the cluster
          ceph_orch_host:
            name: "{{ ansible_facts['hostname'] }}"
            address: "{{ ansible_facts['default_ipv4']['address'] }}"
            labels: "{{ labels }}"
          delegate_to: ceph1

    - name: deploy osd service
      hosts: ceph1
      become: true
      gather_facts: false
      tasks:
        - name: apply osd spec
          ceph_orch_apply:
            spec: |
              service_type: osd
              service_id: osd
              placement:
                host_pattern: '*'
                label: osd
              spec:
                data_devices:
                  all: true

    - name: change osd_default_notify_timeout option
      hosts: ceph1
      become: true
      gather_facts: false
      tasks:
        - name: decrease the value of osd_default_notify_timeout option
          ceph_config:
            action: set
            who: osd
            option: osd_default_notify_timeout
    ``` 
  * *First pass*: run the playbook with expected failing for adding the other nodes to the cluster but expected success for the bootstrap
  * ssh to ceph1, and manually distribute the /etc/ceph/ceph.pub key to the other nodes
  * *Second pass*: run the playbook starting from the adding nodes task: `ansible-playbook -i hosts -vv site.yml --start-at-task="add hosts to the cluster"`
    * if all goes well: we should have the message: BOOTSTRAP COMPLETE along with the initial admin credentials
---

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
  
