# Enabling or Disabling High-Availability for a Dgraph Cluster

**Revision: 01/31/2025**

### Use Cases for This Runbook

These are the scenarios where you need to use this runbook:

- Enable HA
    - Enable high-availability (HA) for an existing non-HA Dgraph cluster
    - Increase the no. of Zero and/or Alpha replicas in an existing HA cluster.
- Disable HA
    - Disable high-availability (HA) for an existing HA Dgraph cluster

## Overview of the procedure(s):

1. Record the RAFT group configuration prior to enabling HA/Disabling HA
2. For Enabling HA
    1. Edit ZeroStatefulSet to have `--replicas=3` and Alpha StatefulSet to have `--raft` superflag to specify the alpha group 
    2. Scale Zero and Alpha Statefulsets to 3 replicas 
    3. Verify that HA is enabled and the RAFT groups have the right members
3. For Disabling HA
    1. Record RAFT state and Max Values from Zero Leader
    2. Remove `w` directory in Alpha-0,  scale down Zero/Alpha STS and delete all Alpha/Zero Replica PVCs
    3. Edit ZeroStatefulSet to have `--replicas=1` and Alpha StatefulSet to remove `--raft` superflag 
    4. Scale up Zero to 1 replica, assign Max values and then scale  Alpha Statefulsets to 1 replica
    5. Verify that HA is enabled and the RAFT groups have the right members

## Steps to enable High Availability:

Following set of steps  assumes a Dgraph cluster setup with no high-availability configuration, consisting of a single Alpha and a single Zero.

### **1. Set  NS  to point to the  target Dgraph cluster and verify Alpha and Zero Pods are running**

```yaml
          export NS=<namespace>
example-  export NS=cluster-0x12345

% kubectl get pods -n $NS 

alphastatefulset-0                 1/1     Running   0          86m
zerostatefulset-0                  1/1     Running   0          86m

```

### 2. Take a note of the RAFT group  members prior to enabling HA

```yaml
% kubectl exec -it zerostatefulset-0 -n $NS -- bash
% curl -s localhost:6080/state | jq '{alphas: .groups."1".members, zeros: .zeros}'
```

Response should  look like below - **Two Groups** - alphas(groupId=1) and zeros(groupId=0) each consists  of one member.

```json

{
  "alphas": {
    "id": "1",
    "groupId": 1,
    "addr": "alphastatefulset-0.dgraph-alpha-service.cluster-0x82b2466e9.svc.cluster.local:7080",
    "leader": true,
    "amDead": false,
    "lastUpdate": "1738270871",
    "learner": false,
    "clusterInfoOnly": false,
    "forceGroupId": false
  },
  "zeros": {
    "id": "1",
    "groupId": 0,
    "addr": "zerostatefulset-0.dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080",
    "leader": true,
    "amDead": false,
    "lastUpdate": "0",
    "learner": false,
    "clusterInfoOnly": false,
    "forceGroupId": false
  }
}

```

### 3. Edit the Zero StatefulSet to update the `--replicas` flag in zero  command, specifying the number of replicas for each Alpha group.

`Note: This will cause a restart for zero statefulset pod`

In this case, we want to have 3 Alpha Replicas, so the value needs to be change to 3. After saving the changes to the  zero sts, the zero pod would restart.

```yaml
k
%% kubectl edit sts zerostatefulset -n $NS

#Make the following changes in the statefulset(--replicas 1 to --replicas 3)
if [[ $ordinal -eq 0 ]]; then
   exec dgraph zero --my=$(hostname -f):5080 --raft "idx=$idx" ....  **--replicas 3** 
else
   exec dgraph zero --my=$(hostname -f):5080 --raft "idx=$idx" ... **--replicas 3** --peer zerostatefulset-0.dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080
fi
 
```

### 4.  Edit Alpha Statefulset to add `--raft` flag in alpha command to specify the Alpha group

`Note: This will cause a restart for Alpha pod`

