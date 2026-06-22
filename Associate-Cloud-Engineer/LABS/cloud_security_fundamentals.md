# Cloud Security Fundamentals
## IAM Custome Roles
### Task 1: View available permissions for a resource
List of all permissions available in our project:
```Bash
# NOTE: This can be an incredibly LONG list
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
```


### Task 2: Get role metadata
```Bash
gcloud iam roles describe [ROLE_NAME]
```
```Bash
gcloud iam roles describe roles/viewer # or roles/editor
```

### Task 3: View the grantable roles on resources
```Bash
gcloud iam list-grantable-roles //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
```

### Task 4: Create a custom role
We need to have the *iam.roles.create* permission.\
The owner of a project or org has this by default.\
Users who are not but be assigned either *Organization ROle Administrator* or *IAM Role Administrator*.

We use custom yaml when creating to provide role definition and flags.
> Note: when creating a custom role, we kust specifiy if it applies to org or project leval
>> This is done with either --organization [ORGANIZATION_ID] or --project [PROJECT_ID] flags
```YAML
title: [ROLE_TITLE]
description: [ROLE_DESCRIPTION]
stage: [LAUNCH_STAGE]
includedPermissions:
- [PERMISSION_1]
- [PERMISSION_2]
```

let's make a custome viewr:
```Bash
vim role-definition.yaml
```
```YAML
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
```
then we create:
```Bash
gcloud iam roles create editor --project $DEVSHELL_PROJECT_ID --file role-definition.yaml
```

We can also do it inline instead of with the file:
```Bash
gcloud iam roles create viewer --project $DEVSHELL_PROJECT_ID \
--title "Role Viewer" --description "Custom role description." \
--permissions compute.instances.get,compute.instances.list --stage ALPHA
```

### Task 5: List custom roles
```Bash
gcloud iam roles list --project $DEVSHELL_PROJECT_ID
```

we can also list roles we've deleted:
```Bash
# just append this to the above
--show-deleted
```

Now that we've listed all custom, we can also list all predefined roles:
```Bash
# again...there are a LOT
gcloud iam roles list
```

### Task 6: Update existing custom role
To start, let's check out one of our existing custom roles:
```Bash
gcloud iam roles describe [ROLE_ID] --project $DEVSHELL_PROJECT_ID
```


Much like creation, we can use a yaml file:
```Bash
nano new-role-definition.yaml
```
```YAML
# append this to whatever the output of the descirbe is under PERMISSIONS
- storage.buckets.get
- storage.buckets.list
```
```YAML
description: Edit access for App Versions
etag: BwZUuhar3_0=
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
- storage.buckets.get
- storage.buckets.list
name: projects/qwiklabs-gcp-03-13007c0ac538/roles/editor
stage: ALPHA
title: Role Editor
```

now we want to update:
```Bash
gcloud iam roles update [ROLE_ID] --project $DEVSHELL_PROJECT_ID \
--file new-role-definition.yaml
```

