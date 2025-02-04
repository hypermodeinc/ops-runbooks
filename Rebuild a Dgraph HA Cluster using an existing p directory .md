# Rebuild a Dgraph HA  Cluster using an existing ‘p’ directory

**Revision: 01/29/2025**

### Use Cases for This Runbook

These are the scenarios where you need to use this Runbook:

- When more than 2 alpha pods are down due to any error condition - Panic, OOM(due to an unoptimised query),  or any other error condition
- Loss of Quorum. For example,  disk full scenario
- Cloning a cluster for testing or root cause analysis

## Overview of the procedure:

1. Obtain Max Values from Zero Leader to record  maxUID, maxTxnTs and maxNsID
2. Set up `alpha-0` pod with the right data in 'p' directory  and cleaning up `w/t` (for alpha-0) and `p/w/t` for alpha-1/alpha-2
3. Clean up Zero PVCs and  scale down alphastatefulsets followed by zerostatefulset 
4. Scale up zerostatefulset and apply UIDs, transaction timestamps and namespace Ids 
5. Start Alpha pods incrementally and verify they are healthy 

## Detailed Steps:

### **1. Set  NS variable to point to the faulty cluster and find leader Alpha and Zero**

```bash
          export NS=<namespace>
example-  export NS=cluster-0x12345

kubectl exec -it zerostatefulset-0 -n $NS -- bash
curl -s localhost:6080/state | jq '{alphas: .groups."1".members[]|select(.leader==true), zeros: .zeros[]|select(.leader==true)}'
```

### 2. Get the max values from the Zero leader - ( maxUID, maxTxnTs and maxNsID) and save these to be used later

Assuming `zerostatefulset-ldr` is the zero leader from Step 1

```java
kubectl exec -it zerostatefulset-ldr -n $NS -- bash
curl -s localhost:6080/state | jq | grep '"max'

##Should return max values like the example below
"maxUID": "669546334271",
"maxTxnTs": "41060000",
"maxNsID": "41061000"
```

### 3. Setting up alpha nodes with the right data in 'p' directories

- If **Alpha-0** and **Zero-0** are the leaders (from Step 2)
    - Rename the ‘**w**’ and ‘**t**’ directories on all Alphas and rename the ‘p’ directories on just Alpha-1 and Alpha-2
    - **Retain** the ‘p’ directory on **Alpha-0** - *This data is retained to rebuild the cluster*
- If **Alpha-0** is not a leader,
    - **Copy the ‘p’ directory from good alpha to alpha-0.**
        - This can bee done using `kubectl copy`
        - First rename the `p` directory in alpha-0
            - `kubectl exec -it alphastatefulset-0 -n $NS -- bash
            mv p p_<date>`
        - The good `p` dir has to be copied first to a local folder and then copied over to the alpha-0 as kubectl does not allow copying data between pods.. **NOTE**: Bring up an EC2 instance in the same region as the cluster. The time and **data transfer cost is very high** if its done across regions.
        - Commands to copy data from good alpha to  outside  - for example, if alpha-1 is good
            - `kubectl cp  -n $NS alphastatefulset-1:/dgraph/p   p`
        - Commands to copy data  from outside to a pod to `alpha-0`
            - `kubectl cp  -n $NS p alphastatefulset-0:/dgraph/p`
        - Alternatively, the data can be copied over to S3 bucket in the same region using `aws s3 cp --recursive` and then copied back to the other alpha pods from S3.
        - If its not possible to exec into the alpha-0 , then it has to be copied over to the pvc by doing `node-shell` into the node
    - Steps to get inside a alpha and renaming the directories:
        - `kubectl exec -it alphastatefulset-0 -n $NS -- bash
        mv t t_<date>; mv w w_<date>    ## Do it for all Alpha
        ##Example- mv t t_11272022 ; mv w w_11272022 )
        mv p p_<date>;          ### *Do it only for the non-leader alpha*`

### 4. Cleanup all Zero pods:

