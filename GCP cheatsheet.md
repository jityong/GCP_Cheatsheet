[On Starting GCloud CLI](#Setup)

[Load Balancers](#Load-Balancers)
- [Create A Network Load Balancer](#Create-Network-Load-Balancer)
- [Create A HTTP(s) Load Balancer](#Create-HTTP(s)-LB)
  - [Create A Backend Service](#Create-Backend-Service)
  - [Create A Default URL Map](#Create-Default-URL-Map)
  - [Create A HTTP Proxy](#Create-HTTP-Proxy)
  - [Create Global Forwarding Rule](#Create-Global-Forwarding-Rule)
- [Internal Load Balancer](#Internal-Load-Balancer)
 
[Instance Templates & Groups](#Instance-Templates-&-Groups)

- [Create NGINX Web Server Clusters](#Create-NGINX-Web-Server-Clusters)
- [Create Instance Templates](#Create-Instance-Templates)
- [Create MIG](#Create-MIG)
- [Create Target Pool](#Create-Target-Pool)
- [Set Firewall Rule](#Firewall-Rules)  

[Kubernetes](#Kubernetes)
- [Fundamentals](#Fundamentals)
- [Create a KE cluster](#Create-a-ke-cluster)
- [Get Authentication Credentials For Cluster](#Get-Authentication-Credentials-For-Cluster)
- [Deploy An Application To Cluster](#Deploy-appplication-to-cluster)
- [K8s Services](#K8s-Service)
  - [Expose K8s Cluster](#Expose-K8s-Cluster)
- [Delete cluster](#Delete-Cluster)

[VMs](#Virtual-Machines)
- [Create VM Instance](#Create-VM-Instance)  
  - [SSH Into VM](SSH-Into-VM)  
  
[IAM Custom Roles](#IAM-Custom-Roles)
- [Get Available Permissions](#List-Of-Available-Permissions)
- [Get Role Metadata](#Get-role-metadata)
- [Get Grantable Roles](#Get-Grantable-Roles)
- [Get Existing Custom Roles](#Get-Custom-Roles)
- [Create A Custom Role](#Create-A-Custom-Role)
  - [Using YAML File](#Create-Using-YAML-File)
  - [Using Flags](#Create-Using-Flags)
- [Update Existing Custom Role](#Update-Existing-Custom-Role)
  - [Update Using YAML File](#Update-Using-YAML-File)
  - [Update Using Flags](#Update-Using-Flags)
- [Disable Custom Role](#Disable-Custom-Role)
- [Deleting Custom Role](#Deleting-Custom-Role)
- [Undeleting Custom Role](#Undeleting-Custom-Role)
  
[CFT Scoreboard & Policy Library Constraints](#CFT-Scoreboard)

[Identity-Aware Proxy](#Identity-Aware-Proxy)

[Cloud KMS - Encryption & Decryption](#Cloud-KMS)

[VPC Networking](#VPC-Networking)
- [Fundamentals](#VPC-Fundamentals)
- [VPC Network Peering](#VPC-Network-Peering)
- [Multiple VPC Networks](#Multiple-VPC-Networks)

[Miscellaneous](#miscellaneous)   
- [Check if NGINX (or another other stuff is running)](#Is-NGINX-Running?)  
- [Setting environment variables](#Setting-environment-variables)
- [Setting a default compute zone](#Setting-a-default-compute-zone)
- [List Forwarding Rules](#List-Forwarding-Rules)
- [Perform Health Check](#Health-Check)
[Others](#Others)  
-  [Terms & Defintions](##Terms-&-Definitions)
-  GCP Console Stuff
   - [User Authentication: Identity-Aware Proxy](https://www.qwiklabs.com/focuses/5562?parent=catalog)

[Questions](#Questions)

# Setup
1. `gcloud config set project [PROJECT_ID]`
2. Set basic configs
   1. `gcloud config set compute/zone [zone]`
   2. 
# Load Balancers

## Create Network Load Balancer
```
gcloud compute forwarding-rules create [lb-name] \
         --region us-central1 \
         --ports=80 \
         --target-pool [Target_pool(created)]
```
> Allows you to balance the system's load based on incoming IP protocol data. IE: address, port, and protocol type.

## Create HTTP(s) LB
Overview:  
HTTP traffic arrives at LB --> Global Forwarding Rule --> HTTP PROXY --> Backend Svc --> Backend group --> MIG's named port
> * Remember to create a [health check](#Health-Check) first! (Used by backend svc creation)
```
gcloud compute instance-groups managed \
       set-named-ports [MIG (created)] \
       --named-ports http:[port_num]
```
> Maps a port name to the relevant port for the `MIG` --> LB svc can now forward traffic to named port.

### Create Backend Service
* Backend services define groups of backends that can receive traffic. 
```
gcloud compute backend-services create [backend_name(new)] \
      --protocol HTTP --http-health-checks http-basic-check --global
```
> * The backend service configuration contains a set of values, such as the protocol used to connect to backends, various distribution and session settings, health checks, and timeouts.   
> * These settings provide fine-grained control over how your load balancer behaves.

### Create Default URL Map
About URL Map: 
* When a request arrives at the load balancer, the load balancer routes the request to a particular backend service or backend bucket based on configurations in a URL map.
```
gcloud compute url-maps create [Map_name(new)] \
    --default-service [Backend_svc(created)]
```
> * A default URL map that directs all incoming requests to the specified backend svc(group)

### Create HTTP Proxy
``` 
gcloud compute target-http-proxies create [Poxy_name(new)] \
    --url-map [Url_map(created)]
```
> Routes request to the URL map

### Create Global Forwarding Rule
```
gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy [Proxy(created)] \
        --ports [portnum]
```

Check your forwarding rules [here](##List-Forwarding-Rules)!
> * Handles and routes incoming requests.
> * Sends traffic to a specific target HTTP or HTTPS proxy depending on the IP address, IP protocol, and port specified. 
> * The global forwarding rule does not support multiple ports.

---
## Internal Load Balancer
Key Learning pts:
- Between VMs, must be within the same network & region. 
--- 

# Instance Templates & Groups

## Create NGINX Web Server Clusters
1. A startup script to be used by every virtual machine instance to setup Nginx server upon startup
2. An instance template to use the startup script [Here](#Create-Instance-Templates)
3. A target pool [Here](#Create-Target-Group)
4. A managed instance group using the instance template [Here](#Create-MIG)


## Create Instance Template
```
gcloud compute instance-templates create [Template(new)_Name] \
         --metadata-from-file startup-script=startup.sh
```

`Startup.sh` example:
```
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

## Create MIG
```
gcloud compute instance-groups managed create [Group_Name(new)] \
         --base-instance-name [prefix_name(new)] \
         --size [num] \
         --template [Template(created)_Name] \
         --target-pool [Target_Pool(created)]
```
> --base-instance-name is the name that will get prefixed with the created VMs in this MIG

## Create Target Pool
```
gcloud compute target-pools create [Pool_Name]
```
>  A target pool allows a single access point to all the instances in a group and is necessary for load balancing (network)  in the future steps.

## Firewall Rules
```
gcloud compute firewall-rules create [Firewall_RuleName(new)] --allow [rule]
```
> ie: rule can be `tcp:80`, which allows connections to the machines on port 80 via the `EXTERNAL_IP` addresses

# Virtual Machines

## Create VM Instance
```
gcloud compute instances create [instance name] --machine-type n1-standard-2 --zone [your_zone]
```
>If `your_zone` is set in the env variable, simply use `$ZONE` instead
> --network and --subnet flags can be used

## SSH Into VM
```
gcloud compute ssh [instance name] --zone [YOUR_ZONE]
```

# Kubernetes

## Fundamentals
1. Cluster: 1 master node and the rest are worker nodes (min 1)
2. Pods stores one/more containers, which are then stored in nodes
    - Shared namespaces for contents (2 containers inside same pod can talk) & network namespace (1 IP addr per pod)
    - Pods not meant to be persistent
3. Services are created for pods, and can be implemented on pods with specified labels
    - Services provides stable [endpoints](#what-is-an-endpoint) for Pods
    - Provides different levels of access (External/Internal IPs)
      1. ClusterIP (Internal) -- the default type means that this Service is only visible inside of the cluster
      2. NodePort (External) -- gives each node in the cluster an externally accessible IP
      3. LoadBalancer (External) --  adds a load balancer from the cloud provider which forwards traffic from the service to Nodes within it
4. Deployments: Drive current state towards desired state
    - On create deployment, creates the num of pods (based on `replica`) w/ specified containers within them
## Managing Deployments Using KE
1. Rolling update
   - Triggered when deployment is edited
      ```
      kubectl edit deployment hello
      ```
   - View entries in rollout history
      ```
      kubectl rollout history deployment/hello
      ```
   - Pause rolling update
      ```
      kubectl rollout pause deployment/hello
      ```
   - View status
      ```
      kubectl rollout status deployment/hello
      ```
   - Resume rolling update
      ```
      kubectl rollout resume deployment/hello
      ```
   - Rollback rolling update
      ```
      kubectl rollout undo deployment/hello
      ```
2. Canary deployments -- to test new deployment in production with only a subset of users
    - Service targets both the original stable deployment as well as the new canary deployment
  - Chance of vising canary deployment dependd on proportion of original : canary replicas
  - Can set `sessionAffinity` in service's spec to ensure user be served the same version (either original/canary), to avoid confusion
3. Blue-green deployments
   - Creating a deployment for a whole seperate new version, wait & check status for it to be completely deployed then switch
   - To switch, simply update the service to point to a new selector (of the new deployment)
   - To rollback, do change the service back to point to selector of previous deployment

## Create a KE cluster
```
gcloud container clusters create [CLUSTER-NAME]
```

>A cluster consists of at least one cluster master machine and multiple worker machines called nodes. 
>> Nodes: Compute Engine virtual machine (VM) instances that run the Kubernetes processes necessary to make them part of the cluster.

---
### Create private cluster
More details [here](https://www.qwiklabs.com/focuses/867?parent=catalog)
```
    --enable-private-nodes
```
1. Specify /28 CIDR range
```
    --master-ipv4-cidr 172.16.0.16/28
```
2. Enable IP aliases
```
    --enable-ip-alias
```
3. Can choose to create a custom subnetwork / let them create one for you
- Let them create one for you:
```
    --create-subnetwork ""
```
> Use this flag on creating the private k8s cluster
> 
- Create a custom one:
```
gcloud compute networks subnets create my-subnet \
    --network default \
    --range 10.0.4.0/22 \
    --enable-private-ip-google-access \
    --region us-central1 \
    --secondary-range my-svc-range=10.0.32.0/20,my-pod-range=10.4.0.0/14
```
> Requires 1x primary address range & 2x secondary address ranges
```
--subnetwork my-subnet \
    --services-secondary-range-name my-svc-range \
    --cluster-secondary-range-name my-pod-range
```
> Use these flags in creation of private k8s cluster
> 

4.  Will need to authorise external address ranges from accessing if needed, otherwise only addresses within the primary & secondary range will be allowed access.
```
gcloud container clusters update [cluster_name] \
    --enable-master-authorized-networks \
    --master-authorized-networks [MY_EXTERNAL_RANGE/32]
```
---

## Get Authentication Credentials For Cluster
```
gcloud container clusters get-credentials [CLUSTER-NAME]
```

## Deploy Appplication To Cluster
```
kubectl create deployment [Deployment_Name] --image=gcr.io/google-samples/hello-app:1.0
```

> * Key in your own new deployment name  
> * `gcr.io/google-samples/hello-app:1.0` is the app's container image, which gets pulled from the GCR bucket

## K8s Service

### Expose K8s Cluster 
`kubectl expose deployment [Deployment_Name]--type=LoadBalancer --port [Port_Num]`
>* --port specifies the port that the container exposes.
> * type="LoadBalancer" creates a Compute Engine load balancer for your container.

## Delete Cluster
```
gcloud container clusters delete [CLUSTER-NAME]
```

# IAM Custom Roles
* Enables principle of least privilege
* Created by combining one or more available Cloud IAM permissions, which are represented in the form: 
```
<service>.<resource>.<verb>
```
> * EG: compute.instances.list allows user to list the GCE instances (VMs)
> * Permissions usually 1:1 to the REST methods

* Caller requires `iam.roles.create` permission to create custom role

## List Of Available Permissions
```
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
```
> Gets the permissions that the current user can add/remove in the given role for the given resources

## Get role metadata
```
gcloud iam roles describe [ROLE_NAME]
```
> Gets the role ID & permission contained in the role. IE roles/editor

## Get Grantable Roles
```
gcloud iam list-grantable-roles //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
```
> Returns a list of all roles that can be applied to a given resource

## Get Custom Roles
```
gcloud iam roles list --project $DEVSHELL_PROJECT_ID
```
> --show-deleted flag to list deleted roles
```
gcloud iam roles list
```
> Lists predefined roles

## Create A Custom Role
* Caller must possess `iam.roles.create` permission

2 ways:
1. [YAML file](#Create-Using-YAML-File) w/ role defintion 
2. [Using flags](#Create-Using-Flags)

### Create Using YAML File
Example YAML File:
```
title: [ROLE_TITLE]
description: [ROLE_DESCRIPTION]
stage: [LAUNCH_STAGE]
includedPermissions:
- [PERMISSION_1]
- [PERMISSION_2]
```
* [ROLE_TITLE] is a friendly title for the role, such as Role Viewer.
* [ROLE_DESCRIPTION] is a short description about the role, such as My custom role description.
* [LAUNCH_STAGE] indicates the stage of a role in the launch lifecycle, such as ALPHA, BETA, or GA.
* `includedPermissions` specifies the list of one or more permissions to include in the custom role, such as iam.roles.get.

Execute: 
```
gcloud iam roles create editor --project $DEVSHELL_PROJECT_ID \
--file [File_name(created)].yaml
```

### Create Using Flags
Example command:
```
gcloud iam roles create viewer --project $DEVSHELL_PROJECT_ID \
--title "Role Viewer" --description "Custom role description." \
--permissions compute.instances.get,compute.instances.list --stage ALPHA
```

### Update Existing Custom Role
* Custom roles make use of `etag` property to prevent conflicting changes to a role made by different users at the same time.
> Updates are only made if the user provided `etag` value matches the retrieved `etag` value from the existing role.
```
gcloud iam roles update
```
Used in two ways:
1. [YAML File](#Update-Using-YAML-File) w/ updated role definition
2. [Using Flags](#Update-Using-Flags)

### Update Using YAML File
1. Get current defintion of role: 
```
gcloud iam roles describe [ROLE_ID] --project $DEVSHELL_PROJECT_ID
```
2. Copy from (1) and create a new YAML file
3. Update the file accordingly
4. Use the update command w/ the filename
```
gcloud iam roles update [ROLE_ID] --project $DEVSHELL_PROJECT_ID \
--file [new-role-definition(created)].yaml
```
> [Role_ID] can be retrieved from step (1)

### Update Using Flags
List of all flags [here](https://cloud.google.com/sdk/gcloud/reference/iam/roles/update)

Examples: 
1. --add-permissions
2. --remove-permissions

```
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--add-permissions storage.buckets.get,storage.buckets.list
```

>Seperate each permission with a comma

## Disable Custom Role
* Set --stage flag to DISABLED.
```
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--stage DISABLED
```
## Deleting Custom Role
```
gcloud iam roles delete [ROLE_ID] --project $DEVSHELL_PROJECT_ID
```
> * Within 7 days: can be undeleted
> * After 7 days, enters permanent deletion process that lasts 30 days
> * After 37 days, [ROLE_ID] can be used again

## Undeleting Custom Role
```
gcloud iam roles undelete [ROLE_ID] --project $DEVSHELL_PROJECT_ID
```

# CFT Scoreboard
Basic idea: Cloud Foundation Toolkit which helps administer policies into Google Cloud environment & determine where misconfigs are occuring.
1. Set up Cloud Asset Inventory(CAI) (enable it)
```
gcloud services enable cloudasset.googleapis.com \
    --project $GOOGLE_PROJECT
```
2. Create/get a Policy Library, copying a sample into /policies/constraints directory
```
git clone https://github.com/forseti-security/policy-library.git

cp policy-library/samples/storage_blacklist_public.yaml policy-library/policies/constraints/
```
3. Create bucket to hold data that CAI will export into
```
gsutil mb -l us-central1 -p $GOOGLE_PROJECT gs://$CAI_BUCKET_NAME
```
4. Generate & export data using CAI into created bucket
```
# Export resource data
gcloud asset export \
    --output-path=gs://$CAI_BUCKET_NAME/resource_inventory.json \
    --content-type=resource \
    --project=$GOOGLE_PROJECT

# Export IAM data
gcloud asset export \
    --output-path=gs://$CAI_BUCKET_NAME/iam_inventory.json \
    --content-type=iam-policy \
    --project=$GOOGLE_PROJECT
```
> CAI is generating data from the specified project and exporting it into specified output path
5. Download and make executable the CFT scoreboard application
```
curl -o cft https://storage.googleapis.com/cft-cli/latest/cft-linux-amd64
# make executable
chmod +x cft
```
6. Run CFT scoreboard
```
./cft scorecard --policy-path=policy-library/ --bucket=$CAI_BUCKET_NAME
```
7. Proceed to update policies w/ more constraints (in /policy-library/policies/constraints) to be captured by the CFT scoreboard
- Example constraint:
```
# Add a new policy to blacklist the IAM Owner Role
cat > policy-library/policies/constraints/iam_whitelist_owner.yaml << EOF
apiVersion: constraints.gatekeeper.sh/v1alpha1
kind: GCPIAMAllowedBindingsConstraintV1
metadata:
  name: whitelist_owner
  annotations:
    description: List any users granted Owner
spec:
  severity: high
  match:
    target: ["organization/*"]
    exclude: []
  parameters:
    mode: whitelist
    assetType: cloudresourcemanager.googleapis.com/Project
    role: roles/owner
    members:
    - "serviceAccount:admiral@qwiklabs-services-prod.iam.gserviceaccount.com"
EOF
```
> Specifically whitelist only the specified member of the specified role. Any other users(members) with that role not whitelisted will be captured by the CFT. 

# Identity-Aware Proxy
More details [here](https://www.qwiklabs.com/focuses/5562?parent=catalog)
- Basic idea: Control / restrict access to selected users for a application, using the app's domain & address
- Can make use of Cryptographic Verification to ensure IAP is not bypassed/ turned off

# Cloud KMS
1. Enable Cloud KMS
```
gcloud services enable cloudkms.googleapis.com
```
2. Create KeyRing:
```
 gcloud kms keyrings create [KEYRING_NAME] --location global
```
3. Create CryptoKey:
```
gcloud kms keys create [CRYPTOKEY_NAME] --location global \
      --keyring [KEYRING_NAME] \
      --purpose encryption
```
>KeyRing stores CryptoKeys
4. Encrypt data using `encrpyt endpoint`
```
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:encrypt" \
  -d "{\"plaintext\":\"$PLAINTEXT\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type: application/json" \
  | jq .ciphertext -r > 1.encrypted
```
  - `PLAINTEXT=$(cat 1. | base64 -w0)`
  >-  Encode fiile to be encrypted to base64 first --> allows binary data to be sent to API as plaintext  
  > - `jq .ciphertext -r > 1.encrypted` uses cli utility `jq`  to parse out `ciphertext` property to file `1.encrypted`

5. To decrypt/ check encrypted
```
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:decrypt" \
  -d "{\"ciphertext\":\"$(cat 1.encrypted)\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type:application/json" \
| jq .plaintext -r | base64 -d
```

## IAM Permissions
1. Manage KMS resources - `cloudkms.admin`
```
gcloud kms keyrings add-iam-policy-binding $KEYRING_NAME \
    --location global \
    --member user:$USER_EMAIL \
    --role roles/cloudkms.admin
```
To get current authorized user:
- ` USER_EMAIL=$(gcloud auth list --limit=1 2>/dev/null | grep '@' | awk '{print $2}')`
  
1. Encrypt & decrypt - `cloudkms.cryptoKeyEncrypterDecrypter`
```
gcloud kms keyrings add-iam-policy-binding $KEYRING_NAME \
    --location global \
    --member user:$USER_EMAIL \
    --role roles/cloudkms.cryptoKeyEncrypterDecrypter
```
# VPC Networking

## VPC Fundamentals
Key learning points:  
1. Default VPC Network
   - Every subnet is associated with a GCP region & CIDR addr for IP addresses range and a gateway
   - A route for each subnet and 1 for `Default internet gateway`(0.0.0.0./0)

2. Default Firewall rules
   - default-allow-icmp
      - Allows ping to `External IP` from all IP addresses
   - default-allow-internal
      - allows all protocols to go through within the same network IP range (internal network ip addresses)
   - default-allow-rdp
       - allows rdp
   - default-allows-ssh
      - allows ssh from all IP addresses; tcp:22

3. To create VM instance, there must be a VPC network

---
## VPC Network Peering
More details found [here](https://www.qwiklabs.com/focuses/964?parent=catalog)
- Allows private connectivity across two VPC networks regardless of whether or not they belong to the same project or the same organization.
  
1. Configure the other parties network into the current project's network peering setting. Do for both side before it can work.
---

## Multiple VPC Networks
Key learning points:
1. Pinging External IP is dependent on ICMP firewall rule, independent of same/ different network its in
2. Internal address: Only accessible by the same network (regardless of zone)
3. VM instance can belong to > 1 network
   - Number of interfaces allowed in instance is dependent on instance's machine type & num of vCPUs. ie 4vCPU --> 4 network interfaces --> up to 4 networks
   - Every interface gets a route for the subnet that it is in
   - Take note of `nic0` (primary interface eth0, the default route) -->  Any traffic other than the directly connected subnets will leave via this (can manually configure routes to change this behavior)

---
## VPC Networks - Controlling Access
Key learning points:
1. Can tag VM instances with network tags and create firewall rules that applies to instances with these network tags
2. Service accounts - to be used by VM instances instead of actual users
   1. Network Admin: Create, modify, delete networking resources (except firewall rules & SSL certificates)
   2. Security Admin: Create, modify & delete firewall rules & SSL certificates
3. Using svc accounts:
   1. After creation & download of key pairs during creation of service account, SSH into VM instance and upload the keypair credentials (json)
   2. Run `gcloud auth activate-service-account --key-file credentials.json` to authorize the VM instance with credentials of Svc acc, so VM instance can use it
   3. Edit permissions @ IAM


# Miscellaneous

## Setting a default compute zone
`gcloud config set compute/zone us-central1-a`
>Your compute zone is an approximate regional location in which your clusters and their resources live.

## Setting environment variables
```
export PROJECT_ID=<your_project_ID>
```
## Is NGINX Running?

```
ps auwx | grep nginx
```

## List-Forwarding-Rules
```
gcloud compute forwarding-rules list
```

## Health Check
```
gcloud compute http-health-checks create http-basic-check
```
>Health checks verify that the instance is responding to HTTP or HTTPS traffic


# Others

## Terms & Definitions



# Questions
- Using firewall to allow tcp:80 to be able to connect to external IP of instances VS exposing a port of clusters using LB? 
- Creating forwarding rule == creating Network LB?
- Backend-services add MIG?
  - Backend services created through this command will start out without any backend groups. To add backend groups, use 'gcloud compute backend-services add-backend' or 'gcloud compute backend-services edit'.
- Creating Load Balancer on GCP Console no need URL MAP, HTTP Proxies etc? But on CLI need? Plus on Console theres a firewall rule to allow for the healthcheck probe IP address to pass but on CLI we don't have that, only hace tcp:80? 


### What is an endpoint?
  - It is a to access its resources(eg a Pod) - the resource behind the 'endpoint'
  - An IP to access a pod, like an object-oriented representation of a REST API endpoint
