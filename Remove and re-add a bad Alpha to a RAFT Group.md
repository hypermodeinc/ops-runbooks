# Remove and re-add a bad Alpha to a RAFT Group

**Revision date**: 01/29/2025

### Use Cases for This Runbook

This procedure is needed when one or more alpha pods of a HA cluster go out-of-sync or enter a CrashLoop . 

**Symptoms**:

- **Queries return inconsistent results**: In some rare scenarios, one of the alphas could return inconsistent query results arising due to variety of issues such as index corruption requiring a rebuild.
- **Alpha is in OOM Loop**: Unoptimised queries could drastically affect the memory usage  resulting in an alpha terminating  due to memory depletion. In scenarios where multiple such queries are issued, the alpha could enter an OOM Loop due to them being replayed from the Write Ahead Log (WAL). The need here is to then clear the WAL which results in a loss of the RAFT state, and thus requiring a complete rebuild.

## Overview of the procedure:

1. Tell the cluster (the zero leader) to remove the bad alpha using it’s IDs (these IDs are obtained from the  **/state** endpoint of Zero Leader )
2. Delete the ‘p', ‘w’, ‘t’ directories to inactivate the bad alpha
3. Restart the alpha pod via kubectl delete, which will cause an alpha process to be restarted and re-join - and receive a fresh data via a snapshot operation from the leader Alpha 

## Detailed Steps:

Below steps assume **zerostatefulset-<ldr>** and **alphastatefulset-<ldr>** are the RAFT leaders while `alphastatefulset-x` is the bad Alpha and it needs to be  removed and re-added to the RAFT group.

1. 

### 1.  **Set the NS variable to point to the faulty cluster**

```yaml
          export NS=<namespace>
example-  export NS=cluster-0x12345
```

### **2. Find the leader Alpha and Zero  in the cluster by checking the state endpoint of any Zero:**

```yaml
kubectl exec -it zerostatefulset-0 -n $NS -- bash
```

```yaml
curl -s localhost:6080/state | jq '{alphas: .groups."1".members, removed: .removed, zeros: .zeros}'
```

Response would be like below. Note that this is for non-HA Cluster. In case of HA, you will see multiple Alphas and zeros

```yaml
{
  "alphas": {
    "1": {
      "**id**": "1",
      "**groupId**": 1,
      "addr": "alphastatefulset-0.dgraph-alpha-service.cluster-0x82b0663f5.svc.cluster.local:7080",
      "**leader**": true,
      "amDead": false,
      "lastUpdate": "1738179285",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    }
  },
  "removed": [],
  "zeros": {
    "1": {
      "**id**": "1",
      "**groupId**": 0,
      "addr": "zerostatefulset-0.dgraph-zero-service.cluster-0x82b0663f5.svc.cluster.local:5080",
      "**leader**": true,
      "amDead": false,
      "lastUpdate": "0",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    }
  }
}
```

### 3. Get the ID, GroupID for the bad Alpha

From the  curl response from step 2, note the following:

- Take a note of which is **leader** alpha and leader zero
- Note the ‘**ID**’ and ‘**GroupID**’ of the **bad alpha** <`alphastatefulset-x`>
- value of “**leader**” attribute for the bad alpha.

### 4. Remove the bad Alpha pod from the RAFT group

Login to the **leader** Zero ****and remove the **Bad Alpha** `alphastatefulset-x` in question. You will use RAFT_ID and GROUP_ID from **Step 3.**

```yaml
kubectl exec -it zerostatefulset-<ldr> -n $NS -- bash
```

```yaml
RAFT_ID=<x>
GROUP_ID=<y>
curl -s "localhost:6080/removeNode?id=$RAFT_ID&group=$GROUP_ID"
```

**Output should be:** `Removed node with group: 1, idx: < the id of the bad alpha >`

**Repeat Step 2  to confirm that the bad Alpha**

- `alphastatefulset-x`  is removed and no longer part of the Alpha group.
- `alphastatefulset-x`  should show up in the **Removed** array of the  response.

If “**leader**” attribute is **true** for the bad alpha,  it will also force Dgraph to elect a new leader.  

**Repeat Step 2** and verify that leader is **now changed** and the bad alpha is no longer the leader by verifying that  “**leader**” attribute is “**false**”.

### 5. **Find the PVC attached to the bad alpha** and rename the p, w, t directories

1. Find the pvc associated with the bad Alpha

```yaml
kubectl describe -n $NS pod <podName> | grep ClaimName

Example- 
% kubectl describe -n $NS pod alphastatefulset-0  | grep ClaimName
    ClaimName:  alphapvc-alphastatefulset-0
```