now we can also do it with just flags:
```Bash
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--add-permissions storage.buckets.get,storage.buckets.list
```
> There is a whole [API Page](https://docs.cloud.google.com/sdk/gcloud/reference/iam/roles/update) for this.

### Task 7: Disable a custom role
This does not delete the role, just ensure it is inactive with no permissions.
```Bash
gcloud iam roles update [ROLE_ID] --project $DEVSHELL_PROJECT_ID \
--stage DISABLED
```

### Task 8: Delete a custom role
```Bash
gcloud iam roles delete [ROLE_ID] --project $DEVSHELL_PROJECT_ID
```

### Task 9: Restore a custom role
Remember we can list deleted roles?
```Bash
gcloud iam roles list --project $DEVSHELL_PROJECT_ID --show-deleted
```

now let's undelete it:
```Bash
gcloud iam roles undelete [ROLE_ID] --project $DEVSHELL_PROJECT_ID
```

---
## Service Accounts and Roles Fundamentals
### Task 1: Create and manage service accounts
create it:
```Bash
gcloud iam service-accounts create my-sa-123 --display-name "my service account"
```
now add roles to it:
```Bash
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:my-sa-123@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/editor
```

### Task 2: Use the client libraries to access BigQuery using a service account
*Nav -> IAM & Admin -> service accounts* then create.\
We want the following details:
- name: bigquery-qwiklab
- bigquery -> bigquery data viewer
- bigquery -> bigquery user


Now go to *Nav -> COmpute engine -> VM INstances* and create.\
- name: bigquery-instance
- region: us-west1
- zone: us-west1-b
- e2-medium
- bookworm
- security -> service account -> bigquery-qwiklab
- security -> access scopes -> set access for each api
- security -> bigquery -> enabled

```Bash
gcloud compute instances create bigquery-instance \
    --project=qwiklabs-gcp-02-1d9203bf7852 \
    --zone=us-west1-b \
    --machine-type=e2-medium \
    --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
    --metadata=enable-osconfig=TRUE,enable-oslogin=true \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=id-bigquery-qwiklab@qwiklabs-gcp-02-1d9203bf7852.iam.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/bigquery,https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --create-disk=auto-delete=yes,boot=yes,device-name=bigquery-instance,disk-resource-policy=projects/qwiklabs-gcp-02-1d9203bf7852/regions/us-west1/resourcePolicies/default-schedule-1,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260609,mode=rw,size=10,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ops-agent-policy=v2-template-1-7-0,goog-ec-src=vm_add-gcloud \
    --reservation-affinity=any \
&& \
printf 'agentsRule:\n  packageState: installed\n  version: latest\ninstanceFilter:\n  inclusionLabels:\n  - labels:\n      goog-ops-agent-policy: v2-template-1-7-0\n' > config.yaml \
&& \
gcloud compute instances ops-agents policies create goog-ops-agent-v2-template-1-7-0-us-west1-b \
    --project=qwiklabs-gcp-02-1d9203bf7852 \
    --zone=us-west1-b \
    --file=config.yaml
```


ssh into it.\
then let's install stuff:
```Bash
sudo apt install python3 python3-pip python3.11-venv -y
python3 -m venv myvenv
source myvenv/bin/activate
```
```Bash
sudo apt-get update
sudo apt-get install -y git python3-pip
pip3 install --upgrade pip
pip3 install google-cloud-bigquery
pip3 install pyarrow
pip3 install pandas
pip3 install db-dtypes
```
make this python file:
```bash
echo "
from google.auth import compute_engine
from google.cloud import bigquery

credentials = compute_engine.Credentials(
    service_account_email='YOUR_SERVICE_ACCOUNT')

query = '''
SELECT
  year,
  COUNT(1) as num_babies
FROM
  publicdata.samples.natality
WHERE
  year > 2000
GROUP BY
  year
'''

client = bigquery.Client(
    project='qwiklabs-gcp-01-62f3c4f9c616',
    credentials=credentials)
print(client.query(query).to_dataframe())
" > query.py
```
add the project id to the query file:
```Bash
sed -i -e "s/qwiklabs-gcp-02-1d9203bf7852/$(gcloud config get-value project)/g" query.py
```
also add service email to it:
```Bash
sed -i -e "s/YOUR_SERVICE_ACCOUNT/bigquery-qwiklab@$(gcloud config get-value project).iam.gserviceaccount.com/g" query.py
```
then run:
```Bash
python3 query.py
```

---
## VPC Network Peering
We can peer across two VPC networks privately regardless ofwherther or not in same proj or even same org.\
This lets us do SaaS ecosystems and allows for peering several network administrative domains or with other orgs. 

### Task 1: Create a custom network in both projects
To start, we have two projects this time, qwiklabs-gcp-04-1b0d595258d2 and qwiklabs-gcp-01-cf920eb51810.\
To switch between, we can run the following:
```Bash
gcloud config set project [PROJECT_ID]
```
> ensure we use two tabs for this as we will be going back and forth a lot

In project A, create a custom network:
```Bash
gcloud compute networks create network-a --subnet-mode custom
```
then a subnet:
```Bash
gcloud compute networks subnets create network-a-subnet --network network-a \
    --range 10.0.0.0/16 --region us-west1
```
followed by a VM:
```Bash
gcloud compute instances create vm-a --zone us-west1-c --network network-a --subnet network-a-subnet --machine-type e2-small
```
and finally firewall rules for ssh:
```Bash
gcloud compute firewall-rules create network-a-fw --network network-a --allow tcp:22,icmp
```

we will then mirror it with project B (in other tab!) but things will be a bit different.\
*I will be skipping this as I dont need for notes*.


### Task 2: Set up VPC network peering session
> FYI, in the GUI console, we switch projects by clicking the project name and then selecting the one we want

In project A, go to *Nav -> VPC ->  VPC Connectivity -> VPC network peering* then create connection.\
- name: peer-ab
- peer to connect to: peer-a
- in another project
- [B ID] and network-b

now to switch to B and do the same.\
- name: peer-ba
- peer to connect to: peer
- in another project
- [B ID] and network-a


in the console, we can list all orutes for all VPC networks within whichever project you are in:
```Bash
gcloud compute routes list --project qwiklabs-gcp-04-1b0d595258d2
```

### Task 3: Test connectivity (optional)
while in vm-b:
```Bash
ping -c 5 <INTERNAL_IP_OF_VM_A>
```

---
## User Auth - Identity Aware Proxy
> this one might require a pre-step of creating a bucket and downloading a zip file
### Task 1: Deploy the application and protect it with IAP



---
## Cloud KMS
### Task 1: 




---
## Private Kubernetes Cluster
### Task 1:



---
# Challenge Lab
































