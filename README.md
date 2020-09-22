Percona XtraDB Cluster with Docker containers
============================================

Create a PXC environment with Docker. 3 nodes by default

Requirements for Mac
------------

- ansible 2.3.2.0 (The version that comes with `brew install ansible` will do just fine)
- Docker for Mac
- docker-py Python module
- pyyaml Python module

For the python modules, just run:
```bash
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
sudo pip install docker-py
sudo pip install pyyaml
```

Role Variables
--------------

Two main variables you need to know: **"servers"** and **"mysql_image_version"**. 

**Servers** is the amount of containers you wish to deploy, is an integrer value. 
**mysql_image_version** can have 2 options: 5.7 or 8.0

The default values are: servers=3 and mysql_image_version=5.7

Example Playbook
----------------

    - hosts: all
      gather_facts: False
      roles:
          - containers

There's just 1 role. To execute it, run a command as follow:

``` 
ansible-playbook site.yml -i hosts --extra-vars "servers=5 mysql_image_version=5.7"
```

In the above case, 3-node Galera cluster will be deployed, using Percona XtraDB Cluster 5.7 (Percona images)

License
-------

BSD

Author Information
------------------

Daniel Guzman Burgos 
<daniel.guzman.burgos at percona.com>