```yaml
% kubectl edit sts alphastatefulset -n $NS

#add --raft "group=1" to the existing dgraph alpha command like below
dgraph alpha --my=$(hostname -f):7080 --security "whitelist=0.0.0.0:255.255.255.255" --zero dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080  --graphql "lambda-url=http://router.fission.svc.cluster.local:80/cluster/0x82b2466e9/0/graphql-worker" --limit "mutations=strict;max-pending-queries=16;" --v=2 --trace "jaeger=" --raft "group=1"
```

### 5. Scale up both the Alpha and Zero StatefulSets to 3 replicas, then verify that they have started successfully and are healthy.

```yaml
% kubectl scale sts alphastatefulset zerostatefulset --replicas=3 -n $NS
```

Confirm that there are now 3 pods for both Alpha and Zero statefulset, and ensure that all pods are healthy with `READY` state as `1/1`

```yaml
% kubectl get pods -n $NS

#In the response, verify that there are 3 alpha and 3 zero pods and all the pods have READY state as 1/1
NAME                 READY   STATUS    RESTARTS   AGE
alphastatefulset-0   1/1     Running   0          55m
alphastatefulset-1   1/1     Running   0          3m58s
alphastatefulset-2   1/1     Running   0          3m58s
zerostatefulset-0    1/1     Running   0          64m
zerostatefulset-1    1/1     Running   0          3m58s
zerostatefulset-2    1/1     Running   0          3m58s
```

### 6. Verify that both Alpha and Zero groups are now having 3 members

```yaml
% kubectl exec -it zerostatefulset-0 -n $NS -- bash
% curl -s localhost:6080/state | jq '{alphas: .groups."1".members, zeros: .zeros}'
```

Response should  look like below - **Two Groups** - alphas(groupId=1) and zeros(groupId=0) each should now have  three members.

```json
{
  "alphas": {
    "1": {
      "id": "1",
      "groupId": 1,
      "addr": "alphastatefulset-0.dgraph-alpha-service.cluster-0x82b2466e9.svc.cluster.local:7080",
      "leader": true,
      "amDead": false,
      "lastUpdate": "1738277045",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    },
    "2": {
      "id": "2",
      "groupId": 1,
      "addr": "alphastatefulset-2.dgraph-alpha-service.cluster-0x82b2466e9.svc.cluster.local:7080",
      "leader": false,
      "amDead": false,
      "lastUpdate": "0",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": true
    },
    "3": {
      "id": "3",
      "groupId": 1,
      "addr": "alphastatefulset-1.dgraph-alpha-service.cluster-0x82b2466e9.svc.cluster.local:7080",
      "leader": false,
      "amDead": false,
      "lastUpdate": "0",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": true
    }
  },
  "zeros": {
    "1": {
      "id": "1",
      "groupId": 0,
      "addr": "zerostatefulset-0.dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080",
      "leader": true,
      "amDead": false,
      "lastUpdate": "0",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    },
    "2": {
      "id": "2",
      "groupId": 0,
      "addr": "zerostatefulset-1.dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080",
      "leader": false,
      "amDead": false,
      "lastUpdate": "0",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    },
    "3": {
      "id": "3",
      "groupId": 0,
      "addr": "zerostatefulset-2.dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080",
      "leader": false,
      "amDead": false,
      "lastUpdate": "0",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    }
  }
}
```

### 7. Additional Verification using Ratel

In the **Cluster** tab of Ratel, the backend should display 3 Zero pods under the "**Zeros**" group and 3 Alpha pods under "**Group #1**," as shown below.

![image.png](Enabling%20or%20Disabling%20High-Availability%20for%20a%20Dgra%2018be0dd4f7368024ba46e8db0f49b457/image.png)

## Steps to disable High Availability:

Converting a  cluster from HA to non-HA, requires rebuilding the cluster with the `p` directory and `max` values from Zero Leader. Additionally, we also need to delete the PVCs associated with the HA Replicas

