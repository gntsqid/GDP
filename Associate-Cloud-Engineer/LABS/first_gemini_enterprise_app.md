# Create your first Gemini Enterprise Application
This one gives us a **required** skill badge that we need to submit.\
I will not be doing the rest of the course as I just need the badge for this one.

## Create Gemini Enterprise App
### Create
This will be a hands-on simulation.\
We will create a Gemini enterprise App and connect it to diverse data stores such as Drive, Calendar, and Cloud Storage.\
We will also be using the general AI asssitant and the pre-built agents as well as NotebookLM.

#### Gemini Enterprise
Find this by searching in the cloud console.\
Select "Create your first app".

Give it a name.\
For the demo it will be "Cymbal Foods - Marketing Team*.\
Then choose a location.\
In this case, we will be "global".

#### Identity Provider
After creting, let's set up an identity provider.\
This can be found in the admin dashboard of our created app.\
It is also called *workforce identity*.

We can choose various third party vendors but in this case we want to use ***Google Identity***.

#### Data Sources
Once identity provider is set, we can go to *Nav -> Connected data stores*.\
Then we can *+ New data store*.\
The first one we will test with is *Google Drive*.

We will see various workspace sources and can search for them.\
In this case, Drive is the first in the list.\
Hit *select*.

After this, we want to select with drives to sync.\
For this demo with will do *ALL*.

Name the connector now.\
FOr the demo, we will keep it as "Google Drive".

Once done with this, let's now go ahead and add a Calendar and Cloud Storage.\
For Calendar, it's exactly the same as Drive.\
For Storage though, we want to choose *Documents* for our unstructured data import.\
This will allow PDFs, HTML, TXT, and other document file types.

We will also select *one time* for our synchronization frequency.\
This is how many times the data will get ingested.

> For folder, we want to select a specific cloud storage bucket to get our files from.

we are ready to move on.

#### Enterprise App
Back in our Overview dashboard, we will see that our webapp is ready!\
We will see a URL we can go to in order to acces it.\
For this demo, it is only available in the simulation, but it takes us to a cool Gemini Web UI page!\

Let's prompt it to test.\
Start with a simple one for the demo like "Give me an update on our latest marketing data.".


##### Enhancing Conversation
We can click the little slider dial button next to the plus sign in order to enhance our conversation.\
This lets us change some search params.\
To start, let's select *internal sources*.\
Now it has access to files we've provided via the storage!

##### Idea Generation
This tab lets us use a multi-agent ideation session.\
We can test it with this demo prompt: "Help me brainstorm a new marketing campaign for Gen-Z customers based on our latest products".

We will now see a generated plan for us to review.\
Once we are happy with it, we can *start session*.

After some time, multiple agents will get to work putting our idea together.

##### Deep Research
This tab is good for in-depth analysis and comprehensive reporting on complex topics.\
We can test with "Research latest marketing tips and tricks that food companies....and generate a list of actionable steps.....improve our current marketing strategy."

Just like with the idea generation, we will get a plan we can review and execute with *start research*.
> Once done, it will give us a comprehensive report with an audio summary.

##### NoteBookLM
We can now transform our source content into a variety of dynamic formats.\
Let's create a new notebook and add data sources such as Google Drive docs, website URLs, Youtube videos, and more.

We can also save repsonses as notes if we need to access them later.

###### Studio
This is a tab within the NoteBookLM that lets us transform material into dytnamic formats such as audio, video, mindmaps, or reports.

--- 
### QUIZ
This is after the simulation.

1. A researcher has a small set of specific text files and wants to create a video overview that summarizes only those sources. Which tool is best suited for this task? 
**NOTEBOOK LM**

2. You ask NotebookLM a question about a product that is not mentioned in any of the sources you uploaded to the current notebook. How will the tool respond? 
**It will generate an answer based only on content of uploaded docs**

3. A retail company has detailed product descriptions in PDF files and also has a corresponding set of structured data (product ID, price, stock level) for each PDF. To ensure that users can filter search results by price or stock level, which data import option should they choose?
**Documents with metadata (ARG)**

4. A marketing team lead needs to quickly understand the primary findings of a customer sentiment pie chart saved as a PNG file. How can they use Gemini Enterprise to achieve this?
**Ask AI to summarize the custoemr sentiment pie chart**

5. Cymbal Foods needs a comprehensive report on global food marketing trends by analyzing hundreds of external websites. Which feature should they use?
**Deep Research Agent**


6. An administrator is concerned that connecting a shared Google Drive to the Gemini Enterprise application will allow everyone in the company to read private, unshared documents. How does Gemini Enterprise handle file permissions?
**Users can only access documents already shared with them via existing permissions**

7. A company stores its weekly sales reports in a designated Cloud Storage bucket. To ensure their Gemini Enterprise application always reflects the latest data, which synchronization frequency should they use?
**periodic**

8. A company’s Google Cloud administrator is performing the initial one-time configuration for a Gemini Enterprise application and needs to ensure the organization’s workforce can securely authenticate and access their internal data. Which task should they perform first?
**identitiy provider**















