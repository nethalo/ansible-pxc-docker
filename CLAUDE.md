# CLAUDE.md - AI Assistant Guide for ansible-pxc-docker

## Project Overview

**Purpose:** Automated deployment of Percona XtraDB Cluster (PXC) using Docker containers
**Author:** Daniel Guzman Burgos (daniel.guzman.burgos at percona.com)
**License:** BSD 2-Clause
**Target Platform:** macOS with Docker for Mac
**Ansible Version:** 2.3.2.0+

This Ansible playbook automates the creation of a multi-node Percona XtraDB Cluster (Galera cluster) in Docker containers for development environments. The default setup creates a 3-node cluster, but this is configurable.

## Directory Structure

```
ansible-pxc-docker/
├── README.md                          # User-facing documentation
├── LICENSE                            # BSD 2-Clause license
├── site.yml                           # Main playbook entry point
├── hosts                              # Inventory file (localhost only)
└── roles/
    └── containers/                    # Primary role for PXC deployment
        ├── README.md                  # Role-specific documentation
        ├── meta/
        │   └── main.yml               # Ansible Galaxy metadata
        ├── defaults/
        │   └── main.yml               # Default variables (user-overridable)
        ├── vars/
        │   └── main.yml               # Fixed role variables
        ├── tasks/
        │   ├── main.yml               # Task orchestration (includes other tasks)
        │   ├── images.yml             # Pull Docker images from Docker Hub
        │   ├── network.yml            # Create Docker network for cluster
        │   ├── container.yml          # Create bootstrap node (pxc1)
        │   ├── mysql-conf.yml         # Config file generation (DISABLED)
        │   └── set-replication.yml    # Create additional cluster nodes
        ├── templates/
        │   └── my.cnf.j2              # Jinja2 template for MySQL config
        └── my.cnf.mysql[1-3]          # Generated config files (artifacts)
```

## Key Concepts

### Percona XtraDB Cluster (PXC)
- **Galera-based clustering:** Synchronous multi-master replication
- **All nodes are equal:** Can read/write to any node
- **Automatic failover:** Built-in high availability
- **Bootstrap node:** First node (pxc1) initializes the cluster
- **Joining nodes:** Subsequent nodes (pxc2, pxc3) join via `CLUSTER_JOIN` env var

### Ansible Role Architecture
- **Single role design:** The `containers` role handles everything
- **No dependencies:** Self-contained role with no external role dependencies
- **Task includes:** Main task file includes other task files in sequence
- **Variable precedence:** defaults < vars < extra-vars (command line)

### Docker Configuration
- **Docker network:** All containers connect to shared "mysql" network
- **Container naming:** pxc1, pxc2, pxc3, etc.
- **Hostname resolution:** Docker network provides DNS for container names
- **No persistent volumes:** Containers are ephemeral (development setup)
- **Security:** Empty root password (development only, NOT production-safe)

## Critical Files Reference

### site.yml (Main Playbook)
- **Purpose:** Entry point for playbook execution
- **Hosts:** Targets `all` (localhost in this case)
- **Roles:** Applies the `containers` role
- **Facts:** Gathering disabled for faster execution
- **Location:** `/home/user/ansible-pxc-docker/site.yml`

### hosts (Inventory)
- **Purpose:** Defines target hosts for Ansible
- **Connection:** Uses `ansible_connection=local` (no SSH)
- **Python:** Explicitly sets Python interpreter to `/usr/bin/python`
- **Location:** `/home/user/ansible-pxc-docker/hosts`

### roles/containers/defaults/main.yml (Default Variables)
```yaml
servers: 3                          # Number of PXC nodes (1-N)
mysql_image_version: 5.7           # Docker image version (5.7 or 8.0)
mysql_server_id: 1                 # Base server ID for replication
mysql_expire_logs_days: "1"        # Binary log retention period
mysql_binlog_format: "ROW"         # Replication format (ROW recommended)
mysql_max_binlog_size: "10M"       # Max size before log rotation
```

### roles/containers/vars/main.yml (Fixed Variables)
```yaml
docker_network_name: mysql         # Docker network name (hardcoded)
```

### roles/containers/tasks/main.yml (Task Orchestration)
**Execution order:**
1. `images.yml` - Pull Percona Docker images
2. `network.yml` - Create Docker network
3. `container.yml` - Create bootstrap node (pxc1)
4. `set-replication.yml` - Create joining nodes (pxc2, pxc3, etc.)

Note: `mysql-conf.yml` is commented out (configuration mounting disabled)

