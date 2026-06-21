# Build a secure Google Cloud Network
Welcome to week 3.

## Securing Virttual Machines useing CHrome Enterprise Premium

### Task 1: Enable IAP TCP forwarding in your GOogle Clopud Project
This one is fairly simple and involves using the Console.\
We will go to *APIs and Services -> Library* then look or serach for **IAP**.\
Here, select *Cloud Identity-Aware Proxy API* and enable it.

### Task 2: Create Linux and Windows instances
For thuis labe we want one Linux machine and two windows ones.\
The linux and one Windows will be for demo purposes and the other windows for testing connections.

The machines weill have the following stuff.\
Linux:
- linux-iap
- us-west1-a
- default network
- external NONE 
> note: the default vm is already Debian 12

Windows (demo):
- windows-iap
- us-west1-a
- Windows Server 2016 Datacenter
- default network
- external NONE

Windows (connectivity):
- windows-connectivity
- us-west1-a
- custom images -> Qwiklabs Resources -> iap-desktop-v001
- Security -> Access scopes -> Allow full access to all cloud APIs
- networking default; leave external on

### Task 3: Test connectivity to your linux and windows instances
We first wnat to attempt to SSH in to our Linux iap.\
It will fail despite the button still existing because it relies on aan external IP.\
Without one, we'd need a firewall rule/internal proxy which we don't have.

For the windows IAP we can see that RDP is grayed out for the same reason.

### Task 4: Configure the required firewall rules for BCE
We want to go to *VPC Network -> Firewall* and create a new rule:
- name: allow-ingress-from-iap
- direction: ingress
- target: all instances in the network
- source: IPv4 ranges
- source ranges: 35.235.240.0/20
- protocols and ports: TCP 22, 3389 (SSH and RDP)

### Task 5: Grant permissions to use IAP TCP forwarding
We want to go to *Security -> Identity-Aware Proxy*.\
Switch tabs to the *SSH and TCP resources*.

Select our two IAPVMs and click *Add principal*.\
We want to enter the service account associated with the Windows connectivity VM.\
In this case it is in the following form:
```Bash
# find under the API and Identitiy Manamgent section of the VM page
873897115788-compute@developer.gserviceaccount.com
```
Select *Cloud IAP -> IAP-secured tuinnel user* role and save.

> The IAP-Secured Tunnel User role will grant the windows-connectivity instance to connect to resources using IAP.
>> This step can also take a few minutes to update.