b.   Find the **volume**  for  the   PVC from 5.a

```yaml
kubectl describe pvc <pvc-name> -n $NS | grep Volume:

Example -
% kubectl describe pvc alphapvc-alphastatefulset-0 -n $NS | grep Volume:
Volume:        pvc-cf76e9a1-e881-4192-90a4-4d81a1d1e2d1
```

c. Find the node name on which the bad Alpha is running and login to the node either by using node-shell or the Lens app

```bash
kubectl get pods -owide -n $NS -- Should print the Node for each pod. 
kubectl node-shell  <node-name>
```

d.  Rename p, w, t on the volume

- Find the matching volume

```bash
lsblk -a | grep <dirNameMatchingPVC>   --- (from Step 5b)
cd <Mount-Point-from-Above>  -- Mount Point is in the response from above command
```

Response should be like 

```bash
lsblk -a | grep pvc-ddd05354-f189-4dee-9cb9-08fcbbcc23c2
nvme3n1       259:5    0   32G  0 disk /var/lib/kubelet/pods/8759a6c0-bf10-4910-886d-438bc0df6866/volumes/kubernetes.io~csi/pvc-ddd05354-f189-4dee-9cb9-08fcbbcc23c2/mount
```

- Cd to the above  mount point  and rename **p/t/w**

```bash
 cd <Mount-Point-from-Above>
 
 Example - cd /var/lib/kubelet/pods/8759a6c0-bf10-4910-886d-438bc0df6866/volumes/kubernetes.io~csi/pvc-ddd05354-f189-4dee-9cb9-08fcbbcc23c2/mount
 mv p p_$(date +%Y-%m-%d) ; mv w w_$(date +%Y-%m-%d) ; mv t t_$(date +%Y-%m-%d)
```

### 6. Delete the Bad Alpha Pod

```
kubectl delete pod <pod-name> -n $NS --force
```

Monitor pod restart with **`kubectl get pods -n $NS -w`** You should notice that the bad alpha should be terminating and restarting.

### 7. Confirm that Snapshot Steam has started from Alpha Leader

Once the status is ‘Running’ and ‘READY’ is ‘0/1’, check the  **Alpha Leader** -logs to confirm if a snapshot stream has started. This might take few minutes.

```bash
kubectl log alphastatefulset-ldr  -n $NS -f | grep -i "Snapshot Streaming"
```

<aside>
ℹ️ snapshot.go:294] Got StreamSnapshot request: context:<id:5 group:1 addr:"alphastatefulset-2.dgraph-alpha-service.cluster-0x1723f2c0.svc.cluster.local:7080" > index:4946613 read_ts:28821222

log.go:34] Sending Snapshot Streaming about 78 GiB of uncompressed data (37 GiB on disk)

</aside>

Once the snapshot stream completes, the following can be seen in Leader Alpha logs:

		snapshot.go:254] Streaming done. Waiting for ACK...

		snapshot.go:259] Received ACK with done: true

		snapshot.go:300] Stream snapshot: OK

		draft.go:137] Operation completed with id: opSnapshot

### 8.  Verify that `alphastatefulset-x` is re-added back to the cluster with a new (incremented) Raft ID and the pod should be READY 1/1 state.

Repeat step 2 to verify that `alphastatefulset-x` is back in the alpha group. Also run `kubectl get pods -n $NS` and make sure READY column shows 1/1 for `alphastatefulset-x`

## Possible problems

1. CrashLoopBackoff
    1. If the alpha restarts with a corrupt p directory in place (or another issue) Dgraph will not come up fully, and specifically will not respond with an HTTP 200 response to the K8s probe (which is currently the /graphql) endpoint, causing K8s to endlessly terminate and restart the pod. The restrarts prevent executing commands on the pod, and particularly prevent removing a bad p directory causing the issue.
    2. Use [Running node-shell](https://www.notion.so/Running-node-shell-a8b40a5f1ac44d6e802a2b211f5b817b?pvs=21) to access the node hosting the PVC of the bad pod, and find the data on that host node to move or delete the p, w and t directories. Without a p directory, Dgraph should come up cleanly. (If node-shell fails, download and try Lens)
2. REUSE_RAFTID error when deleted pod comes back up
    1. We have seen: `REUSE_RAFTID: Duplicate Raft ID <n> to removed member: id:<n>` and we think the deleted pod (or it’s vpc) may “remember” it’s old raftid somehow, and try to use it when re-joining. One possiblity is that Dgraph somehow resets the raftID after the p and w directories are removed, but before the node is removed from the group.
    2. Repeat the process of deleting the p, w and t directories, and delete the pod again. Ideally this will eliminate whatever record of the prior raftid still exists.