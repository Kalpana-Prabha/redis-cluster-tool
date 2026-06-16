# Redis Cluster Lifecycle Tool

A CLI tool that wraps Ansible to provision, operate, and perform a zero-downtime rolling upgrade of a 6-node Redis Cluster (3 masters + 3 replicas) running inside Docker containers that simulate real servers with SSH access.

## Prerequisites

- Docker Engine (or Podman)
- Ansible 2.14+
- Python 3.x

redis-tool runs a prerequisite check before every command and tells you exactly what's missing and how to install it.

## Quick Start

### 1. Start the infrastructure

cd infra
docker compose up -d

This brings up 6 containers (redis-node-1 .. redis-node-6) on a static 10.10.0.0/24 network, each running an SSH server.

### 2. Set up SSH key access

for i in 1 2 3 4 5 6; do
  docker exec redis-node-$i bash -c "echo '$(cat ~/.ssh/redis_cluster_key.pub)' > /home/ansible/.ssh/authorized_keys && chmod 600 /home/ansible/.ssh/authorized_keys && chown -R ansible:ansible /home/ansible/.ssh"
done

### 3. Run redis-tool commands

./redis-tool provision --version 7.0.15
./redis-tool data seed --keys 1000
./redis-tool data verify
./redis-tool status
./redis-tool upgrade --target-version 7.2.6 --strategy rolling
./redis-tool verify --full

## Architecture

Your Machine (Ansible Control Node)
        |
        | SSH (host ports 2221-2226 -> container port 22)
        |
Docker network 10.10.0.0/24
  redis-node-1  10.10.0.11  (master)
  redis-node-2  10.10.0.12  (master)
  redis-node-3  10.10.0.13  (master)
  redis-node-4  10.10.0.14  (replica)
  redis-node-5  10.10.0.15  (replica)
  redis-node-6  10.10.0.16  (replica)

On initial provision, nodes 1-3 are masters (slots 0-5460, 5461-10922, 10923-16383) and nodes 4-6 are their replicas.

## Rolling Upgrade Strategy

./redis-tool upgrade --target-version 7.2.6 --strategy rolling performs:

1. Pre-flight checks - confirm cluster_state:ok, all nodes reachable, current version differs from target, and run a data-integrity baseline.
2. Upgrade replicas first, one at a time - stop Redis, compile/install the new version from source, start Redis with the same config, wait for it to rejoin and resync, then verify cluster_state:ok before moving to the next node.
3. Upgrade masters one at a time, with failover - trigger CLUSTER FAILOVER on the master's replica (already on the new version) so it is promoted to master, wait for the failover to complete, then stop/upgrade/restart the old master, which rejoins as a replica. Verify cluster_state:ok before continuing.
4. Post-upgrade verification - re-run data verify (all 1000 keys must still match), run status (all nodes must report the new version), and print UPGRADE COMPLETE.

This order guarantees that at every point in time each shard has at least one node serving traffic, so the cluster never drops below cluster_state:ok and clients see zero downtime.

If any step fails, the playbook stops immediately, reports which node/step failed, and leaves the cluster as-is (no automatic rollback).

## Project Structure

redis-cluster-tool/
  redis-tool                     - CLI entrypoint (Python)
  ansible/
    ansible.cfg
    inventory/
      hosts.ini
    playbooks/
      provision.yml          - Phase 1: install Redis, configure & form cluster
      seed.yml               - Phase 2: seed 1000 deterministic keys
      verify.yml             - Phase 2: verify data integrity
      status.yml             - Phase 3: cluster topology / status
      upgrade.yml            - Phase 4: zero-downtime rolling upgrade
      full_verify.yml        - Phase 5: full post-upgrade health check
    roles/
      redis/
        tasks/main.yml
        handlers/main.yml
        templates/redis.conf.j2
        defaults/main.yml
  infra/
    Containerfile             - Ubuntu 22.04 + SSH base image
    compose.yml               - 6-node cluster infra (Docker/Podman compose)
  README.md
  output/
    provision_output.txt
    data_seed_output.txt
    status_output.txt
    upgrade_output.txt
    verify_output.txt

## Trade-offs and Assumptions

- Redis is compiled from source on each node to guarantee the exact version specified (7.0.15 / 7.2.6), rather than relying on distro packages.
- nodes.conf format differences between Redis 7.0.x and 7.2.x (extra tls-port/shard-id fields) are patched out before starting the upgraded binary, so cluster topology survives the version change.
- SSH key-based access from the host to each container is set up after the containers start, using a dedicated keypair (redis_cluster_key).
- The Ansible inventory connects to each container via localhost on forwarded ports 2221-2226 (mapped to container port 22).
- data seed / data verify use deterministic values (SHA256 of the key name), so re-running seed is idempotent and verify can independently recompute expected values.
- The key-count step in seed.yml dynamically discovers the current master nodes via CLUSTER NODES, so the reported distribution stays correct even after a rolling upgrade changes which nodes are masters.

## Known Limitations

- Containers have no persistent volumes, so a container restart loses all Redis data and cluster state (the cluster must be re-provisioned).
- Rolling upgrade takes roughly 15-20 minutes total because each node compiles Redis from source.
- Stretch goals S1 (scale out), S2 (scale in), S3 (rollback), and S5 (structured logging) are not implemented. S4 (Idempotency) is implemented for both provision and upgrade commands.
