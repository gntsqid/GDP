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


































