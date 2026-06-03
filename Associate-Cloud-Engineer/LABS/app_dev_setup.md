# Set up and App Dev Environment on Google Cloud
Welcome to week 2.\
This time around we are going to be touching the following:
- cloud storage
- identity and access management
- cloud functions
- pub/sub

## Cloud Storage Qwik Start - Google Cloud Console

### Task 1: Create a bucket
Go to *cloud storage -> buckets* then click create.\
For the name, we are told to use our project ID as it is always unique:
```Bash
qwiklabs-gcp-01-e3a27813d11b
```
we will be using *europe-west1*.\
for default class use *standard*.\
for access control use *uniform* and also **uncheck** *enforece public access prevention on this bucket*.\
Then click create!

### Task 2: upload an object into the bucket
let's download an image and upload it to the bucket.\
They give us 'kitten.png'.\
In the console, go to *Objects -> Upload -> Upload files* and upload the image.

> note: object names must be unique within a bucket (similar to regular ol' folders)

### Task 3: share a bucket publicly
go to permissions and change view to *View by Principles* if not already.\
Then click *Grant Access -> Add principals*.\
Enter in ***allUsers*** and select *cloud storage -> storage object viewer* for the role.\
Then save!

If it worked, we can see it in our bucket by going to the object and grabbing its public URL:
```Bash
https://storage.googleapis.com/qwiklabs-gcp-01-e3a27813d11b/kitten.png
```

## CLI/SDK
This is essentially going over the Cloud Shell terminal.

### Task 1: Create a bucket
we can make a bucket quite easy with CLI commands
```Bash
gcloud storage buckets create gs://<YOUR-BUCKET-NAME>
```

let's try adding something.\
Since the cloud terminal is an ephemeral but real terminal, we can download a file to it:
```Bash
curl https://upload.wikimedia.org/wikipedia/commons/thumb/a/a4/Ada_Lovelace_portrait.jpg/800px-Ada_Lovelace_portrait.jpg --output ada.jpg
```
now throw it in the bucket:
```Bash
gcloud storage cp ada.jpg gs://qwiklabs-gcp-01-4067ea8b8eb1
```
> note: we can tab-complete the bucket name

then let's remove the file we downloaded:
```Bash
rm ada.jpg
```


### Task 3: Download an object from your bucket
```Bash
gcloud storage cp -r gs://YOUR-BUCKET-NAME/ada.jpg .
```

### Task 4: Copy an object to a folder in the bucket
```Bash
gcloud storage cp gs://qwiklabs-gcp-01-4067ea8b8eb1/ada.jpg gs://qwiklabs-gcp-01-4067ea8b8eb1/image-folder/
```

### Task 5: List contents of a bucket or folder
```Bash
gcloud storage ls gs://YOUR-BUCKET-NAME
```

### Task 6: List details for an object
```Bash
gcloud storage ls -l gs://YOUR-BUCKET-NAME/ada.jpg
```

### Task 7: Make an object publicly accessible
```Bash
gcloud storage objects update gs://qwiklabs-gcp-01-4067ea8b8eb1/ada.jpgg --add-acl-grant=entity=allUsers,role=READER
```

### Task 8: Remove public access of an object
```Bash
gcloud storage objects update gs://qwiklabs-gcp-01-4067ea8b8eb1/ada.jpg --remove-acl-grant=allUsers
```

and now to clean up, let's delete the object:
```Bash
gcloud storage rm gs://YOUR-BUCKET-NAME/ada.jpg
```

---
## Cloud IAM
This one is a bit special in that we have to sign in as both student-02-fc28e784f00b@qwiklabs.net and student-02-f854aac1c8ac@qwiklabs.net in separate windows.

### Task 1: Explore the IAM console and project level roles
This will be done as user 1 to start.\
Let's go to Nav -> *IAM & Admin -> IAM*.\
From here, *Grant Access* at the top then scroll down to basic.\
We will see role section with 4 options:
- browser: read access to browse resource hierarchy including folders, org, and policies but not resources
- editor: all permissions of viewer plus allowed to modify states
- owner: all permissions of editor plus allowed to managed roles and permissions for project + resource; also project billing setup
- viewer: read-only that do not affect state of resources

These are known as [primitive roles](cloud.google.com/iam/docs/understanding-roles#primitive_roles).\
It can be noted that *user 1 has owner permissions* to this project.

Let's switch to user 2 and get to teh same menu.\
If we look at the *View by Principals*, we can see both users.\
It is noticeable that user 2 **only has viewer** but user 1 has editor, owner, bigquery admin, and app engine admin!\
User 2 has that same *Grant access* button grayed out.

### Task 2: Prepare a cloud storage bucket for testing
Let's use user 1.\
Make a bucket and ensure it is multi-region.\
Also ensure pblic access is prevented.

now upload a single file.

Now as user 2, go check if they can see that bucket/file.
> yep, they can download it too

### Task 3: Remove project access
Back as user 1, lets go into IAM again.\
Select the pencil on user 2 to edit and delete their viewer role.\
Since they had no other roles, they are now missing entirely!

It can take 80 seconds to work, so wait a moment and then see if user 2 can get into that bucket still.\
Finally I see **Access Denied** when inside the bucket.

### Task 4: Add cloud storage permissions
Let's try as user 1 to now grant access to user 2.\
Let's give them the role *Cloud Storage -> Storage Obejct Viwer*

While user 2 can no longer see project stuff, they can now still access the bucket!
This means they can even do it in the CLI:
```Bash
gcloud storage ls gs://[YOUR_BUCKET_NAME]
```

> Done

## Cloud Monitoring
Cloud monitoring, as it suggests, allows us to see performance, uptime, and overall health of our cloud apps.\
It collectes metrics, events, and emtadata from Google Cloud, AWS, hoested uptime probnes, and more such as Cassandra, Nginx, and Elasticsearch.

We will be installing monitoring and loggins some agents.

### Task 1: Create a compute engine instance
We are just goign to do this by hand in the console.

It will be called *lamp-1-vm*.

### Task 2: Add Apache2 HTTP Server to our instance
We are going to SSH in this time.\
We can do so via the console.

Once in, let's install apache:
```Bash
sudo apt-get update

sudo apt-get install apache2 php7.0
```
> I couldnt SSH in with the browser but I can with CLI:
```Bash
gcloud compute ssh lamp-1-vm --zone=asia-east1-b
```
>> This still failed!\
Here is a good troubleshoot command:
```Bash
gcloud compute ssh lamp-1-vm \
  --project=qwiklabs-gcp-01-5314f7a322ef \
  --zone=asia-east1-b \
  --troubleshoot
```
> Also here is how to read my public key:
```Bash
# Get your public key
cat ~/.ssh/google_compute_engine.pub
```

alright after disabling os something in metadata i am in.

---

now that apache is installed, get it running:
```Bash
sudo service apache2 restart
```
going now to it's public IP shows the Apache default page.


It's time to set up a *Monitoring Metrics Scope* tied to our project.\
Go to -> **Monitoring**.

We want to install some logging agents via our SSH session.\
Here is the install script command:
```Bash
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```
> verify with status check:
```Bash
sudo systemctl status google-cloud-ops-agent"*"
```

### Task 3: Create uptime check
We want to go to the **Monitoring -> Uptime Check** menu and create a new one.\
We will use the following settings:
- protocol: HTTP
- Respurce Type: URL
- Hostname: <external IP>
- Frequency: 1 min

Let's call it *"Lamp Uptime Check"*.

Test and create.

### Task 4: Create an alerting policy
Let's go to **Monitoring -> Alerting**.\
We want to crete a new one.

In *Select a Metric*, uncheck *Active* then select *VM Instance -> Interface -> Network Traffic*.\
Go next.\
For threshold, select *above threshold* amd set it to 500 with a 1 min retest.\
For notification channel select *Manage notification channels*.\
Here we want to *Create Email Channel* and enter our email info.

We can then click into it after and set stuff like body and display name.\
We want *Alert Policy* to be *Inbound Traffic Alert*.

### Task 5: Create a dashboard and chart
Go to Metrics -> *Dashboard* and create custom.

Add a LINE widget and name it *CPU Load*.\
Search fo rit in the *add a metric* menu.

> we are done at this point but the optional rest is important too

### Task 6: View our logs
Now lets go to Menu -> *Logging -> Logs Excplorer*.\
Here we can see essentially all logs.

We want to change from *All resources* to our VM.

### Task 7: Check uptime check results and triggered alerts
Let's shut the VM off and turn it back on after a bit.\
We should get an alert and see major changes in logs as well as our dashboard.


---
## Cloud Run Functions
These are awesome mini automation APIs.

### Task 1: Create a function
Let's go to *Cloud Run -> Services* and *write a function*.

For the settings, we want:
- name: gcfunction
- auth: allow public access
- memory: (default)
- execution env: second gen
- revision scaling: max instances = 5

If something asks us to validate APIs, do so and hit enable.

If it works, we will be set up with a cool IDE console for our function!\
Our current function shows as this:
```javascript
// note: This is javascript
const functions = require('@google-cloud/functions-framework');

functions.http('helloHttp', (req, res) => {
  res.set('Content-Type', 'text/plain');
  res.send(`Hello ${req.query.name || req.body.name || 'World'}!`);
});
```

### Task 2:
deploy\
normally we'd make changes and redeploy but no need for this.\
The redeploy takes some time (about 3 min for me)

### Task 3: Test the function
Hit the test button located in the top bar.\
We can adjust the payload as we please.\
The default is as follows:
```JSON
{
  "name": "Developer"
}
```
let's change it to:
```
{
  "name": "Developer",
  "message":"Hello World!"
}
```
we even get a nice test CLI command:
```Bash
curl -X POST "https://gcfunction-454092789026.us-east4.run.app" \
-H "Authorization: bearer $(gcloud auth print-identity-token)" \
-H "Content-Type: application/json" \
-d '{
  "message":"Hello World!"
}'
```

### Task 4: View Logs
In the *Service Details* page, we can go to *Observability -> Logs*.

> done

---
## Cloud Run Functions (CLI vers.)

### Task 1: Create a function
```Bash
# region
gcloud config set run/region europe-west3

# directory to host function
mkdir gcf_hello_world && cd $_

# create index.js and package.json
touch index.js
touch package.json
```
fill index.js:
```javascript
const functions = require('@google-cloud/functions-framework');

// Register a CloudEvent callback with the Functions Framework that will
// be executed when the Pub/Sub trigger topic receives a message.
functions.cloudEvent('helloPubSub', cloudEvent => {
  // The Pub/Sub message is passed as the CloudEvent's data payload.
  const base64name = cloudEvent.data.message.data;

  const name = base64name
    ? Buffer.from(base64name, 'base64').toString()
    : 'World';

  console.log(`Hello, ${name}!`);
});
```
fill package.json:
```JSON
{
  "name": "gcf_hello_world",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "@google-cloud/functions-framework": "^3.0.0"
  }
}
```
finally npm install:
```Bash
npm install
```
> i got a ton of vulns despite the thing saying I wujldnt but its fine

### Task 2: Deploy the function
```Bash
gcloud functions deploy nodejs-pubsub-function \
  --gen2 \
  --runtime=nodejs22 \
  --region=europe-west3 \
  --source=. \
  --entry-point=helloPubSub \
  --trigger-topic cf-demo \
  --stage-bucket qwiklabs-gcp-04-cfd6cab10c3e-bucket \
  --service-account cloudfunctionsa@qwiklabs-gcp-04-cfd6cab10c3e.iam.gserviceaccount.com \
  --allow-unauthenticated
```
> If you get a service account serviceAccountTokenCreator notification select "n". 

verify it works:
```Bash
gcloud functions describe nodejs-pubsub-function \
  --region=europe-west3 
```


### Task 3: Test the function
```Bash
gcloud pubsub topics publish cf-demo --message="Cloud Function Gen2"
```

### Task 4: View logs
```Bash
gcloud functions logs read nodejs-pubsub-function \
  --region=europe-west3 
  ```

  > done

  ---
  ## Pub/Sub
  this is a messaging service for exchaning event data among appas and services.

  ### Task 1: Setting up
  Go to *View all products -> Analytics -> Pub/Sub -> Topics* then create a topic.

  Let's call it "MyTopic", leaving all else default.


### Task 2: Create a subscription
In the same Topics menu move to -> subscriptions.\
create one.

call it *MySub* and then set topic name to *projects/<PROJECT_ID>/topics/MyTopic*\
delivery type should be set to *pull*.\
all else default and creat


> technically done

### Task 3: knowledge check

<pre>
A publisher application creates and sends messages to a TOPIC. Subscriber applications create a SUBSCTIPTION to a topic to receive messages from it.
</pre>

### Task 4: Public a message to a topic
Going back to the topics page, open up our *MyTopic*.\
In here, go to *Messages* and publish a message.

Let's just do a quick *Hello World*.

### Task 5: View the message
Now, go back to our *MySub* and pull the message.

We can do this with the CLI:
```Bash
gcloud pubsub subscriptions pull --auto-ack MySub
```

or in the menu, enable ACK messages adnd check the *messages* tab in our *MySub*.

> for some reason, the online took forever and didnt even work....
>> therefore use the CLI

> DONE

---
## Pub/Sub (CLI vers.)

### Task 1: Pub/Sub Topics
Create the topic:
```Bash
gcloud pubsub topics create myTopic
```
> this failed due to some error
>> apparently I need the project....let's do this
```Bash
# get project
gcloud config list project
```
```Bash
# add --project flag (qwiklabs-gcp-02-ec616a297f82)
gcloud pubsub topics create myTopic --project qwiklabs-gcp-02-ec616a297f82
```

list all topics:
```Bash
gcloud pubsub topics list
```

we can cleanup and delete (**but dont actually**):
```Bash
gcloud pubsub topics delete myTopic
```

### Task 2: Pub/Sub Subscriptions
Make a subscription tied to our topic:
```Bash
gcloud  pubsub subscriptions create --topic myTopic mySubscription
```

list all subscriptions tied to our topic:
```Bash
gcloud pubsub topics list-subscriptions myTopic
```

### Task 3: Pub/Sub
Let's publish a message:
```Bash
gcloud pubsub topics publish myTopic --message "Hello"
```

now we want to pull a message:
```Bash
gcloud pubsub subscriptions pull mySubscription --auto-ack
```

### Task 4: Pub/Sub pull all measages from all subscriptions
Let's note that we pulled only one messages at the end of task 3.\
We can do it with multiple messages:
```Bash
gcloud pubsub subscriptions pull mySubscription --limit=3
```

> Note: once a message is pulled, you can't pull it again
>> they will loop though?

---
## Pub/Sub (Python vers.)
We can use Python right in the CLI.

### Task 1: Create a virtual env
Install the venv:
```Bash
sudo apt-get install -y virtualenv
```
build it:
```Bash
python3 -m venv venv
```
then activate so we can use:
```Bash
source venv/bin/activate
```
this will put us into it like so:
```Bash
(venv) student_02_a6089fafab15@cloudshell:~ (qwiklabs-gcp-03-d19a1e7184fb)$
```

### Task2: Install the client library
Install it with pip:
```Bash
pip install --upgrade google-cloud-pubsub
```
we are going to grab some specific sample code to run with git:
```Bash
git clone https://github.com/googleapis/python-pubsub.git
```
then we move to that dir:
```Bash
cd python-pubsub/samples/snippets
```

### Task 3: Pub/Sub basics
> skipping

### Task 4: Create a topic
We want to be able to do the same setup we've been doing but with Python.\
Let's first verify what our Project ID is.\
It is stored in a variable:
```Bash
echo $GOOGLE_CLOUD_PROJECT
```
for this specific sample code, let's read the main publisher file:
```Bash
cat publisher.py
python publisher.py -h # see usage and commands
```
> publisher.py is a script that demonstrates how to perform basic operations on topics with the Cloud Pub/Sub API


Looking at the comands we now know how to create a topic:
```Bash
python publisher.py $GOOGLE_CLOUD_PROJECT create MyTopic
```
and list them:
```Bash
python publisher.py $GOOGLE_CLOUD_PROJECT list
```

### Task 5: Create a subscription
```Bash
python subscriber.py $GOOGLE_CLOUD_PROJECT create MyTopic MySub
```
list them:
```Bash
python subscriber.py $GOOGLE_CLOUD_PROJECT list-in-project
```

### Task 6: Publish messages
```Bash
gcloud pubsub topics publish MyTopic --message "Hello"
```

### Task 7: View messages
```Bash
python subscriber.py $GOOGLE_CLOUD_PROJECT receive MySub
```

---
---
## Challenge Lab
### Task 1: Create a bucket
```Bash
gcloud storage buckets create gs://qwiklabs-gcp-03-7f2c57bab1f2-bucket --location=asia-southeast1
```

### Task 2: Create a Pub/Sub topic
```Bash
gcloud pubsub topics create  topic-memories-916
```

### Task 3: Create the thumbnail CLoud Run Function
This one will need the **BUCKET-NAME** used in task 1.

```Bash
# region
gcloud config set run/region asia-southeast1

# directory to host function
mkdir gcf_challenge && cd $_

# create index.js and package.json
touch index.js
touch package.json
```

use this code for index.js:
```javascript
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { PubSub } = require('@google-cloud/pubsub');
const sharp = require('sharp');

functions.cloudEvent('memories-thumbnail-creator', async cloudEvent => {
  const event = cloudEvent.data;

  console.log(`Event: ${JSON.stringify(event)}`);
  console.log(`Hello ${event.bucket}`);

  const fileName = event.name;
  const bucketName = event.bucket;
  const size = "64x64";
  const bucket = new Storage().bucket(bucketName);
  const topicName = "topic-memories-532";
  const pubsub = new PubSub();

  if (fileName.search("64x64_thumbnail") === -1) {
    // doesn't have a thumbnail, get the filename extension
    const filename_split = fileName.split('.');
    const filename_ext = filename_split[filename_split.length - 1].toLowerCase();
    const filename_without_ext = fileName.substring(0, fileName.length - filename_ext.length - 1); // fix sub string to remove the dot

    if (filename_ext === 'png' || filename_ext === 'jpg' || filename_ext === 'jpeg') {
      // only support png and jpg at this point
      console.log(`Processing Original: gs://${bucketName}/${fileName}`);
      const gcsObject = bucket.file(fileName);
      const newFilename = `${filename_without_ext}_64x64_thumbnail.${filename_ext}`;
      const gcsNewObject = bucket.file(newFilename);

      try {
        const [buffer] = await gcsObject.download();
        const resizedBuffer = await sharp(buffer)
          .resize(64, 64, {
            fit: 'inside',
            withoutEnlargement: true,
          })
          .toFormat(filename_ext)
          .toBuffer();

        await gcsNewObject.save(resizedBuffer, {
          metadata: {
            contentType: `image/${filename_ext}`,
          },
        });

        console.log(`Success: ${fileName} → ${newFilename}`);

        await pubsub
          .topic(topicName)
          .publishMessage({ data: Buffer.from(newFilename) });

        console.log(`Message published to ${topicName}`);
      } catch (err) {
        console.error(`Error: ${err}`);
      }
    } else {
      console.log(`gs://${bucketName}/${fileName} is not an image I can handle`);
    }
  } else {
    console.log(`gs://${bucketName}/${fileName} already has a thumbnail`);
  }
});
```
and this to package.json:
```JSON
{
 "name": "thumbnails",
 "version": "1.0.0",
 "description": "Create Thumbnail of uploaded image",
 "scripts": {
   "start": "node index.js"
 },
 "dependencies": {
   "@google-cloud/functions-framework": "^3.0.0",
   "@google-cloud/pubsub": "^2.0.0",
   "@google-cloud/storage": "^6.11.0",
   "sharp": "^0.32.1"
 },
 "devDependencies": {},
 "engines": {
   "node": ">=4.3.2"
 }
}