- Rename/remove the ‘zw’ directories on **all** the Zeros.
- `kubectl exec -it zerostatefulset-0 -n $NS -- mv /dgraph/zw /dgraph/zw_<date>
kubectl exec -it zerostatefulset-1 -n $NS -- mv /dgraph/zw /dgraph/zw_<date>
kubectl exec -it zerostatefulset-2 -n $NS -- mv /dgraph/zw /dgraph/zw_<date>`

### 5. Scale down the Alpha group to 0 replicas. Wait for all alpha pods to be gone.

```java
kubectl scale --replicas=0 sts alphastatefulset -n $NS

# Check for pods to be gone with the following command
kubectl get pods -n $NS | grep alphastatefulset  ## Should return no pods
```

### 6. After the Alphas are scaled down, scale down the Zero group

```java
kubectl scale --replicas=0 sts zerostatefulset -n $NS

# Check for pods to be gone with the following command
kubectl get pods -n $NS | grep zerostatefulset <-- Should return no pods
```

### 7. Scale up the Zero group to 3 replicas

```java
kubectl scale --replicas=3 sts zerostatefulset -n $NS

# Check for all 3 pods to be up and ready  with the following command

kubectl get pods -n $NS | grep zerostatefulset
### Wait  for all  3 pods to be up and READY state 1/1

```

### 8. Assign the UIDs, transaction timestamps and namespace Ids to the Zero endpoint using the values captured in Step (3).

- Find the leader Zero nodes in the cluster by following **Step 1**
- Execute the assign endpoints on the leader zero with values from **Step 2**

**IMPORTANT***:* This MUST be done at **zero leader** and  before starting the Alphas.

```java
kubectl exec -it zerostatefulset-ldr -n $NS –- bash

#Run the following Assign calls with values captured at step 3
curl "localhost:6080/assign?what=uids&num=xxxx"
curl "localhost:6080/assign?what=timestamps&num=xxxx"
curl "localhost:6080/assign?what=nsids&num=xxxx" (Not needed if it does not have multi-tenancy)
```

### 9. Scale the Alpha group up to just 1 replica. Alpha-0 should start up.

```java
kubectl scale --replicas=1 sts alphastatefulset -n $NS
```

### 10. Wait for the schema to lazy-load on Alpha-0.

- Look at the logs as the schema Lazy Loads

```java
kubectl logs alphastatefulset-0 -n $NS -f
```

- Wait until ready state becomes **1/1** for `alphastatefulset-0` pod

**IMPORTANT**: This process could sometime take  sometime depending on how many predicates are there.

```java
kubectl get pods -n $NS
NAME                     READY   STATUS    RESTARTS
alphastatefulset-0       1/1     Running   0
```

### 11. Once the schema is loaded, wait for Alpha-0 to create a snapshot. Check logs for `CreateSnapshot`

```java
kubectl logs alphastatefulset-0 -n $NS -f | grep "CreateSnapshot"
```

### 12. Scale up Alpha group to 3 replicas

```java
kubectl scale –-replicas=3 sts alphastatefulset -n $NS
```

Check Alpha-0 logs for a “Snapshot Streaming” to ensure snapshot transfer is starting for Alpha-1 and Alpha-2.

```java
kubectl logs alphastatefulset-0  -n $NS -f | grep -i "Snapshot Streaming"
```

### 13. Verify that Alpha-1 and Alpha-2 are healthy

After a snapshot stream is successful, confirm that the Alpha-1 and Alpha-2 are ‘READY’ and ‘Running’. The schema load should be much faster this time  on account of the snapshot stream. At this point, the cluster would start processing mutations successfully since quorum is reached.

### 14. Finally, scale the Alpha group to 3 replicas.

```java
kubectl scale –-replicas=3 sts alphastatefulset -n $NS
```

### 15. The cluster should now be fully operational with all 3 Alphas replicas running.

Verify that all 3 alpha and zero pods are running and are in ready state is **1/1**

```java
kubectl get pods -n $NS | grep statefulset

# Results Should look like below
alphastatefulset-0    1/1     Running
alphastatefulset-1    1/1     Running
alphastatefulset-2    1/1     Running
zerostatefulset-0     1/1     Running
zerostatefulset-1     1/1     Running
zerostatefulset-2     1/1     Running
```