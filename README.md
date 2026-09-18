# Elasticsearch NFS Snapshot Repository Verification Troubleshooting Guide

## Layout

Runbook repository: the full NFS snapshot-repo guide stays in this README. Related lab: [elastic_stack_on_hyper-v](https://github.com/nwlterry/elastic_stack_on_hyper-v).


**Date:** July 2026  
**Context:** Elasticsearch 8.18.x cluster with data tiers (hot/cold) using shared filesystem (`fs`) snapshot repository backed by NFS.

---

## Cluster Configuration

| Node Type          | Count | Roles                  | NFS Mount Status          | Notes |
|--------------------|-------|------------------------|---------------------------|-------|
| Master             | 3     | master + data_content  | Mounted + OS access OK    | Assumed `path.repo` configured |
| Data-Hot           | 1     | data_hot               | **Not mounted**           | Missing configuration |
| Data-Cold          | 1     | data_cold              | **Not mounted**           | Missing configuration |

**Total nodes:** 5 (minimum for this tiered setup).  
**Repository:** `fs` type pointing to an NFS share.  
**Symptom:** "Connection issue" / verification failure when creating or verifying the backup repository (typically `RepositoryVerificationException` or "store location is not accessible on the node").

---

## Root Cause (Primary)

Official Elasticsearch requirement for shared file system repositories:

> "To register a shared file system repository, first **mount the file system to the same location on all master and data nodes**. Then add the file system’s path or parent directory to the `path.repo` setting in `elasticsearch.yml` for **each master and data node**."

**Why it fails here:**
- The `data_hot` and `data_cold` nodes are **data nodes**.
- During snapshots, **data nodes stream shard files directly to the repository**.
- The `_verify` API and internal registration/verification logic check accessibility from **all master-eligible and data nodes**.
- Because the hot and cold nodes lack the NFS mount (and likely lack `path.repo` in their `elasticsearch.yml`), verification fails — even if the 3 master nodes can access it perfectly.
- "Connection issue" is often a symptom of `AccessDeniedException`, `NoSuchFileException`, or `RepositoryVerificationException` reported against one of the data nodes (or sometimes masters if `path.repo` is incomplete).

**Secondary common causes (NFS-specific):**
- Numeric UID/GID mismatch for the `elasticsearch` user across nodes (NFS matches by numeric ID, not username).
- `path.repo` not set or not covering the repository `location` on all nodes.
- NFS export restrictions, firewall rules, or mount instability on hot/cold nodes.
- `elasticsearch.yml` changes not applied (requires node restart).
- Permission/ownership issues on the NFS share.

---

## Step-by-Step Troubleshooting & Resolution

### Step 1: Gather Diagnostics (Run on ALL Nodes)

Execute these commands (or via centralized logging/Ansible) and note which node fails.

```bash
# 1. Cluster node overview (run from any node with API access)
curl -s -k -u elastic:<password> 'https://<any-master>:9200/_cat/nodes?v&h=name,ip,roles,master'

# 2. Check path.repo settings (if cluster responding)
curl -s -k -u elastic:<password> 'https://<any-master>:9200/_nodes/settings?pretty' | grep -A 10 '"path.repo"'

# 3. NFS mount status & accessibility
df -h | grep -E 'nfs|NFS'
mount | grep -i nfs
ls -ld /mnt/es-backups          # <-- CHANGE TO YOUR ACTUAL PARENT PATH
ls -l /mnt/es-backups

# 4. Elasticsearch user identity (MUST BE CONSISTENT NUMERIC UID/GID)
id elasticsearch                # or: ps aux | grep elasticsearch | head -1

# 5. Test write access AS the elasticsearch user (critical test)
su - elasticsearch -c "
  touch /mnt/es-backups/test-write-$(hostname).txt && \
  ls -l /mnt/es-backups/test-write-$(hostname).txt && \
  rm /mnt/es-backups/test-write-$(hostname).txt && \
  echo 'SUCCESS: Write test passed on $(hostname)'
" || echo "FAILED on $(hostname)"

# 6. Recent snapshot/repository errors in logs
tail -n 300 /var/log/elasticsearch/elasticsearch.log | grep -E 'snapshot|repository|VerifyNodeRepositoryAction|RepositoryVerificationException|AccessDenied' || \
journalctl -u elasticsearch --since "2 hours ago" | grep -E 'snapshot|repo|verification'
```

**Action:** Share the output (especially the exact error message and which node it references) for precise follow-up.

### Step 2: Confirm Masters Are Correctly Configured

On each of the 3 master nodes:

1. Verify stable NFS mount at the **same path** used on all nodes.
2. Check `elasticsearch.yml`:

```yaml
# /etc/elasticsearch/elasticsearch.yml
path.repo:
  - /mnt/es-backups          # Parent directory — repo "location" must be under this
```

3. If recently changed, a restart would have been needed (but since they "already have access", this is likely OK).

### Step 3: Fix the Data-Hot and Data-Cold Nodes (Main Fix)

**On BOTH data-hot and data-cold nodes, perform these steps:**

#### 3.1 Mount the NFS Share (identical path)

```bash
# Create mount point (use the EXACT same path as masters)
sudo mkdir -p /mnt/es-backups

# Add to /etc/fstab for persistence (example for NFSv4.1)
echo "nfs-server.example.com:/export/elastic-snapshots /mnt/es-backups nfs nfsvers=4.1,rsize=1048576,wsize=1048576,hard,intr,_netdev 0 0" | sudo tee -a /etc/fstab

# Mount
sudo mount -a

# Verify
df -h | grep es-backups
ls -ld /mnt/es-backups
```

**Recommendations for mount options:**
- `nfsvers=4.1` or `4.2` (avoid v3 if possible)
- `hard,intr` (not `soft`)
- Large `rsize`/`wsize` (1MB often good for snapshots)
- `_netdev` (mount after network)

#### 3.2 Configure path.repo

Edit `/etc/elasticsearch/elasticsearch.yml`:

```yaml
path.repo:
  - /mnt/es-backups
```

**Note:** The value(s) in `path.repo` are the **allowed parent directories**. Your repository `location` in the PUT API must reside under one of them (or be an exact match).

#### 3.3 Apply Changes — Rolling Restart

Because `path.repo` is a node startup setting, you **must restart** the Elasticsearch process on data-hot and data-cold nodes.

**Recommended approach (minimal disruption):**

```bash
# Temporarily disable shard allocation (run once from master)
curl -k -u elastic:<password> -X PUT 'https://<master>:9200/_cluster/settings?pretty' -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}'
```

Then restart one data node at a time:

```bash
# On data-hot node
sudo systemctl restart elasticsearch
# Wait for it to join cluster and recover
curl -k -u elastic:<password> 'https://<master>:9200/_cat/health?v'
curl -k -u elastic:<password> 'https://<master>:9200/_cat/recovery?v'

# Repeat for data-cold node
```

Re-enable allocation when done:

```bash
curl -k -u elastic:<password> -X PUT 'https://<master>:9200/_cluster/settings?pretty' -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}'
```

### Step 4: NFS-Specific Validation (All 5 Nodes)

After mounting the NFS share on the data-hot and data-cold nodes, perform these validations on **all five nodes**.

#### 4.1 Detailed: Diagnose and Fix UID/GID Mismatch (Most Common NFS Gotcha)

**Why this causes "connection" / access issues:**

NFS servers authorize access and set file ownership/permissions based on the **numeric UID and GID** sent by the client kernel, **not** the username string ("elasticsearch"). 

If the `elasticsearch` user has different numeric IDs on different nodes (very common when nodes were provisioned separately or via different methods), then:
- Files written from one node appear owned by "nobody" or an unknown user on other nodes.
- Elasticsearch processes on nodes with the "wrong" UID see `Permission denied` or `AccessDeniedException` when trying to create the verification test file or write snapshot data.
- This often manifests exactly as a verification "connection issue" even though network reachability is fine.

**Step-by-step diagnosis (run on every node):**

```bash
# Check current IDs
id elasticsearch
getent passwd elasticsearch
getent group elasticsearch

# Also check what user Elasticsearch is actually running as
ps aux | grep -E '[e]lasticsearch' | head -3
```

**Expected output example (consistent across all nodes):**
```
uid=1000(elasticsearch) gid=1000(elasticsearch) groups=1000(elasticsearch)
```

**If the numeric uid= or gid= values differ between any nodes → you have a mismatch.**

**Remediation — Choose the appropriate option:**

**Option A: Standardize UID/GID on all Elasticsearch nodes (Recommended for long-term consistency)**

Pick one consistent UID/GID (e.g. `1000:1000` — the most common default). Fix nodes that have different values.

On each affected node:

```bash
# 1. Stop Elasticsearch cleanly
sudo systemctl stop elasticsearch

# 2. Change the user and group IDs (replace 1000 with your chosen consistent value)
sudo usermod -u 1000 elasticsearch
sudo groupmod -g 1000 elasticsearch

# 3. Fix ownership of any LOCAL Elasticsearch files on this node
# (data, logs, config — important so the process can still start)
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch /etc/sysconfig/elasticsearch /usr/share/elasticsearch /var/lib/elasticsearch /var/log/elasticsearch 2>/dev/null || true

# 4. Start Elasticsearch again
sudo systemctl start elasticsearch

# 5. Verify the new IDs
id elasticsearch
```

After fixing all nodes, re-run the write test as the `elasticsearch` user on the NFS path from every node.

**Option B: Fix ownership on the NFS server side (quick workaround if you manage the export)**

On the NFS server:

```bash
# Identify the UID used by most of your ES nodes (or pick one)
# Then chown the entire snapshot directory tree to that numeric UID/GID
sudo chown -R 1000:1000 /export/elastic-snapshots

# Re-export
sudo exportfs -ra
```

Then on the ES nodes, re-mount the NFS share and test again.

You can combine this with `no_root_squash` in `/etc/exports` (use with caution in production) and proper idmapd configuration for name-based mapping, but standardizing numeric UIDs on the client nodes is cleaner.

**Option C: Containerized nodes (OpenShift, Rancher, ECK, Kubernetes)**

If any of your Elasticsearch nodes run in containers/pods:

In the Elasticsearch manifest or nodeSet spec, explicitly set the security context so the container runs with the **same numeric UID** used on your VM nodes:

```yaml
spec:
  nodeSets:
  - name: hot
    podTemplate:
      spec:
        securityContext:
          runAsUser: 1000
          runAsGroup: 1000
          fsGroup: 1000
```

Then perform a rolling restart of the affected nodeSet(s). The pod will now use the matching UID when accessing the NFS volume.

**After any UID/GID change:**

1. Re-run the write test from **all** nodes as the `elasticsearch` user.
2. Re-verify the repository: `POST /_snapshot/<repo>/_verify`
3. Take a small test snapshot to confirm everything works end-to-end.

#### 4.2 Permissions Test (as elasticsearch user)

```bash
su - elasticsearch -c "
  mkdir -p /mnt/es-backups/testdir && \
  touch /mnt/es-backups/testdir/file.txt && \
  ls -l /mnt/es-backups/testdir/ && \
  rm -rf /mnt/es-backups/testdir && \
  echo 'SUCCESS: Full rwx test passed on $(hostname)'
" || echo "FAILED write test on $(hostname) - check UID/GID and NFS permissions"
```

#### 4.3 NFS Server Side Checks (if you control the NFS server)

- Verify export allows the IP addresses (or subnet) of **all five** ES nodes with read-write access.
- Recommended `/etc/exports` line example:
  ```
  /export/elastic-snapshots  172.31.0.0/16(rw,sync,no_root_squash,no_subtree_check)
  ```
- After changes: `exportfs -ra`
- On ES nodes: `showmount -e nfs-server` and re-mount if needed.

#### 4.4 SELinux / AppArmor (RHEL 8)

```bash
getenforce
ausearch -m avc -ts recent | grep -E 'elasticsearch|nfs|mount' | tail -20
```

If you see denials, temporarily test with `setenforce 0`. For permanent fix, adjust context or create a local policy for the mount point and Elasticsearch process. Common quick fix:

```bash
sudo chcon -R -t container_file_t /mnt/es-backups   # or httpd_sys_content_t / var_lib_t depending on policy
```

---

**Note:** The original basic UID check is now expanded above in 4.1. The rest of the troubleshooting flow remains the same.

### Step 5: Re-Verify the Repository

Once all nodes are updated and restarted:

```bash
# Verify existing repository
POST /_snapshot/<your-repo-name>/_verify

# Example response on success:
# {
#   "nodes": {
#     "<node-id>": { "name": "...", "successful": true }
#   }
# }
```

If it still fails, the error will now clearly indicate which node(s) are problematic.

You can also run a **Repository Analysis** for deeper diagnostics:

```bash
POST /_snapshot/<your-repo-name>/_analyze
{
  "blob_count": 10,
  "max_blob_size": "10mb",
  "max_total_data_size": "100mb"
}
```

### Step 6: If Still Failing — Additional Checks

- **Exact error message** — Look for the node name in `RepositoryVerificationException`.
- **Inter-node vs NFS-server connectivity** from hot/cold nodes (ping, `showmount`, `nc -zv nfs-server 2049`).
- **Stale mounts**: `sudo umount -f /mnt/es-backups && sudo mount -a`
- **Containerized environments** (OpenShift, Rancher, ECK): Ensure the NFS volume is mounted in the pod spec for **all** node types (master, hot, cold, etc.), not just a subset. Update the Elasticsearch manifest and trigger a rolling restart of the affected nodeSets.
- **VMware / vSAN specifics**: Check for thin provisioning alignment or storage latency if performance is also an issue (separate from verification).

---

## Long-Term Recommendation: Switch to Object Storage

For a cluster with dedicated **data_hot** and **data_cold** tiers, **NFS is not ideal** for snapshot repositories.

**Strongly recommended migration path:**

Use an **S3-compatible repository** (`type: s3`):

- No filesystem mounts required on any data nodes.
- Only the nodes performing the snapshot need S3 access (or use a proxy).
- Better durability, scalability, and operational simplicity.
- Native support for ILM-managed snapshots and cross-cluster replication.
- On-prem options: **MinIO**, Ceph RGW, or commercial S3 gateways.

Example registration:

```json
PUT _snapshot/s3-backup-repo
{
  "type": "s3",
  "settings": {
    "bucket": "elastic-snapshots",
    "region": "us-east-1",
    "endpoint": "https://minio.internal:9000",
    "access_key": "...",
    "secret_key": "...",
    "compress": true
  }
}
```

This eliminates the entire class of "mount on every node + UID consistency" problems.

NFS can work for small/simple clusters, but with your tiered architecture and existing investment in hot/cold nodes, object storage will be more reliable long-term.

---

## Prevention & Operational Best Practices

1. **Never register an `fs` repository until the path is mounted and `path.repo` is set on ALL master + data nodes.**
2. Use infrastructure-as-code to enforce consistent:
   - Mount points and fstab entries
   - `elasticsearch` user UID/GID
   - `path.repo` setting
3. After any change to mounts or `elasticsearch.yml`, always run `_verify` + a test snapshot/restore.
4. Document the repository path, NFS server, and expected UID.
5. Consider dedicated snapshot lifecycle policies (SLM) only after verification succeeds.
6. Monitor NFS latency and stability — it can become a bottleneck for large snapshots compared to local disk or S3.

---

## Summary / Quick Fix Checklist

- [ ] Mount NFS at **identical path** on data-hot and data-cold nodes.
- [ ] Add `path.repo` to their `elasticsearch.yml`.
- [ ] **Critical:** Verify and fix numeric UID/GID mismatch of the `elasticsearch` user across **all 5 nodes** (see detailed steps in 4.1).
- [ ] Test full read/write access as the `elasticsearch` user from the hot and cold nodes on the NFS path.
- [ ] Perform rolling restart of data-hot and data-cold nodes (with allocation toggle).
- [ ] Re-run `POST /_snapshot/<repo>/_verify` and a test snapshot.
- [ ] (Strongly recommended long-term) Plan migration to S3-compatible object storage repository.

---

**Need more targeted help?**  
Provide:
1. The **exact error message** from the verify attempt.
2. Output of `_cat/nodes` showing node names/roles.
3. The **repository `location`** value you are using.
4. The parent path in `path.repo`.
5. `id elasticsearch` output from a master vs a data node.

This guide is based on official Elastic documentation and common production issues with NFS `fs` repositories in tiered clusters.

---

*Document version: 1.1 | Added detailed UID/GID mismatch diagnosis and remediation steps (July 2026) | For internal use / Terry's Elasticsearch environment*

---

See [GROUP.md](GROUP.md) for sibling repositories. Catalog: https://github.com/nwlterry/nwlterry
