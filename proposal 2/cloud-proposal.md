## CLOUD PROPOSAL 

Goal: deploy a scalable microservices architecture on GCP

### GENERAL

### API Quotas 
Based on predicted traffic patterns, we might need to adjust request higher API quotas for our resources in the Cloud Quotas API console.   

#### Resource Hierarchy 
The following resource hierarchy is proposed: 
Organiation (ingenius.studio)
|_ Folder (ingenius-engineering)
    |_ Project (ingenius-lab)
    |_ Project (ingenius-prod)
### IaaC

We will the entire state of Ingenius GCP using Terraform Cloud/Terraform state stored on GCP Cloud Storage Bucket to follow best DevOps practices and have a single source of truth. 
We will have correpsonding lab and prod environments setup on terraform


#### Networking 
Each Project resource will use the default VPC. 
We propose using CloudDNS to manage DNS records within our Project. 

- Cloud Load Balancing (in both prod) 

(will connect with any backend: App Engine, Cloud Run, GKE)
20/month for 100 GiB inbound/outbound, 5 forwarding rules

- NAT Gateway (in both lab and prod): 8-15 per month in each env, 100 GiB data processed   
- NEG: Serverless NEG to point to network endpoints of Cloud Run/App Engine + tie with Load Balancer

- Firewall Rules, CloudDNS 

#### IAM 
As per "The Principle of Least Privlege," the following IAM structure is proposed:  
IAM Principles: 
1) Juno-Developers (Google Group) 
2) Ingenius-Developers (Google Group)
3) Admin (Google Group) 
4) Finance (Google Group)

IAM Policies (at Project scope): 
1) Juno-Developers <-> `roles/editor` 
2) Ingenius-Developers <-> `roles/editor` 
3) Admin <-> `roles/viewer`

IAM Policies (at Billing Account scope): 
1) Juno-Developers <-> `roles/billing.admin`, `roles/billing.user`
2) Ingenius-Developers <-> `roles/billing.admin`, `roles/billing.user`
3) Admin <-> `roles/billing.admin`
4) Finance <-> `roles/billing.creator`

Plus any service accounts 

#### Container Registry 
We will create and manage an Artifact Registry within each Project to host our Docker images. 

### Deployment Options 
Note that the prices provided are for the production environment. The Lab Enviornment will be built with much less compute 

1) GKE

pros: 
- certificates, load balancing, etc. all managed within GKE 
- easily scalable for adding more applications, traffic
- autoscaling 

cons: 
- costly 

1a) 2 Node Cluster (cost-effective) 555 
Cluster option 1a will run on 2 n1-standard-8 nodes with an attached 40 GiB Balanced Persistent Disk. 


1b) 2 Node Regional/Zonal Cluster: 628 

 
Can enable Cluster autoscaler and Node auto-provisioning to reach up to x Nodes based on traffic increase. To see price, do base price + (275(x)*y) with y being a decimal of how much of the month it will be up 


Note that K8 costs can increase with added resources, for example Load Balancers. 
 
2) cloud run (front + back) 

- Cloud Run Service: NodeJS GraphQL Backend 

1 vCPU, 0.5 GiBs per instance, per 10 million requests: 4.15 month 

2 vCPU, 0.5 GiBs per instance, per 10 million requests: 5.05 month 

2 vCPU, 2 GiBs per instance, per 10 million requests: 5.19 per month 
- Cloud Run Service: Next JS Frontend 
- Enterprise CloudSQL MYSQL w/ 1 db-standard-2 instance (cost friendly) 20 GiB: 102.02 
- Enterprise CloudSQL MYSQL w/ 1 db-standard-2 instance (cost friendly) 60 GiB: 108.82
- Enterprise CloudSQL MYSQL w/ 1 db-standard-2 instance (cost friendly) 100 GiB: 115.62
- Enterprise CloudSQL MYSQL w/ 1 db-standard-2 instance (cost friendly) 200 GiB: 132.62
- Enterprise Cloud SQL MySQL w/ 1 db-standard-4 instance: 20-200 GiB: 200.65 - 231.25 
- Enterprise Cloud SQL MySQL w/ 1 db-highmem-4 instance: 20-200 GiB: 256.86 - 287.46 

