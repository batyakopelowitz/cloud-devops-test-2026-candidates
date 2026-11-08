## Problem 3 – Linux pipeline agent: "No space left on device"

### Cause / risk
- The root disk is full. Docker uses 76G (old images, build cache and stopped containers,
  every build adds new layers and nothing cleans them). Old pipeline workspaces in
  `_work` use 12G. About 7G is not explained by the output (logs, temp files, or
  deleted files still held open by a process).
- ext4 reserves ~5% of the disk for root, so the agent (non-root user) fails first.
- Risk: all builds and deployments are blocked, including urgent fixes.

### Investigation commands
```bash
df -h /                                         # disk usage
df -i /                                         # inodes ("no space" can also mean no free inodes)
sudo du -xh --max-depth=1 / | sort -h           # find the missing ~7G across the whole disk
sudo journalctl --disk-usage                    # size of system logs
sudo lsof +L1                                   # deleted files still held open by a process
docker system df                                # images / containers / volumes / build cache
sudo du -h --max-depth=1 /opt/ado-agent/_work | sort -h   # old pipeline workspaces
```

### What to check before deleting anything
- This VM is a build agent: it should hold no unique data. Everything on it can be rebuilt or pulled again.
- No pipeline job is running on the agent (check the agent pool in Azure DevOps, `docker ps`).
- Images we delete are already pushed to Artifact Registry or can be rebuilt.
- Docker volumes are not pruned without checking them (`docker volume ls`).
- Never delete files under `/var/lib/docker` manually, use Docker commands only.

### Safe immediate recovery
1. Disable the agent in the Azure DevOps agent pool so it takes no new jobs, wait for the current job to finish.
2. Free Docker space:
   ```bash
   docker container prune -f
   docker builder prune -a -f
   docker image prune -a -f --filter "until=72h"
   ```
3. Remove old pipeline workspaces (only while the agent is disabled):
   `sudo rm -rf /opt/ado-agent/_work/[0-9]*`
4. Shrink logs: `sudo journalctl --vacuum-size=500M`
5. If the lsof check showed deleted open files, restart that process.
6. Check `df -h /` again, re-enable the agent and run one test pipeline.

### Prevention plan
- Daily cleanup job at night, when no builds run:
  ```
  # /etc/cron.d/build-agent-cleanup
  0 3 * * * root (docker container prune -f && docker image prune -a -f --filter "until=72h" && docker builder prune -a -f --filter "until=72h") >> /var/log/build-agent-cleanup.log 2>&1
  ```
- Enable the agent pool maintenance job in Azure DevOps Server (Collection Settings > Agent pools > Settings) to delete old working directories.
- Limit journald size (`SystemMaxUse=1G` in `/etc/systemd/journald.conf`), then `sudo systemctl restart systemd-journald`.
- Cloud Monitoring alert when disk usage is above 80% (Ops Agent on the VM).
- Optional: move `/var/lib/docker` to a separate disk, so Docker cannot fill the OS disk.

### Verification
- Compare `df -h /` and `docker system df` before and after cleanup.
- A test pipeline (build + push) succeeds.
- `/var/log/build-agent-cleanup.log` shows the cleanup job runs every night.
- Test the alert: create a large temporary file (`fallocate`) to pass 80%, confirm the alert, delete the file.

### Assumptions and tradeoffs
- Assumptions: one agent on this VM, ext4 root disk, the agent runs as a non-root user, no builds run at 03:00.
- Tradeoff: deleting build cache and images makes the next builds slower (layers are rebuilt or pulled again). Acceptable for a build agent.

*Proposed only - not tested on a real agent VM.*