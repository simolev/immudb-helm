# immudb-helm
ImmuDB main-replica k8s deployment

## Codenotary home assignment

Requirements:
- Permanent storage for immudb and it’s replica
- Automated provisioning with one command, like “setup_cluster”
- Grafana dashboard with meaningful metrics
- Ingress using TLS
- (optional) Ingress can load balance read request to immudb replica
- (optional) Health checks 
- (optional) Use KVM/Terraform (or other virtualization/automation technology) to install kubernetes cluster and run deployment in VM. 
- (optional) Automation of updates, i.e. when new version of immudb is released
It is not required to develop a full blown solution, but we should be able to see that you know how to create deployments and automate your work.