can proportionally calculate  based on number of instances you wanna run 

 
- Artifact Registry: store container images (free upto 0.5 GB, 0.10 per 0.5 GB/month). most container images are less than 0.5 GB   
- IAM: backend service account with `roles/cloudsql.client`
- Secret Manager: to store database credentials #TODO: move to other  

abuot 100 active secrets would cost 6.00 
- Cloud Standard Storage (single regional) : store static asserts 0.023 per GB per month 

steps to deploy: 
1) deploy backend on Cloud Run from source (will automatically build container image)   
2) deploy frontend on Cloud Run from source (will automatically build container image)   
3) migrate SQLite database data to CloudSQL 
4) connect CloudSQL instance to Cloud Run backend 

3) cloud run + app engine standard
for both app engine options explain that u can autoscale 

backend: app engine standard 
app engine standard F1 1 instance with 40 GiB: 1.15/month 
app engine standard F1 2 instance with 40 GiB: 31.57/month 

app engine standard F2 1 instance with 40 GiB: 31.57/month 
app engine standard F2 2 instance with 40 GiB: 104.57/month 
4) cloud run + app engine flexible 
will scale number of instances based on load 
1 vCPu, 2 GiB, 10-40 GiB persistent disk, 40 GiB data transfer: 48.76 per month 
2 vCPU, 2 GiB, 10-40 GiB persistent dis, 40 GiB data transfer: 87.56


#### Database 
A schema diagram generated using DBVisualizer is included.  

#### Cache
**Deployment Options**:
1) MemoryStore: Memcache 
1 instance, 2 nodes, 2 vCPU, 8 GiB memory, 730 hours: 249.49/month
same as above but with 1 node: 124.98

2) MemoryStore: Redis    

Shared-Core Nano 3 shards, all month: 139.28/month 


Memcache is best for simple, high-speed caching needs where persistence and complex data types are not required.
Redis offers more versatility with support for complex data structures, persistence, and a broader range of use cases, making it suitable for more advanced applications.



### SECURITY COMPLIANCE 

#### Images
For a higher cost, automatic image scanning within Artifact Registry can be toggled. 
Alternativly, an image scanning tool (ie. Aqua Security Trivy) can be built into CI/CD 
Or can scan images using Docker Hub Vulnerability Scanner

#### Access

Will use GCloud Policy Analyzer to configure least privileges for users + service accounts. 

#### Logging 
50 GiB per month: 1/month
70 GiB per month: 11.40/month 
100 GiB per month 27/month

can setup Cloud Logging to store and later analyze using Cloud Monitoring dahsboards, BigQuery, Cloud Trace if necessary 
1) Platform Logs 
2) GKE workload Logs 

Can specifyically use Cloud Audit Logs to track audit trails over gcloud, record all administrative activity. 

#### Pentesting 

Will use open source cloud security auditiing tools (ie. Prowler, AquaSecurity CloudSploit) to verify GCP security.


### CI/CD
**Options**: 
1) Cloud Build (reccomended)
0.003/build-minute 
2) Github Actions 
2k Minutes per month, 500 MB github free for organizations. then 0.008 per GB of storage, per minute usage  

When a PR is merged into the repository, will be merged into lab env. Will have another workflow that must be manually triggered to promote from lab to prod. 


### OTHER

#### Load Testing 
In order to ensure our systems are up to scale, we can conduct simualted load testing using Locust using GKE. We will shut off these resources when not being used. 

The following components used are: 
- App Engine 
- Artifact Registry 
- Cloud Build
- Cloud Storage 
- GKE 


put in diagram here from https://cloud.google.com/architecture/distributed-load-testing-using-gke

#### Budgeting with GCP
Can register a Commited Use Contract with GCP for upto 57% discount. commit to using x amount of resources (ie. vCPUs, emmory, etc.) for 1-3 years fixed period 

