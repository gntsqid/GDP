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

### Task 1: Default Region and zone
let's set the default region.\
We want use-east1:
```Bash
gcloud config set compute/region us-east1
```
the set default zone to us-east1-b:
```Bash
gcloud config set compute/zone us-east1-b
```

### Task 2: Create multiple web server instances
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

### Task 3: Configure the load balancing service
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

### Task 4: Create the target pool and forwarding rule
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

### Task 5: Send traffic to our instances
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

### Next Steps
The lab is complete.\
Optionally, we can read the following [guide](https://docs.cloud.google.com/load-balancing/docs/network/setting-up-network-backend-service) if we want to learn more.

---

## Application Load Balancing

### Task 1: Set default region and zone

```Bash
gcloud config set compute/region us-central1
```
```Bash
gcloud config set compute/zone us-central1-b
```

### Task 2: Create VMs

```Bash
  gcloud compute instances create www1 \
    --zone=us-central1-b \
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
copy to do www2 and www3.

add firewall rule:
```Bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```

### Task 3: Create Application Load Balancer
Application Load Balancing is implemented on GFE (Google Front End).\
They are distributed globally and operate together using Google global network and control plane.\
We can configure URL rules to route some URLs to one set of instances and others to others.

Requested are alwats routed to instance group closest to user if it has enough capacity and is appropriate for rewquest.\
If closest group too full, then goes on to next closest.

To set upnthe load balancer with compute engine backend, we need to move our VMs to an instance group then manage the instance group for external application load balancing.\
We want to give the backends their own hostnames as well.

Let's create the load balancer template:
```Bash
gcloud compute instance-templates create lb-backend-template \
   --region=us-central1 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```
now the managed group via the template:
```Bash
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=us-central1-b
```
and finally the firewall health check rule:
```Bash
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```
This is an ingress rule that allows traffic from Google Cloud health checking systems (130.211.0.0/22 and 35.191.0.0/16).

Now, let's set up our global static external IP:
```Bash
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```
we can view it using:
```Bash
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```
```Bash
34.117.178.56
```

Time for health check creation:
```Bash
gcloud compute health-checks create http http-basic-check \
  --port 80
```
followed by creation of the backend service:
```Bash
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```
to which we then add our instance group as the backend for:
```Bash
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=us-central1-b \
  --global
```

Now we have our backend set up, let's make a URL mapto route incoming requests to the backend service:
```Bash
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
```
We can read more about URL maps [here](https://cloud.google.com/load-balancing/docs/url-map-concepts).\
In general, it is a Google Cloud config resource used to route requests to backend services/buckets.\
For example, we can use a single URL map to route requests to different destinations based on rules configured as seen below:
- Requests for https://example.com/video go to one backend service
- Requests for https://example.com/audio go to a different backend service.
- Requests for https://example.com/images go to a Cloud Storage backend bucket.
- Requests for any other host and path combination go to a default backend service.

Let's also make a [target HTTP Proxy](https://cloud.google.com/load-balancing/docs/target-proxies) so we can route requests to the URL map:
```Bash
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```
Then, we want to make a [global forwarding rule](https://cloud.google.com/load-balancing/docs/forwarding-rule-concepts) to route incoming requests to the proxy:
```Bash
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```
Note: A forwarding rule and its correspnding IP address represent the frontend config of a Google Cloud load balancer.\
We can learn more at the [forwarding rules overview page](https://cloud.google.com/load-balancing/docs/forwarding-rule-concepts).

### Task 4: Test traffic to instances
Let us leave the CLI for now and go back to the main console.\
We want to search for *load balancing* and click into the first result.\
The puts us in the **Network Services** parent.

Doing so, we will see our *web-map-http* load balancer up and running!

We can look into the load balancer and see it's IP.\
Going to that IP at HTTP will show "Page served from:..... like so:
```Bash
gwyn@firelink:~ $ curl 34.117.178.56
Page served from: lb-backend-group-4fws
```

> LAB DONE

## Use an Internal Application Load Balancer

### Task 1: Create a virtual environment

We start by installing it:
```Bash
sudo apt-get install -y virtualenv
```
then build:
```Bash
python3 -m venv venv
```
finally activate:
```Bash
source venv/bin/activate
```
if successful we will see our terminal change to the venv:
```Bash
(venv) student_03_7e1a5e3ae404@cloudshell:~ (qwiklabs-gcp-00-4887fd53051a)$ 
```

Next, we want to enable Gemini Code Assist in the IDE.\
This will allow us to use gemini APi in CLI:
```Bash
gcloud services enable cloudaicompanion.googleapis.com
```

We want to now go into the *Open Editor* by selecting the pencil icon at the top.\
This will create a git gists style coding environment.

We want to seelct **Cloud Code - No Project** the **Select a Google Cloud Project**.\
For this lab, we will use *qwiklabs....*

### Task 2: Create a backend managed instance group
Using a managed instance group we can automatically create and mainatin identical copies of our service.\
If one fails, the cloud replaces it.\
This increases reliability of our services.

To get started, we want to crete a startup script.\
We can do so in the cloud terminal we've been using (the venv):
```Bash
touch ~/backend.sh
```
We can then find the file in our cloud editor's file explorer.

here is the script to add:
```Bash
sudo chmod -R 777 /usr/local/sbin/
sudo cat << EOF > /usr/local/sbin/serveprimes.py
import http.server

def is_prime(a): return a!=1 and all(a % i for i in range(2,int(a**0.5)+1))

class myHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(s):
    s.send_response(200)
    s.send_header("Content-type", "text/plain")
    s.end_headers()
    s.wfile.write(bytes(str(is_prime(int(s.path[1:]))).encode('utf-8')))

http.server.HTTPServer(("",80),myHandler).serve_forever()
EOF
nohup python3 /usr/local/sbin/serveprimes.py >/dev/null 2>&1 &
```
now we are to look for the little Gemini star in the top-right.\
This will let us take code assist actions.\
We want to select **Explain This** ask it the following:
<pre>
As an Application Developer at Cymbal AI, explain the backend.sh startup script to a new team member. This script is used to run a small Python web server written in a Python file serveprimes.py. Provide a detailed breakdown of the script's key components and explain the function of each command.

For the suggested improvements, don't make any changes to the file's content.
</pre>
It gave me this:
<pre>
Gemini
Responding with 3 Flash Preview
Thinking

Welcome to the team at Cymbal AI! It’s great to have you onboard.

The backend.sh script you're looking at is a startup script designed to quickly deploy a lightweight Python-based microservice. Its primary job is to calculate whether a number is prime and serve that result over HTTP.

Here is a detailed breakdown of what the script does:
1. Permission Setup

shellscript
sudo chmod -R 777 /usr/local/sbin/

    sudo: Runs the command with administrative privileges.
    chmod -R 777: This recursively (-R) sets the permissions of the directory to be readable, writable, and executable (777) by any user.
    /usr/local/sbin/: This is a standard Linux directory for system administration binaries.
    Why it's here: The script needs to ensure it has the rights to create and modify files in this system directory.

2. Creating the Python Server (serveprimes.py)

The script uses a "Here Document" (Heredoc) to write the Python code directly into a file.

shellscript
sudo cat << EOF > /usr/local/sbin/serveprimes.py
... [Python Code] ...
EOF

    cat << EOF: This tells the shell to take everything until the next EOF string and pass it as input to the cat command.
    > /usr/local/sbin/serveprimes.py: This redirects that input into a new file named serveprimes.py.

The Python Logic Inside:

    is_prime(a): A concise function that checks if a number is prime by testing for divisors up to the square root of the number.
    myHandler(http.server.BaseHTTPRequestHandler): This class defines how the server responds to requests.
        do_GET(s): When a user makes a GET request (e.g., visiting http://<IP>/7), it sends a 200 OK response.
        s.path[1:]: This strips the leading slash from the URL path to get the number the user wants to check.
        s.wfile.write(...): It converts the boolean result of is_prime to a string and sends it back as the response body.
    HTTPServer(("",80),myHandler): This initializes the server on all available network interfaces ("") on port 80 (the standard HTTP port).

3. Execution and Backgrounding

shellscript
nohup python3 /usr/local/sbin/serveprimes.py >/dev/null 2>&1 &

    nohup: Short for "no hangup." This allows the Python process to keep running even after you log out of the shell session.
    python3: The interpreter used to run the script.
    >/dev/null 2>&1: This is a redirection trick. It sends all standard output (>) and error messages (2>&1) to /dev/null, which is essentially a "black hole" on Linux. This keeps the logs silent.
    &: This symbol at the end tells the shell to run the command in the background, allowing the script to finish and return control to the user immediately.

Summary for the Team

In short, this script sets up the environment, writes the application logic to disk, and starts a background web server on port 80. While it's very effective for quick prototyping or internal tools at Cymbal AI, keep in mind that for production, we typically look for more restrictive permissions than 777 and robust error handling for input that isn't a valid integer.
</pre>


time to make our instance template:
```Bash
gcloud compute instance-templates create primecalc \
--metadata-from-file startup-script=backend.sh \
--no-address --tags backend --machine-type=e2-medium
```
the *--no-address* indicates it won't be publicly accessible.

After that, let's get a firewall rule opening up port 80:
```Bash
gcloud compute firewall-rules create http --network default --allow=tcp:80 \
--source-ranges 10.142.0.0/20 --target-tags backend
```


time to make a new *backend* instance group:
```Bash
gcloud compute instance-groups managed create backend \
--size 3 \
--template primecalc \
--zone us-east1-b
```

We can verify their creation in the cloud console with *Compute Engine* -> *VM Instances*

### Task 3: Set up the internal load balancer
We now want to create a single private VIP entracne for our internal service to alow other internal application to reach our service reliably without needing to know which specific backend VM is active or avaialble\
To do so, we have set up an internal load balancer and connected it to our instance group.
These are made of 3 parts:
- forwarding rule: private IP to forward traffic to backedn
- backend service: determines how load balancer distributes traffic to VMs (also health check)
- health check: a continuous monitor/health check of backednd VMs to determine to is healthy enough to recieve traffic

time to make the health check:
```Bash
gcloud compute health-checks create http ilb-health --request-path /2
```
then a backend service.\
our sertvice will calculate prime numbers (or something):
```Bash
gcloud compute backend-services create prime-service \
--load-balancing-scheme internal --region=us-east1 \
--protocol tcp --health-checks ilb-health
```
conect instance group to the service:
```Bash
gcloud compute backend-services add-backend prime-service \
--instance-group backend --instance-group-zone=us-east1-b \
--region=us-east1
```
followed by creating the forwarding rule:
```Bash
gcloud compute forwarding-rules create prime-lb \
--load-balancing-scheme internal \
--ports 80 --network default \
--region=us-east1 --address 10.142.0.10 \
--backend-service prime-service
```


### Task 4: Test the load balancer
let's set up a simple isntance in the cloud.\
We are doing so because we want to do an internal test.\
This means we need a VM inside the same network as the application siunce its only accessible within our private cloud network.

We can make one with the following:
```Bash
gcloud compute instances create testinstance \
--machine-type=e2-standard-2 --zone us-east1-b
```
then we ssh into it:
```Bash
gcloud compute ssh testinstance --zone us-east1-b
```


let's get some test curls:
```Bash
curl 10.142.0.10/2
```
```Bash
curl 10.142.0.10/4
```
```Bash
curl 10.142.0.10/5
```

we will get true or false if the number is prime or not.

exit once testing is over and delete the instance:
```Bash
# you will need to confirm with 'y' after
gcloud compute instances delete testinstance --zone=us-east1-b
```

### Task 5: Create a public-facing web server
let's make a new file:
```Bash
touch ~/frontend.sh
```
we want to have a public facing application so we can leverage internal iservices.\
We will essentially make our prime number calculator display a matric of primes.

add the following script:
```Bash
sudo chmod -R 777 /usr/local/sbin/
sudo cat << EOF > /usr/local/sbin/getprimes.py
import urllib.request
from multiprocessing.dummy import Pool as ThreadPool
import http.server
PREFIX="http://10.142.0.10/" #HTTP Load Balancer
def get_url(number):
    return urllib.request.urlopen(PREFIX+str(number)).read().decode('utf-8')
class myHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(s):
    s.send_response(200)
    s.send_header("Content-type", "text/html")
    s.end_headers()
    i = int(s.path[1:]) if (len(s.path)>1) else 1
    s.wfile.write("<html><body><table>".encode('utf-8'))
    pool = ThreadPool(10)
    results = pool.map(get_url,range(i,i+100))
    for x in range(0,100):
      if not (x % 10): s.wfile.write("<tr>".encode('utf-8'))
      if results[x]=="True":
        s.wfile.write("<td bgcolor='#00ff00'>".encode('utf-8'))
      else:
        s.wfile.write("<td bgcolor='#ff0000'>".encode('utf-8'))
      s.wfile.write(str(x+i).encode('utf-8')+"</td> ".encode('utf-8'))
      if not ((x+1) % 10): s.wfile.write("</tr>".encode('utf-8'))
    s.wfile.write("</table></body></html>".encode('utf-8'))
http.server.HTTPServer(("",80),myHandler).serve_forever()
EOF
nohup python3 /usr/local/sbin/getprimes.py >/dev/null 2>&1 &
```

create a new VM instance to host the frontend:
```Bash
gcloud compute instances create frontend --zone=us-east1-b \
--metadata-from-file startup-script=frontend.sh \
--tags frontend --machine-type=e2-standard-2
```
get the firewall to open up port 80 fromy any/any:
```Bash
gcloud compute firewall-rules create http2 --network default --allow=tcp:80 \
--source-ranges 0.0.0.0/0 --target-tags frontend
```

go find the external IP in the cloud console:
```Bash
35.190.184.180
```
connecting in browser works!

> done

---
## Implement Load Blanacing on Compute Engine: Challenge Lab
This is the final "quiz" to get the points for load balancing stuff.\
In this scenario, I am a junior cloud engineer working as part of a team of engineers assigned to provicde network functionality to compute engin VM instances on google cloud VPC network.
> We can't create VM instances, containers, or app engine applications without a VPC
>> Each Google CLoud project has a default one to get us started

In order for us to support load balancing, we need to know the diference between a network load balancer and an HTTP load balancer as well as how to confgigure both.

Our challenge therefore is the following:
- create multiple web serverf instances with firewall rules
- configure load balancing service
- create HTTP load balancer

to do so, we need to follow a few standards as follows:
- all resources in the **Region** region and **Zone** zone

### Task 1: Create multiple web server instances
We may want to set default region and zone again:
```Bash
gcloud config set compute/region X
gcloud config set compute/zone Y
```

A reminder, to create VMs, we can use the following:
```Bash
  gcloud compute instances create web3 \
    --zone=europe-west4-a \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: web3</h3>" | tee /var/www/html/index.html'
```
we want a firewall rule:
```Bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```


### Task 2: Configure the load balancing service
Start by creating external IP:
```Bash
gcloud compute addresses create network-lb-ip-1 \
  --region europe-west4
```
add legacy health check:
```Bash
gcloud compute http-health-checks create basic-check
```
create target pool w/ health check added:
```Bash
gcloud compute target-pools create www-pool \
  --region europe-west4 --http-health-check basic-check
```
add instances to pool:
```Bash
gcloud compute target-pools add-instances www-pool \
    --instances web1,web2,web3
```
create forwarding rule:
```Bash
gcloud compute forwarding-rules create www-rule \
    --region europe-west4 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

### Task 3: Create an HTTP load balancer
> This will mimic our Application one

create the template:
```Bash
gcloud compute instance-templates create lb-backend-template \
   --region=europe-west4 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-12 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```
managed group via the template:
```Bash
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=europe-west4-a
```
firewall health check rule:
```Bash
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```
global static external IP:
```Bash
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```
we can view it using:
```Bash
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```

health check creation:
```Bash
gcloud compute health-checks create http http-basic-check \
  --port 80
```
followed by creation of the backend service:
```Bash
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```
to which we then add our instance group as the backend for:
```Bash
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=europe-west4-a \
  --global
```

Now we have our backend set up, let's make a URL mapto route incoming requests to the backend service:
```Bash
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
```
route requests to the URL map:
```Bash
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```
global forwarding rule to route incoming requests to the proxy:
```Bash
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```

> That should be everything...
>> testing now...

> SUCCESS!






























