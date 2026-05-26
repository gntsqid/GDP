# Associate Cloud Engineer: Load Balancing
This is the first course/lab as part of the Get Certified for A.C.E.. 

A few import notes:
- no pausing
- run lab in incognito window
- only use provided student account

## Load Balancing
There are several ways to [load balance in Google Cloud](https://docs.cloud.google.com/load-balancing/docs/load-balancing-overview#a_closer_look_at_cloud_load_balancers).\
This course focuses on [regional external passthrough network load balancers](https://docs.cloud.google.com/load-balancing/docs/network/networklb-backend-service).

To start, let's test out the cloud shell.\
We will be doing this to verify we are signed in to the wright Project.\
We should see the following *Project_ID*: qwiklabs-gcp-04-cbeaaaa13542

Open up the Cloud Shell using the top-right corner and then type we will see in our terminal session that we are indeed in the right project.\
Next, type the following to see the active account:
```Bash
gcloud auth list
```
```Bash
Credentialed Accounts

ACTIVE: *
ACCOUNT: student-04-d3957a401ff3@qwiklabs.net

To set the active account, run:
    $ gcloud config set account `ACCOUNT`
```

same for projects:
```Bash
gcloud config list project
```
```Bash
[core]
project = qwiklabs-gcp-04-cbeaaaa13542

Your active configuration is: [cloudshell-18417]
[environment: untagged] Read more to tag: g.co/cloud/project-env-tag.
```

## Task 1: Default Region and zone
let's set the default region.\
We want use-east1:
```Bash
gcloud config set compute/region us-east1
```
the set default zone to us-east1-b:
```Bash
gcloud config set compute/zone us-east1-b
```

## Task 2: Create multiple web server instances
We want to load balance 3 compute engine VM instances running Apache.\
We also want a firewall rule allowing HTTP traffic to reach the instances.

We can continue to use the CLI to do so.

Create the first machine:
```Bash
  gcloud compute instances create www1 \
    --zone=us-east1-b \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```

We repeat for the other 2 (www2 and www3.\
A success looks like this:
```Bash
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/zones/us-east1-b/instances/www1].
NAME: www1
ZONE: us-east1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE: 
INTERNAL_IP: 10.142.0.2
EXTERNAL_IP: 104.196.189.149
STATUS: RUNNING
```

Finally, we want to add our firewall rule:
```Bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```
```Bash
Creating firewall...working..Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/global/firewalls/www-firewall-network-lb].                                                                                                                                                                       
Creating firewall...done.                                                                                                                                                                                                                                                                                                          
NAME: www-firewall-network-lb
NETWORK: default
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: tcp:80
DENY: 
DISABLED: False
```

Let's check on our VMs:
```Bash
gcloud compute instances list
```
```Bash
NAME: www1
ZONE: us-east1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE: 
INTERNAL_IP: 10.142.0.2
EXTERNAL_IP: 104.196.189.149
STATUS: RUNNING

NAME: www2
ZONE: us-east1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE: 
INTERNAL_IP: 10.142.0.3
EXTERNAL_IP: 34.138.38.34
STATUS: RUNNING

NAME: www3
ZONE: us-east1-b
MACHINE_TYPE: e2-small
PREEMPTIBLE: 
INTERNAL_IP: 10.142.0.4
EXTERNAL_IP: 34.148.185.37
STATUS: RUNNING
```
specifically for this lab, we want the **external IP**.

I'm going to try to curl one of them.
```Bash
StatusCode        : 200
StatusDescription : OK
Content           :
                    <h3>Web Server: www3</h3>

RawContent        : HTTP/1.1 200 OK
                    Keep-Alive: timeout=5, max=100
                    Connection: Keep-Alive
                    Accept-Ranges: bytes
                    Content-Length: 27
                    Content-Type: text/html
                    Date: Tue, 26 May 2026 23:23:40 GMT
                    ETag: "1b-652c0c22188bf...
Forms             : {}
Headers           : {[Keep-Alive, timeout=5, max=100], [Connection, Keep-Alive], [Accept-Ranges, bytes],
                    [Content-Length, 27]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 27
```
huzzah!

## Task 3: Configure the load balancing service
When configuring the load balancing service, we want to ensure our VM instances recieved packets destined for the static external IP address that I configure.
> instances made with compute enginge are automatically configured to handle this IP

Let's create it now:
```Bash
gcloud compute addresses create network-lb-ip-1 \
  --region us-east1
```
```Bash
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1/addresses/network-lb-ip-1].
```
Let us now add a legacy HTTP health check resource:
```Bash
gcloud compute http-health-checks create basic-check
```
```Bash
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/global/httpHealthChecks/basic-check].
NAME: basic-check
HOST: 
PORT: 80
REQUEST_PATH: /
```

## Task 4: Create the target pool and forwarding rule
A target pool is a group of backend instances receiving incoming traffic from external passtrhough NLBs.\
All backend instances of a target pool must reside in the same Google Cloud region.

Let us creat one now and use the health check, a requirement for it to function:
```Bash
gcloud compute target-pools create www-pool \
  --region us-east1 --http-health-check basic-check
```
```Bash
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1/targetPools/www-pool].
NAME: www-pool
REGION: us-east1
SESSION_AFFINITY: NONE
BACKUP: 
HEALTH_CHECKS: basic-check
```
then we add our instances to the pool:
```Bash
gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3
```
```Bash
Updated [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1/targetPools/www-pool].
```
Now we can add the forwarding rule.\
This will specify how the network traffic is routed to the backend load balancer services as defined [here](https://docs.cloud.google.com/load-balancing/docs/forwarding-rule-concepts).

Create one now:
```Bash
gcloud compute forwarding-rules create www-rule \
    --region  us-east1 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```
```Bash
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1/forwardingRules/www-rule].
```

## Task 5: Send traffic to our instances
Now that load balancing is configured, we can start sending traffic to the forwarding rule.\
Let's grab the external IP address of the www-rule forwarding rule we want our load balancer to use:
```Bash
gcloud compute forwarding-rules describe www-rule --region us-east1
```
```Bash
IPAddress: 34.139.179.216
IPProtocol: TCP
creationTimestamp: '2026-05-26T16:30:47.578-07:00'
description: ''
fingerprint: uCoxHTP3fjQ=
id: '32786692448162760'
kind: compute#forwardingRule
labelFingerprint: 42WmSpB8rSM=
loadBalancingScheme: EXTERNAL
name: www-rule
networkTier: PREMIUM
portRange: 80-80
region: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1
selfLink: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1/forwardingRules/www-rule
selfLinkWithId: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1/forwardingRules/32786692448162760
target: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-04-cbeaaaa13542/regions/us-east1/targetPools/www-pool
```
now let's access the external IP and attach it to a variable:
```Bash
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region us-east1 --format="json" | jq -r .IPAddress)
```
then show it:
```Bash
echo $IPADDRESS
```
```Bash
34.139.179.216
```
We now want to test from another machine the connection to it:
```Bash
while true; do curl -m1 $IPADDRESS; done
```
```Bash
gwyn@firelink:~ $ while true; do curl -m1 34.139.179.216; done

<h3>Web Server: www3</h3>

<h3>Web Server: www3</h3>

<h3>Web Server: www3</h3>

<h3>Web Server: www2</h3>

<h3>Web Server: www1</h3>

<h3>Web Server: www3</h3>

<h3>Web Server: www2</h3>

<h3>Web Server: www3</h3>

<h3>Web Server: www3</h3>

<h3>Web Server: www3</h3>
```
It worked!

## Next Steps
The lab is complete.\
Optionally, we can read the following [guide](https://docs.cloud.google.com/load-balancing/docs/network/setting-up-network-backend-service) if we want to learn more.











