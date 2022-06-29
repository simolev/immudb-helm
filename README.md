# immudb-helm

ImmuDB main-replica k8s deployment

## Codenotary home assignment

The goal is to develop simple deployment using Kubernetes/Helm that contains immudb with read replica, an ingress that exposes immudb endpoints, and a Grafana dashboard to see metrics. Requirements:

* Permanent storage for immudb and it’s replica
* Automated provisioning with one command, like “setup_cluster”
* Grafana dashboard with meaningful metrics
* Ingress using TLS
* (optional) Ingress can load balance read request to immudb replica
* (optional) Health checks
* (optional) Use KVM/Terraform (or other virtualization/automation technology) to install kubernetes cluster and run deployment in VM.
* (optional) Automation of updates, i.e. when new version of immudb is released

It is not required to develop a full blown solution, but we should be able to see that you know how to create deployments and automate your work.

### General reasoning

ImmuDB provides two replication modes: replicating a single database and replicating the systembd database. The first method allows for great flexibility in that any ImmuDB instance can be main or replica for any of its databases. The latter implies that only the system database `systemdb` and default database `defaultdb` are replicated: any further database created on the main node is not copied over to the replicas, and replicas are forbidden from creating databases an users (user definitions and their passwords are replicated as well). The official immudb helm package from https://packages.codenotary.org/helm is leveraged as a starting point. For the first scenario, a Helm post-install job connects to every ImmuDB instance and runs the appropriate command to create the main or replicated database:

```bash
apk add bind-tools libc6-compat
wget https://github.com/vchain-us/immudb/releases/download/v1.3.0/immuadmin-v1.3.0-linux-amd64 -O immuadmin
chmod +x immuadmin
echo "Number of replicas is $REPLICAS"
echo "My hostname is "$(hostname)
READ=$(hostname | sed 's/[^-]*$/immudb-read/')
WRITE=$(hostname | sed 's/[^-]*$/immudb-write/')
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
```

I've then commented the code and implemented the second strategy, that relies upon the creation of environment variables:

```
env:
- name: IMMUDB_REPLICATION_ENABLED
  value: "true"
- name: IMMUDB_REPLICATION_MASTER_ADDRESS
  value: {{ include "immudb.fullname" . }}-write
- name: IMMUDB_REPLICATION_FOLLOWER_USERNAME
  value: "immudb"
- name: IMMUDB_REPLICATION_FOLLOWER_PASSWORD
  value: "immudb"
```

In order for the pods to be able to communicate with each other, the following services are created:

```
<name>-immudb-write  # for the main, write-enabled ImmuDB instance
<name>-immudb-read   # gathering all the replica instances
<name>-immudb-read-n # pointing to the n-th replica
```

### Permanent storage for immudb and its replica

Permanent storage is already configured in the official Helm package:

```
      volumes:
      - name: immudb-storage
        persistentVolumeClaim:
          claimName: {{ include "immudb.fullname" . }}
...
  volumeClaimTemplates:
  - metadata:
      name: immudb-storage
    spec:
      accessModes:
      - ReadWriteOnce
      {{- if .Values.volume.class }}
      storageClassName: {{ .Values.volume.class | quote }}
      {{- end }}
      resources:
        requests:
          storage: {{ .Values.volume.size }}
```

I only had to make sure to configure a StorageClass. Setting up OpenEBS is as easy as running `microk8s enable openebs`, but I also deployed the [NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) with Helm. The desired StorageClass must then be specified in the values.yaml file or on the command line:

```
helm install . --generate-name --set replicaCount=<n> -n <namespace> --set volume.class=<my-storage-class> --set ingress.hostname=<my-domain-name>
```

**Note:** In the official Helm chart 1.3.0, file templates/statefulset.yaml seems to have a typo: `.Values.volume.Class` should be `.Values.volume.class`.

### Automated provisioning with one command, like “setup_cluster”

The Helm chart can be deployed with a single command:

```
helm install --generate-name --set replicaCount=<n> --set volume.class=<my-storage-class> --set ingress.hostname=<my-domain-name> https://github.com/simolev/immudb-helm/releases/download/v1.3.0/immudb-1.3.0.tgz
```

A few things are assumed to pre-exist on the cluster:

1. a StorageClass
2. an Ingress Controller (microk8s enable ingress)
3. correct dns records pointing to the public ip of a cluster node. When the nodes are more than one, they shoud be behind a public-facing loadbalancer or share a virtual ip by means of keeapalived.
4. a valid TLS certificate (kubectl create secret tls tls-wildcard --key=wildcard.key --cert=wildcard.chain)
5. a monitoring stack with Prometheus operator and Grafana (microk8s enable prometheus)

### Grafana dashboard with meaningful metrics