Following set of steps  assumes a Dgraph cluster configured with  high-availability, consisting of 3  Alpha and 3 Zero pods.

### 1.  **Find the leader Alpha and Zero  in the cluster by checking the state endpoint of any Zero:**

```yaml
% kubectl exec -it zerostatefulset-0 -n $NS -- bash
% curl -s localhost:6080/state | jq '{alphas: .groups."1".members[]|select(.leader==true), zeros: .zeros[]|select(.leader==true)}'
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

### 2. Get the max values from the Zero leader - ( maxUID, maxTxnTs and maxNsID) and save these to be used later

Assuming `zerostatefulset-ldr` is the zero leader from Step 1

```yaml
% kubectl exec -it zerostatefulset-ldr -n $NS -- bash
% curl -s localhost:6080/state | jq | grep '"max'

##Should return max values like the example below
"maxUID": "10000",
"maxTxnTs": "20000",
"maxNsID": "0"
```

### 3. Clean up `w` directory in Alpha-0 pod

```yaml
% kubectl exec -it alphastatefulset-0 -n $NS -- mv /dgraph/w /dgraph/w_<date>
```

### 4. Scale down Alpha and Zero statefulsets and verify that no Alpha and Zero pods are running

```yaml
% kubectl scale sts alphastatefulset zerostatefulset --replicas=0 -n $NS
```

### 5. Delete All Zero PVCs,  Alpha-1 and Alpha-2 PVCs

Get the PVCs

```yaml
% kubectl get pvc -n $NS 
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
alphapvc-alphastatefulset-0   Bound    pvc-3c5dc23b-94ab-4216-9612-da46e582dc28   32Gi       RWO            dc-storage     <unset>                 6m6s
alphapvc-alphastatefulset-1   Bound    pvc-a584a942-1745-456e-82ff-51084e540582   32Gi       RWO            dc-storage     <unset>                 6m4s
alphapvc-alphastatefulset-2   Bound    pvc-2fd7fe77-37e8-4109-bf26-7600118efe9c   32Gi       RWO            dc-storage     <unset>                 6m4s
zeropvc-zerostatefulset-0     Bound    pvc-06d9725b-01f5-4705-bf7f-6c4f5f5766c7   10Gi       RWO            dc-storage     <unset>                 6m6s
zeropvc-zerostatefulset-1     Bound    pvc-031f4fe5-af1e-4244-ac18-f13ccc3d38e0   10Gi       RWO            dc-storage     <unset>                 6m6s
zeropvc-zerostatefulset-2     Bound    pvc-fb8e1771-ed23-4924-ae25-8f489f070f52   10Gi       RWO            dc-storage     <unset>                 6m6s
```

Delete  all  the PVCs **except** the one  associated with Alpha-0. In our case `ReclaimPolicy` is `Delete` , so PVs associated with PVCs will get automatically deleted 

```yaml
 % kubectl delete pvc alphapvc-alphastatefulset-1 alphapvc-alphastatefulset-2  zeropvc-zerostatefulset-0  zeropvc-zerostatefulset-1 zeropvc-zerostatefulset-2 -n $NS
persistentvolumeclaim "alphapvc-alphastatefulset-1" deleted
persistentvolumeclaim "alphapvc-alphastatefulset-2" deleted
persistentvolumeclaim "zeropvc-zerostatefulset-0" deleted
persistentvolumeclaim "zeropvc-zerostatefulset-1" deleted
persistentvolumeclaim "zeropvc-zerostatefulset-2" deleted
```

NOTE:

If PVs are **not** configured with the `ReclaimPolicy` set to `Delete`, you will need to patch each of the PVs to update the policy before deleting the PVCs. Alternatively, the volumes will have to be manually deleted.

```yaml
% kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'

Example,
% kubectl patch pv pvc-3c5dc23b-94ab-4216-9612-da46e582dc28  -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

### 6. Edit the Zero StatefulSet to update the `--replicas` flag in zero  command, reducing the number of replicas to 1 for each Alpha group.

