---
name: docker-troubleshoot
description: Troubleshoot Docker Compose startup errors. Use when encountering network conflicts, container name collisions, orphaned containers, or other Docker setup issues.
---

# Docker Troubleshooting

## Common Issues and Fixes

### 1. Network Subnet Conflict

**Error message:**
```
failed to create network: Error response from daemon: invalid pool request: Pool overlaps with other one on this address space
```

**Cause:** The subnet defined in docker-compose conflicts with an existing Docker network.

**Fix:**
1. Open `docker-compose.dev.yml`
2. Find the `networks` section at the bottom
3. Change the subnet to an unused range (e.g., `172.28.0.0/16` instead of `172.20.0.0/16`)
4. Update all `ipv4_address` values for each service to match the new subnet

**Alternative:** Clean up unused networks with:
```bash
docker network prune
```

### 2. Container Name Already in Use

**Error message:**
```
Error response from daemon: Conflict. The container name "/container-name" is already in use
```

**Cause:** Orphaned containers from a previous run weren't cleaned up properly.

**Fix:** Remove the conflicting containers:
```bash
docker rm -f postgres-dev migrate-dev server-dev web-dev nginx-dev
```

Or remove all stopped containers:
```bash
docker container prune
```

### 3. Full Cleanup

When Docker state is severely corrupted:

```bash
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker network rm caliber-fleet-management_app_network
docker system prune -f
```

## Diagnostic Commands

```bash
docker ps -a                    # List all containers
docker network ls               # List all networks
docker network inspect <name>   # Inspect network subnet
```
