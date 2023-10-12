# Live stream analysis using Video Analyzer for Media (Video Indexer)

Use Video Analyzer for Media to process a live stream and display the data in a test player !

![Test Player](images/live-media-analytics-player1.png?raw=true)

This solution uses Azure functions and two Logic apps to process a live program (from a live channel in Azure Media Services) with Video Analyzer for Media, and display the result with Azure Media Player playing the live stream.

The Logic apps workflow does the following :

**Step 1 Logic app**

* it runs every 60 seconds
* subclips the last minute
* sends this subclip asset to Video Analyzer for Media, which runs in a Media Services Account (recommended)

![Screen capture](images/logicapp5-live1.png?raw=true)

**Step 2 Logic app**

* called by Video Analyzer for Media when indexing is complete (using a callback url)
* gets the insights, update the timestamps
* sends this data to a Cosmos database
* deletes the Video Analyzer for Media video and the subclip asset

![Screen capture](images/logicapp5-live2.png?raw=true)

## Step by step configuration

### 1. Create a Video Analyzer for Media and AMS accounts

Use the "Connect" button to Azure in Video Analyzer for Media portal to create an account ([more info](https://docs.microsoft.com/en-us/azure/azure-video-analyzer/video-analyzer-for-media-docs/connect-to-azure#connect-to-azure-manually-advanced-option)).

You can install Video Analyzer for Media into a new or existing AMS account.

Go to the [Video Analyzer for Media Developer Portal](https://api-portal.videoindexer.ai/products/authorization), sign in, and retrieve the Subscription Primary Key.

### 2. Create a Service Principal

In the Azure portal or AZ CLI, create a Service Principal attached to the AMS account previously created. Save the password too. It will be needed in step #3. To do it within the portal, go to the API tab in the account ([follow this article](https://docs.microsoft.com/en-us/azure/media-services/media-services-portal-get-started-with-aad#service-principal-authentication)).

Please note that it may take some time for the Service Principal to be active.

### 3. Deploy the Azure functions

If not already done : fork the repo, deploy Azure Functions and select the **"media-functions-for-logic-app"** Project (IMPORTANT!)

Follow the guidelines in the [git tutorial](1-CONTRIBUTION-GUIDE/git-tutorial.md) for details on how to fork the project and use Git properly with this project.

Note : if you never provided your GitHub account in the Azure portal before, the continuous integration will probably fail and you won't see the functions. In that case, you need to setup it manually. Go to your azure functions deployment / Functions app settings / Configure continuous integration. Select GitHub as a source and configure it to use your fork.

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fmedia-services-dotnet-functions-integration%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

### 4. Create a Cosmos database and container

By default, the template is configured to use a database id named "vidb" and a container id (collection) named "vicol". So please create such database id and container id. Use "/date" as the partition key for the container id.

Create a settings 'CosmosDBConnectionString' in the Azure functions app settings and store in it the Cosmos DB Connection string. It is used by the function to retrieve the insights and pass them to the player.

### 5. Configure live streaming with AMS

This sample use AMS v2 but it would be better to use AMS v3. You can use the REST API, SDKs, Azure portal or [AMSE for v2](http://aka.ms/amseforv2).

Make sure that the AMS streaming endpoint is started.
To do so, go to the Azure portal or AMSE, select the Azure Media Services account which has been created in step #1, and start the default streaming endpoint.

![Screen capture](images/start-se-1.png?raw=true)

![Screen capture](images/start-se-2.png?raw=true)

Note : you cannot create a AMS v2 live channel with the Azure portal anymore (feature is deprecated). Please use the REST API, SDK or AMSE for v2.

Create a channnel "Channel1" and program "Program1" (with an ArchiveWindow of minimum 20 min) in the Media Services account used by the functions. Start them. Connect a live encoder (for example, [Wirecast](https://www.telestream.net/wirecast/)) and push the live RTMP stream to the channel. If you want to use another name for the channel and program, then you will have to edit the step 1 logic app to reflect the new names.

Important : setup 10 S3 media reserved units in the Media Services account if you process a SD stream. A higher resolution stream may require more S3 units. You can create a new support request in the Azure portal to require more S3 units, as the default limit is 10.

### 6. Deploy the logic apps

Deploy the two logic apps using this template:

Click the button to deploy the template in your subscription:
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fmedia-services-dotnet-functions-integration%2Fmaster%2Fmedia-functions-for-logic-app%2Flogicapp5-livevideoindexer-deploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

Once deployed, fix the errors in both logic apps (go to designer):

* Video Indexer components (select the location and subscription in all Video Indexer connectors)
* Check the Cosmos DB components and connection

### 7. Setup the test player

A sample html player is [provided here](liveanalysisplayer).
You need to download the two files, edit the html file and publish them on a web server.
Editing must be done to change the following links:

* update the media player source url to use your custom live program URL (search for '<source src='),
* specify the URL of the Azure function **query-cosmosdb-insights** ('var functionquerycosmosdbinsights =')
* specify the location of the Video Indexer ('var videoindexerregion = ')

Make sure that you add * to the CORS configuration of the Azure function deployment.

## Notes

* You can to customize the channel name, program name and language of the audio. To do so, change the parameters in the live-subclip-analytics function call and  video indexer upload component from the step1 logic app.
* to increase the performance, it is recommended to limit the resolution of the live stream. This will speed up the processing of Video Indexer. For example, start testing by sending a SD resolution stream (example: 854x480)
* monitor the job queue(s) and allocate the right number of S3 media reserved units  

![Screen capture](images/logicapp5-live-param1.png?raw=true)

![Screen capture](images/logicapp5-live-param2.png?raw=true)

## Functions documentation

This [page](Functions-documentation.md) lists the functions available and describes the input and output parameters.