`Note: This will restart  zero statefulset pods`

```yaml
kubectl edit sts zerostatefulset -n $NS

#Make the following changes in the statefulset(--replicas 3 to --replicas 1)
if [[ $ordinal -eq 0 ]]; then
   exec dgraph zero --my=$(hostname -f):5080 --raft "idx=$idx" --trace "jaeger=" ... **--replicas 1** --enterprise_license=/opt/dgraph/license --cid 0x82b2466e9
else
   exec dgraph zero --my=$(hostname -f):5080 --raft "idx=$idx" --trace "jaeger=" ... **--replicas 1** --enterprise_license=/opt/dgraph/license --cid 0x82b2466e9 --peer zerostatefulset-0.dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080
fi

```

### 7.   Edit Alpha Statefulset to remove `--raft` flag in alpha command to specify the Alpha group

`Note: This will cause a restart for Alpha pod`

Note: This step is optional, but recommended.

```yaml
kubectl edit sts alphastatefulset -n $NS

#remove --raft "group=1" from  the existing dgraph alpha 
dgraph alpha --my=$(hostname -f):7080 --security "whitelist=0.0.0.0:255.255.255.255" --zero dgraph-zero-service.cluster-0x82b2466e9.svc.cluster.local:5080  --graphql "lambda-url=http://router.fission.svc.cluster.local:80/cluster/0x82b2466e9/0/graphql-worker" --limit "mutations=strict;max-pending-queries=16;" --v=2 --trace "jaeger=" ~~--raft "group=1"~~
```

### 8. Scale up zero Statefulset to 1 replica and apply max values

```yaml
kubectl scale --replicas=1 sts zerostatefulset -n $NS

kubectl get pods -n $NS | grep zerostatefulset
### Wait  for zero  pod to be up and READY state 1/1
```

Execute the assign endpoints on zero with values from **Step 2**

```java
kubectl exec -it zerostatefulset-0 -n $NS –- bash

#Run the following Assign calls with values captured at step 2
curl "localhost:6080/assign?what=uids&num=xxxx"
curl "localhost:6080/assign?what=timestamps&num=xxxx"
curl "localhost:6080/assign?what=nsids&num=xxxx" (Not needed if it does not have multi-tenancy)
```

### 9. Scale the Alpha Statefulset  up to  1 replica. Alpha-0 should start up.

Wait until ready state becomes **1/1** for `alphastatefulset-0` pod

```java
kubectl scale --replicas=1 sts alphastatefulset -n $NS

kubectl get pods -n $NS | grep alpha
NAME                     READY   STATUS    RESTARTS
alphastatefulset-0       1/1     Running   0
```

### 10.  Verify that both Alpha and Zero groups are now having 1 member

```java
kubectl exec -it zerostatefulset-0 -n $NS -- bash
curl -s localhost:6080/state | jq '{alphas: .groups."1".members, zeros: .zeros}'
```

Response should  look like below - **Two Groups** - alphas(groupId=1) and zeros(groupId=0) each should now have  1 member.

```json
{
  "alphas": {
    "1": {
      "id": "1",
      "groupId": 1,
      "addr": "alphastatefulset-0.dgraph-alpha-service.cluster-0x82b250de5.svc.cluster.local:7080",
      "leader": true,
      "amDead": false,
      "lastUpdate": "1738370678",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    }
  },
  "zeros": {
    "1": {
      "id": "1",
      "groupId": 0,
      "addr": "zerostatefulset-0.dgraph-zero-service.cluster-0x82b250de5.svc.cluster.local:5080",
      "leader": true,
      "amDead": false,
      "lastUpdate": "0",
      "learner": false,
      "clusterInfoOnly": false,
      "forceGroupId": false
    }
  }
}
```

In Ratel, you can verify that Alpha and Zero Groups have only 1 member.

![image.png](Enabling%20or%20Disabling%20High-Availability%20for%20a%20Dgra%2018be0dd4f7368024ba46e8db0f49b457/image%201.png)