A ServiceMonitor CRD needs to be created in order for Prometheus to add the job definition for scraping metrics from the ImmuDB installation. this is defined in the `templates/servicemonitor.yaml` file. Since we created a number of services, a custom label `immudb-monitor` is applied to the one we want to monitor in order to avoid doubles.

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "immudb.fullname" . }}
  labels:
    {{- include "immudb.labels" . | nindent 4 }}
    app.kubernetes.io/monitor: immudb-monitor
spec:
  selector:
    matchLabels:
      {{- include "immudb.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/monitor: immudb-monitor
  endpoints:
    - port: metrics
```

It makes sense to add the [dashboard](https://github.com/codenotary/immudb/blob/master/tools/monitoring/grafana-dashboard.json) from the official immudb repository. A `grafana.` subdomain is also added to the Ingress definition.

### Ingress using TLS

An Nginx Ingress Controller is assumed to be present and configured with a defualt wildcard TLS certificate for the domain. The web interface and the metrics page are exposed under the hostnames:

* `immudb-write.`for the read/write instance
* `immudb-read.`for load-balance across all read-only instances. Not really useful, except maybe for exposing metrics to an external collector. At least defines a hostname to be used for the gRPC
* `immudb-read-<n>.`for directly accessing the n-th read-only replica instance

Please note that the `/metrics` endpoint is exposed together with the web interface: this is just for demonstration purposes as it does NOT make sense to expose metrics over the Internet if Prometheus is internal to the cluster.

### (optional) Ingress can load balance read request to immudb replica

In the official Helm chart, a service is already configured for the gRPC port, which is where clients will be directing their requests. With a number of replicas >0, this will end up load-balancing requests across both the writer instance and all the read-only replicas. To load-balance requests exclusively to the read-only replicas, a new service must be created with a selector `app.kubernetes.io/role: read`that matches only the read-only replicas. Also, the ConfigMap for Nginx TCP load-balancing must relate the external ports to the Kubernetes services:

```
data:
  "3322": {{.Release.Namespace}}/{{ include "immudb.fullname" . }}-write:3322
  "3323": {{.Release.Namespace}}/{{ include "immudb.fullname" . }}-read:3322
```

**Note:** plain Nginx cannot do host-based TCP load-balancing, therefore a new port must be used for reads. With this setup, external port 3322 will connect to the read/write instance, and port 3323 will load-balance requests across all read-only replicas.

### (optional) Health checks

Liveness and readiness probes are already configured in the official Helm release:

```
          livenessProbe:
            httpGet:
              path: /readyz
              port: metrics
            failureThreshold: 9
          readinessProbe:
            httpGet:
              path: /readyz
              port: metrics
```

Since a dedicated `/readyz` endpoint is provided, it makes sense to rely on it. Otherwise, we would probably have defined a gRPC probe:

```
    livenessProbe:
      initialDelaySeconds: 90
      grpc:
        port: 3322
```

### (optional) Use KVM/Terraform (or other virtualization/automation technology) to install kubernetes cluster and run deployment in VM

CDK can be used to create an EC2 instance on AWS. Custom commands can be provided to be run upon VM creation, to setup a Kubernetes cluster, install Helm and deploy the ImmuDB chart.

```
    const myVm = new ec2.Instance(this, 'myVm', {
      instanceType: new ec2.InstanceType('t2.micro'),
      machineImage: machineImage,
      vpc: defaultVpc,
      securityGroup: myVmSecurityGroup,
      keyName: 'myKey',
      // init script
      init: ec2.CloudFormationInit.fromElements(
        ec2.InitCommand.shellCommand('sudo systemctl enable iscsid'),
        ec2.InitCommand.shellCommand('sudo snap install microk8s --classic --channel=1.24'),
        ec2.InitCommand.shellCommand('sudo microk8s enable dns ingress metrics-server prometheus openebs helm3'),
        ec2.InitCommand.shellCommand('sudo microk8s helm3 install --generate-name --set replicaCount=2 --set volume.class=openebs-hostpath https://github.com/simolev/immudb-helm/releases/download/v1.3.0/immudb-1.3.0.tgz')
      )
    })
```

### Room for improvement

This was just an exercise and not every aspect has been thoroughly tested since the goal was more to create a proof-of concept and a base for further discussion. I recognize there's plenty of room for improvement: the provided solution can be made more flexible, more dependable and more portable in a number of ways:

* the monitoring stack could be deployed together with the main application
* automatic deployment of Kubernetes should create a HA n-node cluster
* microk8s can be replaced with kubeadm
* hostnames could be automatically created/deleted by a hooked dns service provider
* hooks to the firewall could create/delete the relevant rules
* individual certificates could be created on-demand by cert-manager
* a user different from immudb should be created and used for replication
* passwords should always saved to and retrieved from a secret
* a value should determine whether to deploy replication at the database level or systemdb level
* the name of the replicated database should be configurable

### Highlights of the provided solution

* The number of read-only replicas is configurable at install time, as well as modifiable later
* Minimum number of replicas is zero: only the read/write instance will be deployed in that case
* Each replica's web interface is individually accessible via a dedicated hostname, all under a specified second-level domain name