## Development Workflow

### Prerequisites Check
```bash
# Verify Docker is running
docker ps

# Verify Ansible is installed
ansible --version  # Should be 2.3.2.0+

# Verify Python modules
python -c "import docker; import yaml"
```

### Running the Playbook

**Basic execution (3-node cluster, PXC 5.7):**
```bash
ansible-playbook site.yml -i hosts
```

**Custom configurations:**
```bash
# 5-node cluster with PXC 5.7
ansible-playbook site.yml -i hosts --extra-vars "servers=5 mysql_image_version=5.7"

# Single node for testing
ansible-playbook site.yml -i hosts --extra-vars "servers=1"

# 3-node cluster with MySQL 8.0
ansible-playbook site.yml -i hosts --extra-vars "servers=3 mysql_image_version=8.0"
```

### Verifying Deployment

```bash
# Check running containers
docker ps

# Check cluster status (Galera variables)
docker exec pxc1 mysql -uroot -e "SHOW STATUS LIKE 'wsrep%';"

# Check cluster size (should match number of nodes)
docker exec pxc1 mysql -uroot -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

# Connect to MySQL
docker exec -it pxc1 mysql -uroot

# View container logs
docker logs pxc1
docker logs pxc2
```

### Cleanup

```bash
# Stop and remove all containers
docker stop $(docker ps -a -q --filter "name=pxc")
docker rm $(docker ps -a -q --filter "name=pxc")

# Remove network
docker network rm mysql

# Remove images (optional)
docker rmi percona/percona-xtradb-cluster:5.7
```

## Code Conventions and Patterns

### Ansible Best Practices Used
1. **Role-based organization:** Single-purpose role with clear structure
2. **Task separation:** Logical grouping of tasks in separate files
3. **Variable abstraction:** Sensible defaults with override capability
4. **Template usage:** Jinja2 templates for configuration files
5. **Idempotency:** Docker modules handle state management

### Docker Module Usage
- **docker_image:** Pull images from Docker Hub
- **docker_network:** Create/manage Docker networks
- **docker_container:** Create and configure containers

### Variable Naming Conventions
- **Prefix by component:** `mysql_*` for MySQL-related vars, `docker_*` for Docker
- **Descriptive names:** Clear purpose (e.g., `mysql_expire_logs_days`)
- **String vs int:** Some numeric values are strings for template compatibility

### Task Naming
- **Action-oriented:** Start with verb (Pull, Create, Set, Wait)
- **Specific:** Clearly describe what the task does
- **Consistent:** Follow same pattern across task files

## Important Behavioral Notes

### Known Issues and Limitations

1. **Image version hardcoded:**
   - `images.yml` pulls version 5.7 only
   - Does NOT use `mysql_image_version` variable
   - **Impact:** Cannot pull 8.0 image automatically
   - **Workaround:** Manually pull image or fix the task

2. **No persistent storage:**
   - Containers have no mounted volumes
   - Data is lost when containers are removed
   - **Impact:** Not suitable for persistent development data
   - **Recommendation:** Add volume mounts if data persistence needed

3. **Security configuration:**
   - `MYSQL_ALLOW_EMPTY_PASSWORD=yes` is set
   - Root password is empty
   - **Impact:** Insecure, development-only
   - **Never use in production**

4. **Cluster name hardcoded:**
   - `CLUSTER_NAME: "dani-pxc-docker"` in task files
   - Not configurable via variables
   - **Impact:** Cannot change cluster name without editing tasks

5. **Startup timing:**
   - 40-second pause before creating additional nodes
   - **Reason:** Wait for bootstrap node to initialize
   - **May need adjustment:** Slower systems might need longer wait

6. **MySQL configuration disabled:**
   - `mysql-conf.yml` task is commented out
   - Configuration files are generated but not used
   - **Impact:** Containers use default PXC configuration
   - **Enable if:** Custom MySQL settings are required

### Loop Logic in set-replication.yml
```yaml
with_sequence: start={{2 if servers|int == 3 else servers }} end="{{ servers }}" format=pxc%01x
```
- **Normal case (servers=3):** Creates pxc2, pxc3
- **Edge case (servers=1):** Loop doesn't run (start=1, end=1)
- **Other values:** Creates from pxc{servers} to pxc{servers}
- **Potential bug:** Logic seems incorrect for servers != 3

## Common Tasks for AI Assistants

### Adding New Variables
1. Add to `roles/containers/defaults/main.yml` for user-configurable vars
2. Add to `roles/containers/vars/main.yml` for fixed role vars
3. Use in templates with Jinja2 syntax: `{{ variable_name }}`
4. Pass to tasks using `env:` for Docker environment variables

