# 1. Get to know our App

### What is our App?

We want to dig deeper and know more about the articles that are being linked inside tweets.  For this, we will leverage the Watson Alchemy API service which can extract
keywords, entities, concepts, and document-level sentiment from web links.  So our flow will extract the links from the tweets and feed them to the Watson service which 
will then write the analysis to a Cloudant database.  

# 2. Getting started

If you have already completed version 1.0 of the app, you can skip this part and go directly to section 3, else please follow these directions to create your boilerplate and Node Red flow.

####Set up the Node-RED boilerplate
* Login to your Bluemix account.  If you do not have one already, go to http://ace.ng.bluemix.net, and sign up for a new account.  
* After you login with your Bluemix account, you will see a list of *Applications* and *Services* in Bluemix. We are going to create a Node-RED boilerplate application and *bind* the Alchemy API service to the application.  A Boilerplate can be imagined as a bundled application with relevant bound services for a particular targeted usage to get up and running quickly. 
* In the upper menu bar, go to **CATALOG**  
* This page lists all kinds of different boilerplates, runtimes and services that are offered by Bluemix. Browsing through this page can give you an idea about all the exciting features offered by Bluemix.*   
* From the Boilerplates section, select the **Node-RED starter** boilerplate.  
*Here you can see the SDK for Node.JS and the two services - Cloudant and Monitoring & Analytics that come as a part of the Node-RED boilerplate. We will be using the Cloudant service and also binding the Watson Alchemy API service as well.*  
* Provide a **unique name** to the app on the right side of the page and click **CREATE**. A non-unique name would notify and urge you to think of a more creative name.  
* In a couple of minutes, your application will be staging and ready to run. 
* You will land up on the 'Start coding with Node-RED' page. The instructions on this page are to get familiar with Cloud Foundry command line interface to control your application. You can go through the steps and try them out at leisure. We won't be needing that for our app right now. 
* On the left panel, click on **Overview**. This is the overview of your Application showing the link to go to the App, status of the app, instances, memory and the services bound to it.  
* We want to bind the Watson "Alchemy API" service to the App. Lets click on the **ADD A SERVICE OR API** button.
* Search for the "Alchemy API" service in the search tab or select the **Alchemy API** service in the Watson section.
* In the right hand side panel, the app name would be pre-filled and the service name would be randomly generated. The selected plan should be defaulted to Free plan. Click **CREATE**.   
* The app will now **RESTAGE**, which should take a minute. When done, your Alchemy API service is now bound with the Node-RED App and should show up on the App Overview page.  

####We are done with the setup of the Node-RED app and the Watson Alchemy API service. 

# 3. Let's create our Node-RED flow!

#### The Node-RED flow

* Go to your application's URL (from the App overview page, you can click on the link next to **Routes** right beneath the App name).
* This will land you on the Node-RED in BlueMix page. Click on the red button **Go to your Node-RED flow editor**
* The left panel of this page includes all the different nodes available for direct use to plug and play within your flow structures. The main area called the 'sheet' is where you pull in the nodes and connect them to each other to make a functional app. On the right panel, 'info' displays the information about a particular node when selected and the 'debug' tab is the space where we validate and visualize the state of data flowing at any time within the flow.
* On the top right corner, click the icon:
![alt text](https://raw.githubusercontent.com/CDSLab/Insight2015/master/CandidatesApp1.0/images/%245A2C55CE3129CFC8.bmp)
 and select **Import** > **Clipboard**  
* In the textbox, paste the content of the file **CandidatesApp2.0_Import_Sheet1.txt** in the folder **files** of this project. Click **Ok**  
![alt text](https://raw.githubusercontent.com/CDSLab/Insight2015/master/CandidatesApp2.0/images/node_flow.jpg)

Now let's do some configuration of our flow:

* Set the Twitter credentials  
Double click on the blue Twitter node on the left of your flow and setup the "Login as" using the icon next to it. You will need to have a Twitter account to authenticate to the Twitter service.
The twitter node is set by default to monitor "Jeb Bush", but you can change this to another candidate or to be anything you want.
 
![alt text](https://raw.githubusercontent.com/CDSLab/IDUG2015/master/CandidatesApp/images/edit_twitter_in_node.bmp) 
* Point the cloudant node to your service.  Open it up and in the dropdown select the service that was created and bound to your Node Red application as part of the boilerplate
creation process.
* Now we are ready to deploy our flow.  Click **Deploy**.  
* If you activate the debug node coming out of "Feature extract", then in the **debug** panel on the right, you should start seeing the live tweets that are being saved into your 
Cloudant database. 

*Let's take a closer look at what we just did.*  We imported a flow that represents a flow of data from left to right.  On the left we have a Twitter node that is triggered whenever
a new tweet about our topic comes in.  Then a function node has some javascript that checks if a URL is included in that tweet.  Next, the Watson alchemy API node extracts key 
information about the article linked from the URL - double click on it and you can see all the things we are extracting - title, entities, keywords, concepts, publication date and
document sentiment.  All of these things are passed along in the JSON payload into the Cloudant node, where they are stored into a Cloudant database called "features".


# 4. Define a map-reduce view in our Cloudant database 

* Open up your app in Bluemix, and click on the Cloudant service.  There should be a green launch button in the upper right.  Click it to launch the Cloudant management console.
* If tweets have gone into Cloudant already, then you should see a database already created called "features", this is where the document features from the articles are being stored.  Click on it.
* Click on the plus sign next to "All Design Docs", and select "New View".
* Under "Design Document", select "New Design Document", and give "app" as the design doc name.
* For "Index name", name is "concepts"
* Under "map function", paste in the following code:
``` 
    function (doc) 
    {  
  
       for(var concept in doc.features.concept) {

          if (doc.features.concept[concept].relevance > 0.9) {
        
             emit(doc.features.concept[concept].text, 1);
    
          }
  
       }

    }
```


* Under "Reduce (optional)", choose "_count".

*Let's take a closer look at what we just did.*  We defined a map-reduce view.  It first searches for all the concepts with a relevance score greater than 0.9 and assigns a 1 to them,
then the reduce counts up all of the like concepts and returns the value of the count for each concept.

# 5. Define a REST API in Node-Red 

* In the screen where you defined your view, in the upper-right hand corner, click "Query Options" and select the "Reduce" checkbox and click the blue "Query" button.
* Now click "API URL", and "Copy" the REST API URL to your clipboard.
* Now, adjust permissions in Cloudant so that our view can be called with a REST API.  To do this, go to the top-level page for your "features" database, and click on 
"Permissions", and select the checkbox next to "Reader" for "Everybody Else".
* Now, go back to your Node Red flow.  Open up the node labeled "http request", and paste in the URL that you just copied to your clipboard.  Now click on the red Deploy button but this
time choose the option "Deploy->modified nodes".
* Now, if you go to http://your_app_URL/concepts, you should see a count of the most common concepts discussed in the articles linked from the tweets you have collected!

*Let's take a closer look at what we just did.*  We defined a REST API in Node Red to return the top counts of the most common concepts extracted from the articles tweeted about our
topic.  But we already had a REST API from Cloudant, so why did we create another?  Two reasons, one is that the REST API in Cloudant doesn't sort the counts, so if you open up the
function node in our flow after the http request, you can see some Javascript code that does this.  The other reason is that we can avoid any same-origin policy problems if we wish to
also use our NodeJS/Node Red application to render a UI on top of this REST API.




 






