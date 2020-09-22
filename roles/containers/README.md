MySQL Master/Slave(s) with Docker (for Mac) containers
============================================

Create a MySQL environment with Docker (for Mac). Single server or a Master and multiple slaves.

Requirements
------------

- ansible 2.3.2.0
- Docker for Mac
- docker-py Python module
- pyyaml Python module

For the python modules, just run:
```bash
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
pip2 install docker-py
sudo pip install docker-py
sudo pip install pyyaml
```

Role Variables
--------------

Two main variables you need to know: **"servers"** and **"mysql_image_version"**. **Servers** is the amount of containers you wish to deploy, is an integrer value. 

**mysql_image_version** can have 2 options: 5.6 or 5.7.

The default values are: servers=1 and mysql_image_version=5.6

Example Playbook
----------------

    - hosts: all
      gather_facts: False
      roles:
          - containers

There's just 1 role. To execute it, run a command as follow:

``` 
ansible-playbook /Users/daniel/ansible/docker-mysql/site.yml -i /Users/daniel/ansible/docker-mysql/hosts --extra-vars "servers=5 mysql_image_version=5.7"
```

In the above case, 1 Master and 4 Slaves will be deployed, using MySQL 5.7 (Percona images)

License
-------

BSD

Author Information
------------------

Daniel Guzman Burgos 
daniel.guzman.burgos at percona.com
