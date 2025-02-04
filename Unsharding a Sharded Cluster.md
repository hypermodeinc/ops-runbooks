# Unsharding a Sharded Cluster

**Revision date**: 01/31/2025

### Use Cases for This Runbook

This procedure is applicable for unsharding a Dgraph cluster, i.e. converting a cluster with multiple Alpha groups into one with a single group. 

**Overview of the procedure:**

1. Stop incoming requests to the cluster.
2. Take a full export of the cluster data in RDF (default) or JSON format.
3. Manually combine/stitch the multiple exported schemas into a single schema file.
4. Scale down the Alpha and Zero replicas and delete PVCs. 
5. Modify the existing Alpha and Zero statefulsets to remove sharding options and and scale up with just HA enabled.
6. Import the data exported in (2) using the Live Loader. 
7. Start incoming requests to the cluster.

NOTE: A check for the counts of the different `Types` of nodes and a small collection of 

## Detailed Steps:

Before proceeding, configure `kubectl` to use the correct namespace by default for all subsequent commands.

```yaml
          export NS=<namespace>
example-  export NS=cluster-0x12345
```

Optional: You can set the context to use the namespace in question:

```yaml
kubectl config set-context --current --namespace=$NS
```

### **1. Stop incoming requests to the cluster**

Stop incoming requests to the cluster by updating the headless service for the cluster.  This is to ensure that the latest copy of the data is exported by preventing new updates whilst the export is in progress. 

Update the value of `app` in the service selector `spec.selector` to a different (any) value. 

In this example, the name of the headless service is `dgraph-service` , and the selector is set to `dgraph-alpha` .

```bash
selector:
  app: dgraph-alpha
```

Modify the value of `app` as below:

```yaml
kubectl patch svc dgraph-service --type='merge' -p '{"spec":{"selector":
{"app":"dgraph-alpha-modified"}}}'
```

**NOTE**: Replace the name of the headless service viz. `dgraph-service` above to the actual service name of your cluster.

Success Response:

```yaml
service/dgraph-service patched
```

**How is data exported from a sharded Dgraph cluster ?**

- On a sharded cluster, one Alpha-replica from each of the Alpha groups is involved in an export of its data, apart from the replica servicing the request for the  `export` mutation.
- For example, on a cluster with two groups and three HA replicas each, data is exported from the replica from an existing group which received the the `export` request, as well as from from another replica belonging to the other (remaining) group.
- The target can be either a path on the pod’s local storage, or an AWS S3 bucket, depending on the `destination` specified in the `export` mutation. In case of local path on the pod, say, `/dgraph/exports` , data is exported to that path, but on both Alpha replicas involved in the Groups’ exports.
- Exported data and schema files need to be accessed directly from the pod’s local-storage for subsequent steps and a final Live Load.

### 2. Run a cluster-wide export

- Open an interactive terminal to any of the Alpha pods( preferably the leader), from any of the existing Alpha groups/shards:

```yaml
kubectl exec -it $(kubectl get pods | awk '/alpha/ {print $1}') -- bash
```

- Create an `export.graphql` file on the pod specifying the export format, destination etc.
For a cluster with a single tenant, create the following mutation:

```yaml
cat << EOF > export.graphql
mutation {
  export(input: {
	  format: "RDF", 
	  destination: "/dgraph/exports",
  }) {
    response {
      code
      message
    }
  }
}
EOF
```

- To export data from a multi-tenant cluster, specify `namespace: -1` to create a cluster-wide export for all namespaces.
    
    ```yaml
    cat << EOF > export.graphql
    mutation {
      export(input: {
    	  namespace: -1,
    	  format: "RDF", 
    	  destination: "/dgraph/exports",
      }) {
        response {
          code
          message
        }
      }
    }
    EOF
    ```
    

- Start an export of the data using `curl` as below:

```
curl --request POST "http:localhost:8080/admin" \
	--header "Content-Type: application/graphql" \
	--upload-file export.graphql
```

- To export to an AWS S3 bucket, setup authentication using AWS Keys and set the `destination` to the target S3 bucket:
    
    ```yaml
    ACCESS_KEY="<AWS Access Key ID>" && SECRET_KEY="<AWS Secret Access Key>" && \
    cat << EOF > export.graphql
    mutation {
      export(input: {
    	  namespace: -1,
    	  destination: "/dgraph/exports",
    	  accessKey: $ACCESS_KEY,
     	  secretKey: $SECRET_KEY,
      }) {
        response {
          code
          message
        }
      }
    }
    EOF
    ```
    
- If multi-tenancy is enabled for the cluster, we need to login to the cluster as the `Guardian of the galaxy` user, and use the resultant `accessJwt` for taking a cluster-wide export of all namespaces’ data.
    
    ```yaml
    $ USER=groot && PASSWORD=<groot-password>
    ```
    
    ```yaml
    ACCESS_JWT=curl localhost:8080/admin --header "Content-Type: application/graphql" \
    	-d 'mutation {login(userId: $USER, password: $PASSWORD, namespace: -1) \
    	{response {accessJWT refreshJWT}}}' | jq -r '.data.login.response.accessJWT'
    
    ```
    