### Modifying Container Configuration
1. Environment variables: Edit `env:` section in container creation tasks
2. MySQL configuration: Uncomment `mysql-conf.yml` and mount configs
3. Network settings: Modify `docker_network_name` in vars/main.yml
4. Image version: Fix `images.yml` to use `{{ mysql_image_version }}`

### Adding New Task Files
1. Create task file in `roles/containers/tasks/`
2. Include in `roles/containers/tasks/main.yml` using `- include: taskfile.yml`
3. Place in correct execution order
4. Follow naming convention: descriptive, action-oriented

### Testing Changes
1. Use verbose mode: `ansible-playbook site.yml -i hosts -vv`
2. Use check mode: `ansible-playbook site.yml -i hosts --check` (limited Docker support)
3. Test with minimal setup: `--extra-vars "servers=1"`
4. Verify idempotency: Run playbook twice, second run should show no changes

### Debugging
```bash
# Ansible debugging
ansible-playbook site.yml -i hosts -vvv  # Very verbose

# Docker debugging
docker logs pxc1                          # View container logs
docker inspect pxc1                       # Inspect container config
docker network inspect mysql              # Check network configuration
docker exec pxc1 env                      # View environment variables

# MySQL debugging
docker exec pxc1 mysql -uroot -e "SELECT @@hostname, @@server_id;"
docker exec pxc1 cat /etc/mysql/my.cnf   # View MySQL config
```

## Environment Variables Reference

### Docker Container Environment Variables
- **MYSQL_ALLOW_EMPTY_PASSWORD:** Set to "yes" for passwordless root
- **CLUSTER_NAME:** Galera cluster identifier (currently "dani-pxc-docker")
- **CLUSTER_JOIN:** Bootstrap node hostname for joining nodes (set to "pxc1")

These are specific to the `percona/percona-xtradb-cluster` Docker image.

## Git Branch Information

**Current branch:** `claude/claude-md-mi33m5hqb2eyjw31-01Qj71QYZg4Jae6HYT3gTCkU`
**Main branch:** Not specified in current configuration
**Status:** Clean working directory

### Git Workflow for AI Assistants
1. Develop on the `claude/claude-md-*` branch
2. Commit with clear, descriptive messages
3. Push to remote: `git push -u origin <branch-name>`
4. Never push to other branches without permission

## Testing Strategy

### Manual Testing Checklist
- [ ] Playbook executes without errors
- [ ] Correct number of containers created
- [ ] All containers in "Up" status
- [ ] Docker network exists and containers are connected
- [ ] MySQL accessible on all nodes (empty password)
- [ ] Cluster status shows all nodes (`wsrep_cluster_size`)
- [ ] Nodes are synced (`wsrep_local_state_comment = Synced`)
- [ ] Can write to one node and read from another

### Test Commands
```bash
# Verify cluster topology
docker exec pxc1 mysql -uroot -e "SHOW STATUS LIKE 'wsrep_cluster_size';"

# Test replication
docker exec pxc1 mysql -uroot -e "CREATE DATABASE test_replication;"
docker exec pxc2 mysql -uroot -e "SHOW DATABASES LIKE 'test%';"

# Check node status
docker exec pxc1 mysql -uroot -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
```

## Troubleshooting Guide

### Common Issues

**Containers fail to start:**
- Check Docker daemon is running: `docker ps`
- Check image exists: `docker images | grep percona`
- View container logs: `docker logs pxc1`

**Nodes won't join cluster:**
- Verify network exists: `docker network ls`
- Check bootstrap node is running: `docker ps | grep pxc1`
- Increase wait time in `set-replication.yml` (40 seconds default)
- Check cluster logs: `docker logs pxc1 | grep -i cluster`

**Playbook fails with "docker_py" error:**
- Install Python module: `pip install docker-py`
- Verify installation: `python -c "import docker"`

**Playbook changes don't take effect:**
- Ansible caches fact gathering: Use `gather_facts: False` (already set)
- Docker containers are running: Stop/remove before re-running playbook
- Image is cached: Pull latest image manually or use `pull: yes`

## File References by Task

| Task | Reads | Writes/Creates | Dependencies |
|------|-------|----------------|--------------|
| images.yml | - | Docker images (pulled) | Docker Hub |
| network.yml | - | Docker network "mysql" | - |
| container.yml | defaults/main.yml, vars/main.yml | Container "pxc1" | Docker image, network |
| set-replication.yml | defaults/main.yml, vars/main.yml | Containers "pxc2"+ | Container "pxc1" running |
| mysql-conf.yml (disabled) | templates/my.cnf.j2 | my.cnf.mysql[1-N] | - |

