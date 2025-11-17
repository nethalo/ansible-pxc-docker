Percona XtraDB Cluster with Docker containers
============================================

Create a PXC environment with Docker. 3 nodes by default

Requirements for Mac
------------

### Tested Environment

This playbook has been tested on:
- **macOS Tahoe 26.0.1** (Build 25A8364)
- **Homebrew 4.6.20**
- **Docker Desktop 28.5.1** (Build e180ab8ab8)
- **Ansible 12** (via Homebrew)

### Installation Steps for macOS

1. **Install Homebrew** (if not already installed):
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

2. **Install Docker Desktop**:
   - Download from [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
   - Or install via Homebrew:
   ```bash
   brew install --cask docker
   ```
   - Start Docker Desktop and ensure it's running

3. **Install Ansible**:
   ```bash
   brew install ansible
   ```

4. **Install Python dependencies**:
   ```bash
   pip3 install docker pyyaml
   ```

   Or if you need to use the older docker-py module:
   ```bash
   pip3 install docker-py pyyaml
   ```

### Minimum Requirements

- Ansible 2.3.2.0 or higher
- Docker for Mac
- Python 3.x with docker (or docker-py) and pyyaml modules

Role Variables
--------------

Two main variables you need to know: **"servers"** and **"mysql_image_version"**.

**Servers** is the amount of containers you wish to deploy, is an integer value.
**mysql_image_version** specifies the Percona XtraDB Cluster Docker image tag (e.g., 5.7, 8.0, 8.0.43)

The default values are: servers=3 and mysql_image_version=8.0.43

For available versions, see the [Percona XtraDB Cluster Docker Hub page](https://hub.docker.com/r/percona/percona-xtradb-cluster/tags) or the [release notes](https://docs.percona.com/percona-xtradb-cluster/8.0/release-notes/8.0.43-34.html)

Example Playbook
----------------

    - hosts: all
      gather_facts: False
      roles:
          - containers

There's just 1 role. To execute it, run commands as follows:

**Deploy with default settings (3 nodes, PXC 8.0.43):**
```bash
ansible-playbook site.yml -i hosts
```

**Deploy with custom settings:**
```bash
# 5-node cluster with PXC 8.0.43
ansible-playbook site.yml -i hosts --extra-vars "servers=5"

# 3-node cluster with PXC 5.7
ansible-playbook site.yml -i hosts --extra-vars "mysql_image_version=5.7"

# 5-node cluster with specific version
ansible-playbook site.yml -i hosts --extra-vars "servers=5 mysql_image_version=8.0.43"
```

### Verifying the Deployment

After the playbook completes, verify your cluster:

```bash
# Check running containers
docker ps

# Check cluster size (should match number of nodes)
docker exec pxc1 mysql -uroot -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

# Check cluster status
docker exec pxc1 mysql -uroot -e "SHOW STATUS LIKE 'wsrep%';"

# Connect to MySQL
docker exec -it pxc1 mysql -uroot
```

### Cleanup

To remove the cluster:

```bash
# Stop and remove all PXC containers
docker stop $(docker ps -q --filter "name=pxc")
docker rm $(docker ps -aq --filter "name=pxc")

# Remove the Docker network
docker network rm mysql
```

License
-------

BSD

Author Information
------------------

Daniel Guzman Burgos 
<daniel.guzman.burgos at percona.com>