```
```Bash
npm install
```
> double check below before trying
```Bash
gcloud functions deploy nodejs-pubsub-function \
  --gen2 \
  --runtime=nodejs22 \
  --region=<REGION-NAME> \
  --source=. \
  --entry-point=helloPubSub \
  --trigger-topic <TOPIC-NAME> \
  --stage-bucket qwiklabs-gcp-04-cfd6cab10c3e-bucket \
  --service-account cloudfunctionsa@qwiklabs-gcp-04-cfd6cab10c3e.iam.gserviceaccount.com \
  --allow-unauthenticated
```
I ended up doing the following:
```Bash
student_02_84842e50102b@cloudshell:~/gcf_challenge (qwiklabs-gcp-02-46a1eaa4b990)$ gcloud functions deploy memories-thumbnail-creator \
> --gen2 \
> --runtime=nodejs22 \
> --source=. \
> --entry-point=memories-thumbnail-creator \
> --trigger-bucket qwiklabs-gcp-02-46a1eaa4b990-bucket \
> --region=asia-east1 \
> --allow-unauthenticated
```


> SOME CRAZY PERMISSION ISSUES
>> I had ot do it with GUI console instead



Finally test the function by adding the following to the bucket:
```Bash
# We will be adding this to the curl command OR by manually downloading and adding(?)
# https://storage.googleapis.com/cloud-training/gsp315/map.jpg
```


### Task 4: Remove the previous cloud engineer
> This one should just involve going to IAM and removing user 2's viewer access

































































