# Create a streaming data pipeline using Azure Databricks, Event Hub and Synapse


## Requirements

1. Active Azure Subscription
2. Azure Event Hub 
3. Azure Synapse Analytics
4. Azure Stream Analytics Job

### 1. Create a Event Hub Namespace and Entity

For this project we will be extracting data from the `coincap.io API`. You need to create a unique eventhub namespace, for this project I have created one named `cryptodatastream`. After creating a namespace you can now create an event hub entity, I have created one named `coincaphub`. Setting up these resources is easy you can use this [documentation](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-create) to help you get started.

### 2. Create a Synapse Analytics workspace and Dedicated SQL Pool. 

You also need a Synapse Analytics workspace and a dedicated SQL pool to serve as the data warehouse to store the data processed from the coincap.io API. Use these guides to set up your [workspace](https://learn.microsoft.com/en-us/azure/synapse-analytics/get-started-create-workspace) and [dedicated sql pool]([https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-portal](https://learn.microsoft.com/en-us/azure/synapse-analytics/get-started-analyze-sql-pool). Remember to save your credentials somewhere, you will need them later to setup your connections in databricks.

### 3. Set up a Azure Stream Analytics Job

You can use the official [documentation](https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-quick-create-portal#create-a-stream-analytics-job) from Microsoft to create a Stream Analytics Job. You do not need to set up a job input or output, this will be covered later.


## Create the destination table (sink) in the Synapse Dedicated SQL Pool

Run the SQL code below on Azure Synapse Studio, this will create the destination table that will store the data collected and processed from the coincap API.

```	
CREATE TABLE [schema].[table name]
(
    [id_asset_statistics_history] int IDENTITY(1,1), --Automatically increases the value for this field for every row insert
		[id] varchar(60),
		[asset_rank] bigint,
		[symbol] varchar(60),
		[asset_name] varchar(60),
		[supply] float,
		[maxSupply] float,
		[marketCapUsd] float,
		[volumeUsd24Hr] float,
		[priceUsd] float,
		[changePercent24Hr] float,
		[vwap24Hr] float,
		[explorer] varchar(600),
		[runtime_timestamp] datetime
)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    HEAP
);
```
## Get the connection strings for your services

### 1. Event Hub Connection String: 
To get the connection string for your event hub entity follow this path `Event Hubs > Event Hub Namespace (What you just created) > Event Hubs (Under Entities) > Event Hub (Entity you created) > Shared access policies`. Click Add, create a name and select Listen. You will need to create another policy for sending data to the event hub, you can use the screenshot below as reference to find your connection string for the respective policy.
We will need the send policy for the python script that will send data to the event hub. 

![Get connection string from event hub](https://user-images.githubusercontent.com/50084105/228879400-dfe8a725-3f93-484c-8bba-2383ac2fea31.png)

These strings will look like this: `"Endpoint=sb://<namespace>.servicebus.windows.net/;SharedAccessKeyName=<policy name>;SharedAccessKey=<key>;EntityPath=<entity>"`

## Python script to send data to the Event Hub

You can use this code to stream data from the coincap API to the Event Hub. This script processes the json received from the API and appends the runtime timestamp in epochs to the final json that is sent to the Event Hub. Note: A JSON array is sent to the Event Hub.

```
{
   "id":"bitcoin",
   "rank":"1",
   "symbol":"BTC",
   "name":"Bitcoin",
   "supply":"19332443.0000000000000000",
   "maxSupply":"21000000.0000000000000000",
   "marketCapUsd":"547205357840.2894946660999555",
   "volumeUsd24Hr":"7843196880.5992743425954912",
   "priceUsd":"28305.0289009148763385",
   "changePercent24Hr":"-0.5321226064725184",
   "vwap24Hr":"28485.3107379122638250",
   "explorer":"https://blockchain.info/",
   "timestamp":1680191532104
}
```

```
import requests
import json
import asyncio
from azure.eventhub import EventData
from azure.eventhub.aio import EventHubProducerClient
import datetime

EVENT_HUB_CONNECTION_STR = "<Event Hub Connection String from Earlier>"
EVENT_HUB_NAME = "<Event Hub Entity you created>"

def get_asset_data():
    url = "https://api.coincap.io/v2/assets"
    payload={}
    headers = {}
    response = requests.request("GET", url, headers=headers, data=payload)
    asset_data = json.loads((response.text))
    assets = []
    ts = asset_data["timestamp"]
    for asset in asset_data["data"]:    
        asset["timestamp"] = ts
        assets.append(json.dumps(asset))
    return assets

async def run():
    # Create a producer client to send messages to the event hub.
    # Specify a connection string to your event hubs namespace and
    producer = EventHubProducerClient.from_connection_string(
        conn_str=EVENT_HUB_CONNECTION_STR, eventhub_name=EVENT_HUB_NAME
    )
    async with producer:
        # Create a batch.
        event_data_batch = await producer.create_batch()
        stream_data = get_asset_data()
        # Add events to the batch.
        for i in stream_data:
            event_data_batch.add(EventData(i))
            print(f"\rCoins sent to EventHub: {stream_data.index(i)+ 1}", end='', flush=True)
        # Send the batch of events to the event hub.
        await producer.send_batch(event_data_batch)
        print("\nData Published at: "+ str(datetime.datetime.utcnow()))

asyncio.run(run())    
```
This script should generate the output below

![Screenshot (15)](https://user-images.githubusercontent.com/50084105/228889499-a05b0edd-4297-4dc2-80e4-01bbaec04f95.png)


## Running the scripts

1. Run the code in the Stream Analytics Job from the overview page of the Job. You should see this screen


2. Run the script that streams the coincap data to the eventhub. The data for this run was created at 2023-03-30 12:28 UTC.


3. After a few minutes the data will be inserted in the Synapse destination table. You can verify this with the runtime column for the latest entries.
