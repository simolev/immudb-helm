# immudb-helm
ImmuDB main-replica k8s deployment

## Codenotary home assignment
The goal is to develop simple deployment using Kubernetes/Helm that contains immudb with read replica, an ingress that exposes immudb endpoints, and a Grafana dashboard to see metrics. Requirements:
- Permanent storage for immudb and it’s replica
- Automated provisioning with one command, like “setup_cluster”
- Grafana dashboard with meaningful metrics
- Ingress using TLS
- (optional) Ingress can load balance read request to immudb replica
- (optional) Health checks 
- (optional) Use KVM/Terraform (or other virtualization/automation technology) to install kubernetes cluster and run deployment in VM. 
- (optional) Automation of updates, i.e. when new version of immudb is released

It is not required to develop a full blown solution, but we should be able to see that you know how to create deployments and automate your work.

### General reasoning
ImmuDB provides two replication modes: replicating a single database and replicating the systembd database. The first method allows for great flexiility in that any ImmuDB instance can be main or replica for any of its databases. The latter implies that only the system database systemdb and default database defaultdb are replicated: any further database created on the main node is not copied over to the replicas, and replicas are forbidden from creating databases an users (user definitions and their passwords are replicated as well). The official immudb helm package from https://packages.codenotary.org/helm is leveraged as a starting point. For the first scenario, a Helm post-install job connects to every ImmuDB instance and runs the appropriate command to create the main or replicated database:

    apk add bind-tools libc6-compat
    wget https://github.com/vchain-us/immudb/releases/download/v1.3.0/immuadmin-v1.3.0-linux-amd64 -O immuadmin
    chmod +x immuadmin
    echo "Number of replicas is $REPLICAS"
    echo "My hostname is "$(hostname)
    READ=$(hostname | sed 's/[^-\]*$/immudb-read/')
    WRITE=$(hostname | sed 's/[^-\]*$/immudb-write/')
    echo "Service name for reads is $READ"
    echo "Service name for writes is $WRITE"
    timeout 30 sh -c 'until nslookup $0; do echo waiting for service $0; sleep 1; done' $WRITE
    timeout 30 sh -c 'until nc -z $0 $1; do sleep 1; done' $WRITE 3322
    echo -n "immudb" | ./immuadmin -a $WRITE login immudb
    ./immuadmin -a $WRITE database create replicadb
    ./immuadmin -a $WRITE logout
    for i in $(seq 0 $((REPLICAS-1))); do
      timeout 30 sh -c 'until nslookup $0; do echo waiting for service $0; sleep 1; done' $READ-$i
      timeout 30 sh -c 'until nc -z $0 $1; do sleep 1; done' $READ-$i 8080
      echo -n "immudb" | ./immuadmin -a $READ-$i login immudb
      ./immuadmin -a $READ-$i database create --replication-enabled=true --replication-follower-username=immudb --replication-follower-password=immudb --replication-master-address $WRITE --replication-master-database=replicadb replicadb
      ./immuadmin -a $READ-$i logout
    done

I've then commented the code and implemented the second strategy, that relies upon the creation of environment variables:

    env:
    - name: IMMUDB_REPLICATION_ENABLED
      value: "true"
    - name: IMMUDB_REPLICATION_MASTER_ADDRESS
      value: {{ include "immudb.fullname" . }}-write
    - name: IMMUDB_REPLICATION_FOLLOWER_USERNAME
      value: "immudb"
    - name: IMMUDB_REPLICATION_FOLLOWER_PASSWORD
      value: "immudb"


### Permanent storage for immudb and its replica
Permanent storage is already configured in the 