## Security Considerations

### Current Security Posture
- **Root password:** Empty (CRITICAL: Development only!)
- **Network exposure:** Docker network only (not exposed to host by default)
- **Authentication:** No authentication required
- **Encryption:** No SSL/TLS configured
- **Audit logging:** Not enabled

### If Adapting for Production
1. Set proper MySQL root password via `MYSQL_ROOT_PASSWORD`
2. Create application users with limited privileges
3. Enable SSL/TLS for client connections
4. Configure Galera encryption (SST/IST)
5. Use Docker secrets for sensitive data
6. Add persistent volumes with proper permissions
7. Configure firewall rules
8. Enable audit logging
9. Regular security updates for images

## Performance Considerations

- **Resource limits:** No CPU/memory limits set on containers
- **Binary logs:** Set to 10M rotation, 1-day retention (low for development)
- **Network:** Docker bridge network (acceptable for local development)
- **Storage:** No volume mounts (using container storage, slower)

## Advanced Usage

### Extending the Playbook

**Add persistent volumes:**
```yaml
# In container.yml and set-replication.yml
volumes:
  - /path/on/host:/var/lib/mysql
```

**Enable custom MySQL configuration:**
1. Uncomment `- include: mysql-conf.yml` in tasks/main.yml
2. Modify templates/my.cnf.j2 as needed
3. Add volume mount in container tasks:
```yaml
volumes:
  - ./my.cnf.mysql{{ item }}:/etc/mysql/my.cnf
```

**Add healthchecks:**
```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s
  timeout: 5s
  retries: 3
```

### Variable Expansion Examples

**Using extra-vars:**
```bash
ansible-playbook site.yml -i hosts \
  --extra-vars "servers=5 mysql_image_version=8.0 mysql_binlog_format=MIXED"
```

**Using vars file:**
```bash
# Create custom-vars.yml
---
servers: 7
mysql_image_version: 8.0

# Run playbook
ansible-playbook site.yml -i hosts --extra-vars "@custom-vars.yml"
```

## Documentation Links

- **Percona XtraDB Cluster:** https://www.percona.com/doc/percona-xtradb-cluster/
- **PXC Docker Image:** https://hub.docker.com/r/percona/percona-xtradb-cluster
- **Ansible Docker Modules:** https://docs.ansible.com/ansible/latest/collections/community/docker/
- **Galera Cluster:** https://galeracluster.com/library/documentation/

## Quick Reference Commands

```bash
# Deploy cluster
ansible-playbook site.yml -i hosts

# Deploy with custom settings
ansible-playbook site.yml -i hosts --extra-vars "servers=5 mysql_image_version=5.7"

# Check cluster status
docker exec pxc1 mysql -uroot -e "SHOW STATUS LIKE 'wsrep%';"

# Connect to MySQL
docker exec -it pxc1 mysql -uroot

# View logs
docker logs pxc1

# Cleanup everything
docker stop $(docker ps -q --filter "name=pxc") && \
docker rm $(docker ps -aq --filter "name=pxc") && \
docker network rm mysql

# Rebuild cluster from scratch
docker stop $(docker ps -q --filter "name=pxc"); \
docker rm $(docker ps -aq --filter "name=pxc"); \
docker network rm mysql; \
ansible-playbook site.yml -i hosts
```

---

## Summary for AI Assistants

This is a **simple, focused Ansible project** for local PXC development:
- Single role (`containers`) handles entire deployment
- Creates Docker containers running Percona XtraDB Cluster
- Configurable cluster size (default 3 nodes)
- Development-focused (insecure, ephemeral storage)
- macOS target platform

**When making changes:**
1. Respect the single-role architecture
2. Maintain backward compatibility with default variables
3. Test with different `servers` values (1, 3, 5)
4. Verify Galera cluster forms correctly
5. Update this CLAUDE.md file if adding features

**Common modification requests:**
- Add persistent storage: Uncomment mysql-conf.yml, add volume mounts
- Change cluster name: Make `CLUSTER_NAME` a variable
- Fix image version: Update images.yml to use `{{ mysql_image_version }}`
- Add security: Implement password via environment variables
- Production-ready: Add healthchecks, resource limits, logging

Always test changes with minimal setup first (`servers=1`) before testing multi-node clusters.