- Start an export using the `accessJwt` obtained above:
    
    ```yaml
    curl --request POST "http:localhost:8080/admin" \
    	--header "Content-Type: application/graphql" \
    	--header "X-Dgraph-AccessToken: $ACCESS_JWT" \
    	--upload-file export.graphql
    ```
    
    Tail Alpha logs to ensure that all shards received the proposal for the `export` mutation:
    
    ```prolog
    kubectl logs -f <alpha-pod> | grep -i 'export'
    
    export.go:44] Got export request through GraphQL admin API
    export.go:1015] Got readonly ts from Zero: 180
    export.go:1019] Requesting export for groups: [2 1]
    ```
    
    The size of the exported data is also available:
    
    ```yaml
    log.go:33] Export Sent data of size 33 GiB
    log.go:33] Export Sent data of size 33 TiB
    ```
    
    Once completed successfully, check which of the sharded replicas were involved in the export, along with the path for each:
    
    TODO: To get logs from a Sharded Cloud CLuster
    
    ```yaml
    export.go:935] Export DONE for group 2 at timestamp 180.
    export.go:935] Export DONE for group 1 at timestamp 180.
    export.go:978] Export request: group_id:2 read_ts:180 unix_ts:1738332003 format:"rdf"  OK.
    export.go:1057] Export at readTs 180 DONE
    queue.go:330] task 0x18a011e2b: exported files: [dgraph.r180.u0131.1400/g01.rdf.gz dgraph.r180.u0131.1400/g01.schema.gz dgraph.r180.u0131.1400/g01.gql_schema.gz]
    queue.go:241] task 0x18a011e2b: completed successfully
    ```
    
    In the example above, the cluster is configured with 2 shards/groups and thus, each shard will contain its exported data within the same directory via. `dgraph.r180.u0131.1400` in the example above. 
    

### 3. Merge the multiple exported schemas into a single one.

**IMP NOTE: ***Access to a jump server or a local workstation is required for this step.*

While exporting from a sharded cluster,  separate schema-files are created; one each for every Alpha group, containing just those predicates and type definitions from that group.

```prolog
export.go:483] Exporting to file at /dgraph/exports/dgraph.r180.u0131.1400/g01.schema.gz
export.go:483] Exporting to file at /dgraph/exports/dgraph.r180.u0131.1400/g02.schema.gz
```

However, before we can LiveLoad data into an unsharded cluster, we need to ensure that we have just one schema file - a mandatory requirement for the Live and Bulk loaders. 
The behavior is the same with the data files and the GraphQL schema:

```prolog
Exporting to file at /dgraph/exports/dgraph.r180.u0131.1400/g01.rdf.gz
Exporting to file at /dgraph/exports/dgraph.r180.u0131.1400/g02.rdf.gz

Exporting to file at /dgraph/exports/dgraph.r180.u0131.1400/g01.gql_schema.gz
Exporting to file at /dgraph/exports/dgraph.r180.u0131.1400/g02.gql_schema.gz
```

The goal is to merge or stitch the multiple schemas together, by just copying all of the predicate and type definitions from either, into a single `g01.schema` file. This single schema file would be subsequently used to load data into our unsharded cluster with just one group/shard. 

- Copy the exports from the pods to the jump server:

```jsx
kubectl cp alphastatefulset-0:/dgraph/exports/ && \
kubectl cp alphastatefulset-3:/dgraph/exports/
```

- If the export was targeted to an S3 bucket, download the export files from the bucket directly:

```yaml
aws s3 cp --recursive s3:///<bucket-name>/exports/ && \
aws s3 cp --recursive s3:///<bucket-name>/exports/
```

- Decompress the DQL schema files:

```bash
for file in `ls -l ./export/dgraph* | grep '.schema.gz' | grep -v 'gql_schema' | awk '{print $9}'`; do \echo "Extracting schema $file"; gzip -dk ./export/dgraph*/$file; echo "Done extracting $file"; done
```

Once extracted, using a text editor like `vi` or `nano` manually copy all of the definitions for user predicates and types from the Group 2 schema file viz. `g02.schema.gz` , to the Group 1’s file i.e. `g01.schema.gz` . 

This ensures that we only have one consolidated schema file, a mandatory requirement for data-loads to unsharded clusters. 

**IMPORTANT:** Do not copy any of the internal `dgraph.*` predicate or type definitions between the schemas.

To cite an example, let’s assume we have the following Group 1 and Shard 2 schemas respectively:

```yaml
# Group 1

[0x0] <source>:[uid] . 
[0x0] <target>:[uid] . 
[0x0] <account>:default . 
[0x0] <dgraph.type>:[string] @index(exact) . 
[0x0] <dgraph.drop.op>:string . 
[0x0] <dgraph.graphql.xid>:string @index(exact) @upsert . 
[0x0] <dgraph.graphql.schema>:string . 
[0x0] <dgraph.graphql.p_query>:string @index(sha256) . 
[0x0] type <dgraph.graphql> {
	dgraph.graphql.schema
	dgraph.graphql.xid
}
[0x0] type <dgraph.graphql.persisted_query> {
	dgraph.graphql.p_query
}
```

```yaml
# Group 2

[0x0] <date>:default . 
[0x0] <amount>:default . 
[0x0] type <dgraph.graphql> {
	dgraph.graphql.schema
	dgraph.graphql.xid
}
[0x0] type <dgraph.graphql.persisted_query> {
	dgraph.graphql.p_query
}
```

After copying, the contents of `g01.schema` must be:

```bash
# Original group 1 definitions: 

[0x0] <source>:[uid] . 
[0x0] <target>:[uid] . 
[0x0] <account>:default . 
[0x0] <dgraph.type>:[string] @index(exact) . 
[0x0] <dgraph.drop.op>:string . 
[0x0] <dgraph.graphql.xid>:string @index(exact) @upsert . 
[0x0] <dgraph.graphql.schema>:string . 
[0x0] <dgraph.graphql.p_query>:string @index(sha256) . 
[0x0] type <dgraph.graphql> {
	dgraph.graphql.schema
	dgraph.graphql.xid
}
[0x0] type <dgraph.graphql.persisted_query> {
	dgraph.graphql.p_query
}

# Copied Group 2 definitions

[0x0] <date>:default . 
[0x0] <amount>:default . 
```

### 4. Update Zero and Alpha statefulsets and Kubernetes replica-counts.

- Scale down all Alphas and Zeros

```bash
kubectl scale sts zerostatefulset alphastatefulset --replicas=0 
```

 

- Delete all Alpha and Zero PVCs:

```bash
for pvc in $(kubectl get pvc | awk '$0 ~ /alpha|zero/ { print $1 }'); do 
echo ""; echo "Deleting PVC $pvc"; kubectl delete pvc $pvc; echo "Done"; done

```

- Update the `dgraph zero` startup config for the Zero statefulset:

```bash
if [[ $ordinal -eq 0 ]]; then
      exec dgraph zero --my=$(hostname -f):5080 --raft "idx=$idx" --replicas 3
     else
      exec dgraph zero --my=$(hostname -f):5080 --raft "idx=$idx" --replicas 3 --peer <zero-.svc.cluster.local:5080
```

- IMPORTANT: If your Alpha STS configs include the `--raft group=` superflag, remove the flags from the Alpha STS.

```bash
dgraph alpha --my=$(hostname -f):7080 --zero dgraph-zero-service.cluster-0x82b1f5037.svc.cluster.local:5080  ~~--raft group=$idx~~
```

- Update/ensure Kubernetes’ `.spec.replicas` is set to `3` as required for an HA cluster, and scale-up the replicas to start the cluster with new directories

```bash
kubectl scale sts zerostatefulset alphastatefulset --replicas=3 
```

Once all of the Alpha and Zero pods have a `READY` status of `1/1` , we have a fresh cluster with  no data and can proceed with a Live Load next.

### 5. Live Load using the exported data-files and stitched/merged schema file

NOTE: Check the current Zero leader in the new cluster by calling the `/state` endpoint.

- Start a new live load in a `tmux` session or equivalent.

```bash
tmux new -s liveload
```

- Once open, port-forward to the Zero leader and (any) one of the Alpha pods:

```bash
ZERO_LEADER=<"Zero-leader-pod"> # For Example, "zerostatefulset-0"
ZERO_IP=$(kubectl get pods $ZERO_LEADER | awk '{print $6}')
nohup kubectl port-forward $ZERO_LEADER 5080:5080 & \
nohup kubectl port-forward $(kubectl get pods -l app=dgraph-alpha | head -1) 7080:7080 &
```

- Start a Live Load:

```bash
dgraph live --zero=localhost:5080 --alpha localhost:7080 \
--schema ./export/dgraph*/g01.schema \
--files ./export/dgraph*
```

- If ACLs are enabled, we need to login during a Live Load:

```bash
dgraph live --zero=localhost:5080 --alpha localhost:7080 \
--schema ./export/dgraph*/g01.schema \
--files ./export/dgraph* \
--creds "user=groot;namespace=1"
```

- This would initiate a Live Load into the new HA cluster running.
- Once completed, run any existing consistency checks and then start traffic back into your cluster:

```bash
kubectl patch svc dgraph-service --type='merge' -p '{"spec":{"selector":
{"app":"dgraph-alpha"}}}'
```

This concludes the process for converting a sharded HA cluster into an unsharded HA cluster.