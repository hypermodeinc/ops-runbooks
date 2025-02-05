# Freshly Rebuilding Zero to avoid idx issue

This runbook is to freshly create the Zero StatefulSet to avoid  IDX issue. The only state the zero has is the **zw** directory, so once that is removed, it should come up cleanly.

To work properly with the alpha, the zero should know the timestamp, max UID and max Namespace ID used in the alpha data. So before Freshly creating Zeros, we need to retrieve those values and apply those once the Zero Pods come back up cleanly.

**`NOTE**: This runbook will  incur **downtime**.`

### Use Cases for This Runbook

This procedure is needed when one or more zero pods  cannot join the RAFT group due to idx errors

**Symptoms**:

- One of more zero(s) not working (crashing, throwing errors, cannot restart etc)
- A Zero was removed  using `/removeNode`  and it is crashing with  error `Error while joining cluster: rpc error: code = Unknown desc = REUSE_RAFTID: Reusing removed id`

## Overview

1. First Get the Leader Zero to retrieve the “**max**” values
2. Scale Down the Alpha and Zero STS in that order.
3. Delete Zero PVCs
4. Scale Zero STS to 3 Replicas and Apply Max Values
5. Scale Apha STS to first 1 Replica and then to 3 Replicas.

## Detailed Steps:

### 1.  **Set the NS variable to point to the faulty cluster**

```yaml
          export NS=<namespace>
example-  export NS=cluster-0x12345
```

### 2. Find the leader Zero

```java
kubectl exec -it zerostatefulset-0 -n $NS -- bash
```

```java
curl -s localhost:6080/state | jq '{alphas: .groups."1".members[]|select(.leader==true), zeros: .zeros[]|select(.leader==true)}'
```

### 3. Get the max values from the Zero node - ( maxUID, maxTxnTs and maxNsID) and save these to be used later

Assuming `zerostatefulset-ldr` is the zero leader from Step 2

```java
# Exec Into the Leader Zero
kubectl exec -it zerostatefulset-ldr -n $NS -- bash
```

```
curl -s localhost:6080/state | jq | grep '"max'
```

##Should return max values like the example below
"maxUID": "669546334271",
"maxTxnTs": "41060000",
"maxNsID": "41061000"

### 4. First Scale Down Alpha STS and then Zero STS

```java
kubectl scale --replicas=0 sts alphastatefulset -n $NS

```

```java
kubectl scale --replicas=0 sts zerostatefulset -n $NS
```

### 5.  Delete Zero PVCs

```java
kubectl delete pvc -l app=dgraph-zero -n $NS
```

### 6. Scale Zero STS to 3 Replicas

```java
kubectl scale --replicas=3 sts zerostatefulset -n $NS
```

### 7.  Assign Max Values on Zero Leader and verify that max values are correct

- Find the leader Zero nodes in the cluster by following **Step 2**
- Execute the assign endpoints on the leader zero with values from **Step 3**

```java
kubectl exec -it zerostatefulset-ldr -n $NS –- bash
```

```java
#Run the following Assign calls with values captured at step 3
curl "localhost:6080/assign?what=uids&num=xxxx"
curl "localhost:6080/assign?what=timestamps&num=xxxx"
curl "localhost:6080/assign?what=nsids&num=xxxx"
```

- Verify that Max Values are reflected on Zero Leader

Exec into Leader Zero

```java
kubectl exec -it zerostatefulset-ldr -n $NS -- bash
```

```java
curl -s localhost:6080/state | jq | grep '"max'
```

Verify the values retrieved are higher that with Values retrieved at Step 3 - Typically incremented by `10,000`

### Scale Apha STS to first 1 Replica and then to 3 Replicas.

```java
kubectl scale --replicas=1 sts alphastatefulset -n $NS
```

Wait for the Schema to Lazy Load and for the alpha to create a Snapshot

```java
kubectl logs alphastatefulset-0 -n <namespace> -f | grep "CreateSnapshot"
```

Now Scale Alpha STS to 3 Replicas:

```java
kubectl scale --replicas=3 sts alphastatefulset -n $NS
```

**Note**:

When a follower zero fails to come up, It can be repaired  using this method. But its better to rebuild zero than changing the script.

1. Follower going bad requires idx value change
    1. Remove Node for Zero
    2. Delete PVC of the bad zero
    3. Change Zero STS to add an idx value (if Zero with RAFTID 2 was removed, then we need to assign it 4 as idx for Zero cannot be reused.
    
    ```yaml
    idx=$(($ordinal + 1))
    if [ "$idx" -eq 2 ]; then
    idx=4
    fi
    ```