### Task 6: Use IAP desktop to connect to the Windows and Linux instances
[IAP Desktop](https://github.com/GoogleCloudPlatform/iap-desktop) is a specific tool for connecting.

We will use it by RDPing into our windows connectivity VM.\
we download the RDP file and connect using our personal machine with specific creds.\
once in find the IAP Desktop shortcut and run it.
> FYI its crazy laggy

it will prompt for a sign in where we will use the one provided for the lab itself (the studen-xxx-xxx qwiklab one).
```Bash
student-03-e99149e3f20b@qwiklabs.net:siK6Ts6acke2
```
After allowing a bunch of stuff, we can then get access back to the .exe.\
In this case, let's select the our project from the list and add it.\
If it works, we will now see our hierarchy:
- Google Cloud
    - <project name>
    - <zone>
        - VM
        - VM
        - VM


We can now close our RDP.

### Task 7: Demonstrate tunneling using SSH and RDP connections
Back in the VM page, let's select the down-arrow on the windows connectivity VM and *set windows password*.\
We want to copy and save it:
```Bash
student_03_e99149e3f:8*K;y>/@v1@J@2V
```
now let's download a new RDP file using the drop-down.\
We can now long in with this new passowrd on our actual account instaad of the other one.
> note it took forever and was also super laggy

From here, let's go to the *Google Cloud SDK Shell* using the desktop shortcut and run the following to SSH into our linux machine:
```Bash
gcloud compute ssh linux-iap
```
we type our 'y' and then select teh right zone (us-west1-a).\
If it works, we will see "no external IP so *using iap*.\
We will get a PuTTy alert to accept.

Now in Putty, go to settings and allow tunnel connections locally.\
This is done by going to the top left icon in the window and routing to *Change settings* followed by going to *Connection -> SSH -> Tunnels*.\
We now want to hit the checkbox for *local ports...* the apply and close putty.

We now want to use this command to get in:
```Bash
gcloud compute start-iap-tunnel windows-iap 3389 --local-host-port=localhost:0  --zone=us-west1-a
```
we will now get:
```Bash
Listening on port [50279] # we need this number
```

Now, let's go back to the compute enginge page and make a password for our windows IAP instance.
```Bash
student_03_e99149e3f:qXg0[aY}DnnV<+7
```
Now lets go back to the RDP session and go into RDP in the server.\
Go to localhost:<port number> and enter the creds we got above and voila!\
We are now in our Windows IAP.

> Done
>> You learned how to use BeyondCorp Enterprise (BCE) and Identity-Aware Proxy (IAP) TCP forwarding by deploying 2 VM's, windows-iap and linux-iap, without IP addresses and configuring an IAP tunnel which gave you access to both instances using a third VM, windows-connectivity.

---
##  Multiple VPC Networks
A VPC allows for the maintainence of isoalted environments within a larger cloud structure for granular data protectionm, network access, and application security.

This lab will have us creatre two custom mode netowrks (managementnet and privatenet) with firewall rules and VMs as seen in the foillowing network diagram:\
[diagram](https://cdn.qwiklabs.com/OBtRY37ZCmWiHi%2FHsG8XCSGDBfsuKk3IMJVgQscsg2E%3D)

### Task 1: Create custome mode VPC networks with firewall rules
To start go to *nav -> VPC network -> VPC Networks*.\
We will see a *default* and *mynetwork* network which are typical premades.

Let's *Create a VPC network* and name is **managementnet** with *custom* subnet creation>
- name: managementsubnet-1
- region: us-east4
- ip range: 10.130.0.0/20

We can even generate the CLI commands once done:
```Bash
gcloud compute networks create managementnet --project=qwiklabs-gcp-04-e7994c056ff5 --subnet-mode=custom --bgp-routing-mode=regional --bgp-best-path-selection-mode=legacy && gcloud compute networks subnets create managementsubnet-1 --project=qwiklabs-gcp-04-e7994c056ff5 --range=10.130.0.0/20 --stack-type=IPV4_ONLY --network=managementnet --region=us-east4
```

Create and make another.\
This time we will use CLI to create our *privatenet* network:
```Bash
gcloud compute networks create privatenet --subnet-mode=custom
```
then make the two subnets:
```Bash
gcloud compute networks subnets create privatesubnet-1 --network=privatenet --region=us-east4 --range=172.16.0.0/24
```
```Bash
gcloud compute networks subnets create privatesubnet-2 --network=privatenet --region=europe-west1 --range=172.20.0.0/20
```

we can also use the CLI to see all VPC now:
```Bash
gcloud compute networks list
```
we can also sort by subnets:
```Bash
gcloud compute networks subnets list --sort-by=NETWORK
```

> Firewall rules time

Let's go now to *VPC network -> Firewall*.\
Create a firewall rule:
- name:  	managementnet-allow-icmp-ssh-rdp
- network: managementnet
- targets: all in network
- source filer: ipv3 ranges
- source ranges: 0..0/0
- proto/port: specific then check tcp 22 and 3389 then other with icmp

here is equivalent command line:
```Bash
gcloud compute --project=qwiklabs-gcp-04-e7994c056ff5 firewall-rules create managementnet-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=managementnet --action=ALLOW --rules=tcp:22,tcp:3389,icmp --source-ranges=0.0.0.0/0
```

now do the same for privatenet:
```Bash
gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=privatenet --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
```
we can list all rules as needed:
```Bash
gcloud compute firewall-rules list --sort-by=NETWORK
```

### Task 2: Create VM instances
first one:
- name: managementnet-vm-1
- region: us-east4
- zone; us-east4-c
- series: E2
- machine type: e2-micro

Importantly, make seure to switch network to managementnet and subnet to mangaemntsubtenet-1

equiv code:
```Bash
gcloud compute instances create managementnet-vm-1 \
    --project=qwiklabs-gcp-04-e7994c056ff5 \
    --zone=us-east4-c \
    --machine-type=e2-micro \
    --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=managementsubnet-1 \
    --metadata=enable-osconfig=TRUE,enable-oslogin=true \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=124687507084-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --create-disk=auto-delete=yes,boot=yes,device-name=managementnet-vm-1,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260609,mode=rw,size=10,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ops-agent-policy=v2-template-1-7-0,goog-ec-src=vm_add-gcloud \
    --reservation-affinity=any \
&& \
printf 'agentsRule:\n  packageState: installed\n  version: latest\ninstanceFilter:\n  inclusionLabels:\n  - labels:\n      goog-ops-agent-policy: v2-template-1-7-0\n' > config.yaml \
&& \
gcloud compute instances ops-agents policies create goog-ops-agent-v2-template-1-7-0-us-east4-c \
    --project=qwiklabs-gcp-04-e7994c056ff5 \
    --zone=us-east4-c \
    --file=config.yaml \
&& \
gcloud compute resource-policies create snapshot-schedule default-schedule-1 \
    --project=qwiklabs-gcp-04-e7994c056ff5 \
    --region=us-east4 \
    --max-retention-days=14 \
    --on-source-disk-delete=keep-auto-snapshots \
    --daily-schedule \
    --start-time=09:00 \
&& \
gcloud compute disks add-resource-policies managementnet-vm-1 \
    --project=qwiklabs-gcp-04-e7994c056ff5 \
    --zone=us-east4-c \
    --resource-policies=projects/qwiklabs-gcp-04-e7994c056ff5/regions/us-east4/resourcePolicies/default-schedule-1
```

now to make the privagenet one with CLI:
```Bash
gcloud compute instances create privatenet-vm-1 --zone=us-east4-c --machine-type=e2-micro --subnet=privatesubnet-1
```
list all as needed with the following:
```Bash
gcloud compute instances list --sort-by=ZONE
```

### Task 3: Explore connectivitiyyu between vm instacnes
> im not gonna bother with this but it asks the following:
>>  Which instance(s) should you be able to ping from mynet-region-1-vm using internal IP addresses?\
**mynet-region-2-vm**

### Task 4: Create a VM instance with multiple network interfaces
Create the following VM:
- name: vm-appliance
- region: us-east4
- zone: us-east4-c
- series: E2
- e2-standard-4

use privatenet interface and privatesubnet-1 subnet\
THEN add another with mangamentnet....\
THEN add anohter mynetwork network with mynetwork subnetwork.



>  Note: In a multiple interface instance, every interface gets a route for the subnet that it is in. In addition, the instance gets a single default route that is associated with the primary interface eth0. Unless manually configured otherwise, any traffic leaving an instance for any destination other than a directly connected subnet will leave the instance via the default route on eth0. 


---
## Controlling Access
### Task 1: Create web servers
start with the *blue server*:
- name: blue
- region: us-west4
- zone; us-west4-b
- network tags: web-server

> note the tag will later allow it to have http enabled without manually doing a firewall for each one.

create another one called green the same but without the tag.

SSH into blue to start setting up nginx:
```Bash
sudo apt-get install nginx-light -y
```
```Bash
sudo nano /var/www/html/index.nginx-debian.html
# change "Welcome to nginx!" to "Welcome to the blue server!"
```

ssh into green to do the same respectively.

### Task 2: Create the firewall rule
We want to make a rule that applies to all with the *web-server* tag.\
Go to the *VPC network -> Firewall* tab and create the following rule:
- name: allow-http-web-server
- network: default
- taget: specified target tags
- target tags: web-server
- source filter: ipv4 ranges
- source ranges: 0.0.0.0/0
- protocols: tcp 80, other icmp

now create a test-vm:
```Bash
gcloud compute instances create test-vm --machine-type=e2-micro --subnet=default --zone=us-west4-b
```

> Im not going to but at this point we can prove that we can curl blue to see its web page but not green cause of the tag.

### Task 3: Explore network and security admin roles
Let's see current permissions.\
THe test-vm uses the [compute engine default service account](cloud.google.com/compute/docs/access/service-accounts#compute_engine_default_service_account) as it is what happens when an instance is created.\
Let's list or delete all firewall rules from test0vm.\
SSH into test-vm and run the following:
```Bash
gcloud compute firewall-rules list
```
This should fail.\
so will the following:
```Bash
gcloud compute firewall-rules delete allow-http-web-server
```
this of course is because of lack of permissions.

Let's fix this by making an appropraite service account with the correct permissions.\
Go to *IAM and Admin -> Service accounts* and create service account:
- name: Network-admin
- role: compute engine -> compute network admin

once done, click the 3 dots and *manage keys*.\
Add a key and download the JSON output.\
Name it *credentials.json*.
```JSON
{
  "type": "service_account",
  "project_id": "qwiklabs-gcp-01-3334e9e2e273",
  "private_key_id": "....",
  "private_key": "-----BEGIN PRIVATE\n...\nEND PRIVATE KEY-----\n",
  "client_email": "network-admin@qwiklabs-gcp-01-3334e9e2e273.iam.gserviceaccount.com",
  "client_id": "111143673333349264964",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/network-admin%40qwiklabs-gcp-01-3334e9e2e273.iam.gserviceaccount.com",
  "universe_domain": "googleapis.com"
}

```
> Note: network admin role is read-only for firewall rules and SSL certs
>> Those require SecurityAdmin role
>>> it instead allows for creation/modification/deletion of networking resources

SSH back into test-vm and upload the json file.\
Now we authroize its use:
```Bash
gcloud auth activate-service-account --key-file credentials.json
```
now we can list firewall rules:
```Bash
gcloud compute firewall-rules list
```
but we still can't delete.

let's go back to IAM and add security admin to the role.


---
## Configure an Application LOad balancer with google cloud
### Task 1: Configure HTTP and health check firewall rules
*VPC -> Firewall*\
*firewall rule*.

create two, one for all http and one for health check.\
here is normal one:
```Bash
gcloud compute --project=qwiklabs-gcp-01-1ff8cb2eaff9 firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
```
Here is health check one:
```Bash
gcloud compute --project=qwiklabs-gcp-01-1ff8cb2eaff9 firewall-rules create default-allow-health-check --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=http-server
```


### Task 2: Confuigure instance templates and create instance groups
*compute engine -> instance templates*

create one:
```Bash
gcloud compute instance-templates create us-central1-template --project=qwiklabs-gcp-01-1ff8cb2eaff9 --machine-type=e2-micro --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh,enable-oslogin=true --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=565248268349-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append --region=us-central1 --tags=http-server --create-disk=auto-delete=yes,boot=yes,device-name=us-central1-template,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260609,mode=rw,size=10,type=pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

then a regional mirror.\
to do so, we can go into the above template (assuming GUI) and hit *create similar* and just make a few tweaks.\
here is cli anyways:
```Bash
gcloud compute instance-templates create us-east4-template --project=qwiklabs-gcp-01-1ff8cb2eaff9 --machine-type=e2-micro --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default --metadata=startup-script-url=gs://spls/gsp215/gcpnet/httplb/startup.sh,enable-oslogin=true --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=565248268349-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append --region=us-east4 --tags=http-server --create-disk=auto-delete=yes,boot=yes,device-name=us-east4-template,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260609,mode=rw,size=10,type=pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```
the only changes we made were name and region

now go to *Compute engine -> instange groups* instead and create.
> here is where we can have auto-scaling instances!

first one for us central:
```Bash
gcloud beta compute instance-groups managed create us-central1-mig \
    --project=qwiklabs-gcp-01-1ff8cb2eaff9 \
    --base-instance-name=us-central1-mig \
    --template=projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/instanceTemplates/us-central1-template \
    --size=1 \
    --zones=us-central1-c,us-central1-f,us-central1-b \
    --target-distribution-shape=BALANCED \
    --instance-redistribution-type=none \
    --default-action-on-vm-failure=repair \
    --action-on-vm-failed-health-check=default-action \
    --on-repair-allow-changing-zone=yes \
    --force-update-on-repair \
    --standby-policy-mode=manual \
    --list-managed-instances-results=paginated \
    --target-size-policy-mode=individual \
&& \
gcloud beta compute instance-groups managed set-autoscaling us-central1-mig \
    --project=qwiklabs-gcp-01-1ff8cb2eaff9 \
    --region=us-central1 \
    --mode=on \
    --min-num-replicas=1 \
    --max-num-replicas=2 \
    --target-cpu-utilization=0.8 \
    --cpu-utilization-predictive-method=none \
    --cool-down-period=45 \
    --stabilization-period=600
```
now for us-east one (is same just region)

### Task 3: Configure the Application Load Balancer
we want to balance between our two backends (both *migs*)

Go to **ALL PRODUCTS** *networking -> network services -> load balancing* and create a load balancer.

> Note: This was a ton of settings, so only providing the CLI
```Bash
POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/securityPolicies
{
  "description": "Default security policy for: http-backend",
  "name": "default-security-policy-for-backend-service-http-backend",
  "rules": [
    {
      "action": "allow",
      "match": {
        "config": {
          "srcIpRanges": [
            "*"
          ]
        },
        "versionedExpr": "SRC_IPS_V1"
      },
      "priority": 2147483647
    },
    {
      "action": "throttle",
      "description": "Default rate limiting rule",
      "match": {
        "config": {
          "srcIpRanges": [
            "*"
          ]
        },
        "versionedExpr": "SRC_IPS_V1"
      },
      "priority": 2147483646,
      "rateLimitOptions": {
        "conformAction": "allow",
        "enforceOnKey": "IP",
        "exceedAction": "deny(403)",
        "rateLimitThreshold": {
          "count": 500,
          "intervalSec": 60
        }
      }
    }
  ]
}

POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/backendServices
{
  "backends": [
    {
      "balancingMode": "RATE",
      "capacityScaler": 1,
      "group": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/regions/us-central1/instanceGroups/us-central1-mig",
      "maxRatePerInstance": 50
    },
    {
      "balancingMode": "UTILIZATION",
      "capacityScaler": 1,
      "group": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/regions/us-east4/instanceGroups/us-east4-mig",
      "maxUtilization": 0.8
    }
  ],
  "cdnPolicy": {
    "cacheKeyPolicy": {
      "includeHost": true,
      "includeProtocol": true,
      "includeQueryString": true
    },
    "cacheMode": "CACHE_ALL_STATIC",
    "clientTtl": 3600,
    "defaultTtl": 3600,
    "maxTtl": 86400,
    "negativeCaching": false,
    "serveWhileStale": 0
  },
  "compressionMode": "DISABLED",
  "connectionDraining": {
    "drainingTimeoutSec": 300
  },
  "description": "",
  "enableCDN": true,
  "healthChecks": [
    "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/healthChecks/http-health-check"
  ],
  "iap": {
    "enabled": false
  },
  "ipAddressSelectionPolicy": "IPV4_ONLY",
  "loadBalancingScheme": "EXTERNAL_MANAGED",
  "localityLbPolicy": "ROUND_ROBIN",
  "logConfig": {
    "enable": true,
    "optionalMode": "EXCLUDE_ALL_OPTIONAL",
    "sampleRate": 1
  },
  "name": "http-backend",
  "portName": "http",
  "protocol": "HTTP",
  "securityPolicy": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/securityPolicies/default-security-policy-for-backend-service-http-backend",
  "sessionAffinity": "NONE",
  "timeoutSec": 30
}

POST https://compute.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/backendServices/http-backend/setSecurityPolicy
{
  "securityPolicy": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/securityPolicies/default-security-policy-for-backend-service-http-backend"
}

POST https://compute.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/urlMaps
{
  "defaultService": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/backendServices/http-backend",
  "name": "http-lb"
}

POST https://compute.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/targetHttpProxies
{
  "name": "http-lb-target-proxy",
  "urlMap": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/urlMaps/http-lb"
}

POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/forwardingRules
{
  "IPProtocol": "TCP",
  "ipVersion": "IPV4",
  "loadBalancingScheme": "EXTERNAL_MANAGED",
  "name": "http-lb",
  "networkTier": "PREMIUM",
  "portRange": "80",
  "target": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/targetHttpProxies/http-lb-target-proxy"
}

POST https://compute.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/targetHttpProxies
{
  "name": "http-lb-target-proxy-2",
  "urlMap": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/urlMaps/http-lb"
}

POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/forwardingRules
{
  "IPProtocol": "TCP",
  "ipVersion": "IPV6",
  "loadBalancingScheme": "EXTERNAL_MANAGED",
  "name": "http-lb-forwarding-rule",
  "networkTier": "PREMIUM",
  "portRange": "80",
  "target": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/targetHttpProxies/http-lb-target-proxy-2"
}

POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-1ff8cb2eaff9/regions/us-central1/instanceGroups/us-central1-mig/setNamedPorts
{
  "namedPorts": [
    {
      "name": "http",
      "port": 80
    }
  ]
}

POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-1ff8cb2eaff9/regions/us-east4/instanceGroups/us-east4-mig/setNamedPorts
{
  "namedPorts": [
    {
      "name": "http",
      "port": 80
    }
  ]
}
```


in order to do the next step, we need to make a siege vm to stress test.

```Bash
gcloud compute instances create siege-vm \
    --project=qwiklabs-gcp-01-1ff8cb2eaff9 \
    --zone=europe-west4-b \
    --machine-type=e2-medium \
    --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
    --metadata=enable-osconfig=TRUE,enable-oslogin=true \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=565248268349-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --create-disk=auto-delete=yes,boot=yes,device-name=siege-vm,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260609,mode=rw,size=10,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ops-agent-policy=v2-template-1-7-0,goog-ec-src=vm_add-gcloud \
    --reservation-affinity=any \
&& \
printf 'agentsRule:\n  packageState: installed\n  version: latest\ninstanceFilter:\n  inclusionLabels:\n  - labels:\n      goog-ops-agent-policy: v2-template-1-7-0\n' > config.yaml \
&& \
gcloud compute instances ops-agents policies create goog-ops-agent-v2-template-1-7-0-europe-west4-b \
    --project=qwiklabs-gcp-01-1ff8cb2eaff9 \
    --zone=europe-west4-b \
    --file=config.yaml \
&& \
gcloud compute resource-policies create snapshot-schedule default-schedule-1 \
    --project=qwiklabs-gcp-01-1ff8cb2eaff9 \
    --region=europe-west4 \
    --max-retention-days=14 \
    --on-source-disk-delete=keep-auto-snapshots \
    --daily-schedule \
    --start-time=01:00 \
&& \
gcloud compute disks add-resource-policies siege-vm \
    --project=qwiklabs-gcp-01-1ff8cb2eaff9 \
    --zone=europe-west4-b \
    --resource-policies=projects/qwiklabs-gcp-01-1ff8cb2eaff9/regions/europe-west4/resourcePolicies/default-schedule-1
```
we want to ssh in and install [siege](https://github.com/joedog/siege):
```Bash
sudo apt-get update && sudo apt-get -y install siege
```
```Bash
export LB_IP=[LB_IP_v4]
siege -c 150 -t120s http://$LB_IP
```
here is what the output looks like after closing it (ctrl+c):
```Bash
student-02-9e2ff1efd537@siege-vm:~$ siege -c 150 -t120s http://$LB_IP
New configuration template added to /home/student-02-9e2ff1efd537/.siege
Run siege -C to view the current settings in that file
^C
{       "transactions":                        10814,
        "availability":                       100.00,
        "elapsed_time":                        18.26,
        "data_transferred":                     1.63,
        "response_time":                        0.25,
        "transaction_rate":                   592.22,
        "throughput":                           0.09,
        "concurrency":                        149.09,
        "successful_transactions":             10814,
        "failed_transactions":                     0,
        "longest_transaction":                  2.42,
        "shortest_transaction":                 0.17
}
```

### Task 5: Denylist Siege VM
This one involves **Cloud Armor Security Policy**.\
Ensure you have the siege-vm external IP [34.13.144.146]
 
back to *ALL PRODUCTS -> Networking -> Network Securiry -> Cloud Armor* and create a policy.\
we are essentially allowing all BUT the siege IP

CLI POST request:
```Bash
POST https://compute.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/securityPolicies
{
  "adaptiveProtectionConfig": {
    "layer7DdosDefenseConfig": {
      "enable": false
    }
  },
  "description": "",
  "name": "denylist-siege",
  "rules": [
    {
      "action": "deny(403)",
      "description": "",
      "match": {
        "config": {
          "srcIpRanges": [
            "34.13.144.146/32"
          ]
        },
        "expr": {
          "description": "",
          "expression": "",
          "location": "",
          "title": ""
        },
        "versionedExpr": "SRC_IPS_V1"
      },
      "preview": false,
      "priority": 1000
    },
    {
      "action": "allow",
      "description": "Default rule, higher priority overrides it",
      "match": {
        "config": {
          "srcIpRanges": [
            "*"
          ]
        },
        "expr": {
          "description": "",
          "expression": "",
          "location": "",
          "title": ""
        },
        "versionedExpr": "SRC_IPS_V1"
      },
      "preview": false,
      "priority": 2147483647
    }
  ],
  "type": "CLOUD_ARMOR"
} && POST https://compute.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/backendServices/http-backend/setSecurityPolicy
{
  "securityPolicy": "projects/qwiklabs-gcp-01-1ff8cb2eaff9/global/securityPolicies/denylist-siege"
}
```

we can verify by SSH back into siege and trying to curl the load balancer.\
WE will get a 403.

---
##  Enhance Application Reliability and Scalability with Internal Load Balancing
### Task 1: Configure HTTP and health check firewall rules
We are given two asia-south1 subnetes in the *my-internal-app* with subnet-a and subnet-b.

Create firewall rrule:
```Bash
gcloud compute --project=qwiklabs-gcp-01-88c9b16942a7 firewall-rules create app-allow-http --direction=INGRESS --priority=1000 --network=my-internal-app --action=ALLOW --rules=tcp:80 --source-ranges=10.10.0.0/16 --target-tags=lb-backend
```
now health check:
```Bash
gcloud compute --project=qwiklabs-gcp-01-88c9b16942a7 firewall-rules create app-allow-health-check --direction=INGRESS --priority=1000 --network=my-internal-app --action=ALLOW --rules=tcp --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=lb-backend
```

### Task 2: Configure instance templates and create instance groups
*compute engine -> instance templates*

create one fore both subnets.

```Bash
gcloud compute instance-templates create instance-template-1 --project=qwiklabs-gcp-01-88c9b16942a7 --machine-type=e2-micro --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=subnet-a --metadata=^,@^startup-script=\#\!/bin/bash$'\n'apt-get\ update$'\n'apt-get\ install\ -y\ apache2\ php\ libapache2-mod-php$'\n'cat\ \<\<\'EOF\'\ \>\ /var/www/html/index.php$'\n'\<h1\>Internal\ Load\ Balancing\ Lab\</h1\>$'\n'\<h2\>Client\ IP\</h2\>$'\n'Your\ IP\ address\ :\ \<\?php\ echo\ \$_SERVER\[\'REMOTE_ADDR\'\]\;\ \?\>$'\n'\<h2\>Hostname\</h2\>$'\n'Server\ Hostname:\ \<\?php\ echo\ gethostname\(\)\;\ \?\>$'\n'\<h2\>Server\ Location\</h2\>$'\n'Region\ and\ Zone:\ \<\?php$'\n'\ \ \$ch\ =\ curl_init\(\)\;$'\n'\ \ curl_setopt\(\$ch,\ CURLOPT_URL,\ \"http://metadata.google.internal/computeMetadata/v1/instance/zone\"\)\;$'\n'\ \ curl_setopt\(\$ch,\ CURLOPT_HTTPHEADER,\ array\(\'Metadata-Flavor:\ Google\'\)\)\;$'\n'\ \ curl_setopt\(\$ch,\ CURLOPT_RETURNTRANSFER,\ 1\)\;$'\n'\ \ \$zone\ =\ curl_exec\(\$ch\)\;$'\n'\ \ \$parts\ =\ explode\(\'/\',\ \$zone\)\;$'\n'\ \ echo\ end\(\$parts\)\;$'\n'\?\>$'\n'EOF$'\n'rm\ -f\ /var/www/html/index.html$'\n'systemctl\ restart\ apache2,@enable-oslogin=true --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=850152442772-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append --region=asia-south1 --tags=lb-backend --create-disk=auto-delete=yes,boot=yes,device-name=instance-template-1,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260609,mode=rw,size=10,type=pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```
the second one is just called *2 and then uses subnet-b


nopw to make the instance groups

```Bash
gcloud beta compute instance-groups managed create instance-group-1 \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --base-instance-name=instance-group-1 \
    --template=projects/qwiklabs-gcp-01-88c9b16942a7/global/instanceTemplates/instance-template-1 \
    --size=1 \
    --zone=asia-south1-c \
    --default-action-on-vm-failure=repair \
    --action-on-vm-failed-health-check=default-action \
    --force-update-on-repair \
    --standby-policy-mode=manual \
    --list-managed-instances-results=pageless \
    --target-size-policy-mode=individual \
&& \
gcloud beta compute instance-groups managed set-autoscaling instance-group-1 \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --zone=asia-south1-c \
    --mode=on \
    --min-num-replicas=1 \
    --max-num-replicas=1 \
    --target-cpu-utilization=0.8 \
    --cpu-utilization-predictive-method=none \
    --cool-down-period=45 \
    --stabilization-period=600
```
```Bash
gcloud beta compute instance-groups managed create instance-group-2 \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --base-instance-name=instance-group-2 \
    --template=projects/qwiklabs-gcp-01-88c9b16942a7/global/instanceTemplates/instance-template-2 \
    --size=1 \
    --zone=asia-south1-b \
    --default-action-on-vm-failure=repair \
    --action-on-vm-failed-health-check=default-action \
    --force-update-on-repair \
    --standby-policy-mode=manual \
    --list-managed-instances-results=pageless \
    --target-size-policy-mode=individual \
&& \
gcloud beta compute instance-groups managed set-autoscaling instance-group-2 \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --zone=asia-south1-b \
    --mode=on \
    --min-num-replicas=1 \
    --max-num-replicas=1 \
    --target-cpu-utilization=0.8 \
    --cpu-utilization-predictive-method=none \
    --cool-down-period=45 \
    --stabilization-period=600
```
then we want to make an individual VM:
```Bash
gcloud compute instances create utility-vm \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --zone=asia-south1-c \
    --machine-type=e2-micro \
    --network-interface=network-tier=PREMIUM,private-network-ip=10.10.20.50,stack-type=IPV4_ONLY,subnet=subnet-a \
    --metadata=enable-osconfig=TRUE,enable-oslogin=true \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=850152442772-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --create-disk=auto-delete=yes,boot=yes,device-name=utility-vm,image=projects/debian-cloud/global/images/debian-12-bookworm-v20260609,mode=rw,size=10,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ops-agent-policy=v2-template-1-7-0,goog-ec-src=vm_add-gcloud \
    --reservation-affinity=any \
&& \
printf 'agentsRule:\n  packageState: installed\n  version: latest\ninstanceFilter:\n  inclusionLabels:\n  - labels:\n      goog-ops-agent-policy: v2-template-1-7-0\n' > config.yaml \
&& \
gcloud compute instances ops-agents policies create goog-ops-agent-v2-template-1-7-0-asia-south1-c \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --zone=asia-south1-c \
    --file=config.yaml \
&& \
gcloud compute resource-policies create snapshot-schedule default-schedule-1 \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --region=asia-south1 \
    --max-retention-days=14 \
    --on-source-disk-delete=keep-auto-snapshots \
    --daily-schedule \
    --start-time=08:00 \
&& \
gcloud compute disks add-resource-policies utility-vm \
    --project=qwiklabs-gcp-01-88c9b16942a7 \
    --zone=asia-south1-c \
    --resource-policies=projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1/resourcePolicies/default-schedule-1
```

### Task 3: COnfigure the internal load balancer
*ALL PRODUCTS -> networking -> network services -> load balancing*

*network load balancer -> passthrough load balancer -> internal*:
- name: my-ilb
- region: asia-south1
- network: my-internal-app

```Bash
POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1/backendServices
{
  "backends": [
    {
      "balancingMode": "CONNECTION",
      "failover": false,
      "group": "projects/qwiklabs-gcp-01-88c9b16942a7/zones/asia-south1-c/instanceGroups/instance-group-1"
    },
    {
      "balancingMode": "CONNECTION",
      "failover": false,
      "group": "projects/qwiklabs-gcp-01-88c9b16942a7/zones/asia-south1-b/instanceGroups/instance-group-2"
    }
  ],
  "connectionDraining": {
    "drainingTimeoutSec": 300
  },
  "description": "",
  "failoverPolicy": {},
  "healthChecks": [
    "projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1/healthChecks/my-ilb-health-check"
  ],
  "loadBalancingScheme": "INTERNAL",
  "logConfig": {
    "enable": false
  },
  "name": "my-ilb",
  "network": "projects/qwiklabs-gcp-01-88c9b16942a7/global/networks/my-internal-app",
  "networkPassThroughLbTrafficPolicy": {
    "zonalAffinity": {
      "spillover": "ZONAL_AFFINITY_DISABLED"
    }
  },
  "protocol": "TCP",
  "region": "projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1",
  "sessionAffinity": "NONE"
}

POST https://compute.googleapis.com/compute/beta/projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1/forwardingRules
{
  "IPAddress": "10.10.30.5",
  "IPProtocol": "TCP",
  "allowGlobalAccess": false,
  "backendService": "projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1/backendServices/my-ilb",
  "description": "",
  "ipVersion": "IPV4",
  "loadBalancingScheme": "INTERNAL",
  "name": "my-ilb-forwarding-rule",
  "networkTier": "PREMIUM",
  "ports": [
    "80"
  ],
  "region": "projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1",
  "subnetwork": "projects/qwiklabs-gcp-01-88c9b16942a7/regions/asia-south1/subnetworks/subnet-b"
}
```

---

## Challenge Lab
We are dealing with the folowing structure:
- GCP
    - acme-vpc
        -acme-mgmt-subnet
            - compute engine: bastion
                - SSH access: admin
        - acme-app-subnet
            - compute engine: juice-shop
                - SSH access: admin ; customers

We need to implement the following firewall rules:
- bastion does not have a public IP
- can only SSH to bastion and only via IAP
- can only SSH to juice-shop via bastion
- HTTP open to the world for juice-shop only

### Task 1: ICMP SSH access
```Bash
{
  "allowed": [
    {
      "IPProtocol": "tcp",
      "ports": [
        "22",
        "3389"
      ]
    }
  ],
  "creationTimestamp": "2026-06-15T17:47:11.771-07:00",
  "description": "",
  "direction": "INGRESS",
  "disabled": false,
  "enableLogging": false,
  "id": "7567911225315369952",
  "kind": "compute#firewall",
  "logConfig": {
    "enable": false
  },
  "name": "bastion-ssh-access",
  "network": "projects/qwiklabs-gcp-02-f14d3b9bb933/global/networks/acme-vpc",
  "priority": 1000,
  "selfLink": "projects/qwiklabs-gcp-02-f14d3b9bb933/global/firewalls/bastion-ssh-access",
  "sourceTags": [
    "permit-ssh-iap-ingress-ql-822"
  ]
}
```

> honestly this one was mostly just following the instructions on the page.
>> network tags important\
>> IAP access is its own IP range and is what SSH in console browser does under hood










