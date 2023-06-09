# Create a batch data pipeline using Azure Data Factory, Data Lake Storage and Synapse


## Requirements

1. Active Azure Subscription
2. Azure Data Factory 
3. Azure Synapse Analytics
4. Azure Storage Account

### 1. Create a Azure Storage account and container. 

You need to create a storage account and a container so that the databricks connection can write data to Azure Synapse. Use these guides to set up a [storage account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal) and [container](https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-portal).

### 2. Create a Synapse Analytics workspace and Dedicated SQL Pool. 

You also need a Synapse Analytics workspace and a dedicated SQL pool to serve as the data warehouse to store the data processed from the coincap.io API. Use these guides to set up your [workspace](https://learn.microsoft.com/en-us/azure/synapse-analytics/get-started-create-workspace) and [dedicated sql pool]([https://learn.microsoft.com/en-us/azure/storage/blobs/blob-containers-portal](https://learn.microsoft.com/en-us/azure/synapse-analytics/get-started-analyze-sql-pool). Remember to save your credentials somewhere, you will need them later to setup your connections in databricks.

### 3. Set up Azure Data Factory

Creating a Azure Data Factory workspace is easy. You can use this [documentation](https://learn.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory) from Microsoft to setup your factory. 

## Create the destination table (sink) in the Synapse Dedicated SQL Pool

Run the SQL code below on Azure Synapse Studio, this will create the destination table that will store the data collected and processed from the coincap API.

```	
CREATE TABLE [schema].[table name]
(
	[id_asset_price_history] int IDENTITY(1,1),
	[exchangeId] varchar(60),
	[baseId] varchar(60),
	[quoteId] varchar(60),
	[baseSymbol] varchar(60),
	[quoteSymbol] varchar(60),
	[volumeUsd24Hr] float,
	[priceUsd] float,
	[volumePercent] float,
  [runtime_timestamp] datetime
)
WITH
(
  DISTRIBUTION = ROUND_ROBIN,
	HEAP
);
```
## Get the connection strings for your services

### 1. Synapse Analytics Connection String: 
Go to your Synapse Analytics and follow this path to get your connection string `SQL Pools > (Select the SQL Pool you created) > Connection Strings > JDBC Tab > SQL Authentication`. It will look similar to the string below:

`jdbc:sqlserver://{workspacename}.sql.azuresynapse.net:1433;database={dbname};user={userid};password={your_password_here};encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.sql.azuresynapse.net;loginTimeout=30;`

### 2. Azure Data Lake Storage Connection String: 
Go to your Storage Account and select `Access Keys` to get your connection string. It will look similar to the string below:

`DefaultEndpointsProtocol=https;AccountName=<storage account>;AccountKey=<key>;EndpointSuffix=core.windows.net`

![Screenshot (21)](https://user-images.githubusercontent.com/50084105/228976604-c7ea7b79-854b-4b3e-8151-cd031e4b0f9a.png)

## Python script to send data to the Event Hub

You can use this code to upload data from the coincap API to Azure Data Lake. This script processes the json received from the API and appends the runtime timestamp after converting epochs to the ISO 8601 format. Note: A JSON array is uploaded to the container.

```
{
   "exchangeId":"Binance",
   "baseId":"bitcoin",
   "quoteId":"tether",
   "baseSymbol":"BTC",
   "quoteSymbol":"USDT",
   "volumeUsd24Hr":"2739949617.7015283309311878",
   "priceUsd":"28170.5081736755863902",
   "volumePercent":"32.5424100913471523",
   "timestamp":"2023-03-30T22:22:23.011000"
}
```

```
import requests
import json
import asyncio
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient
import datetime as DT
import os
def get_coin_prices(coin_list):
    payload = ""
    headers = {}
    coin_rates = []
    for coin in coin_list:
        url= "https://api.coincap.io/v2/assets/"+coin+"/markets"
        print(f"\rCoin Data Extracted: {coin_list.index(coin)+ 1}/{len(coin_list)}", end='', flush=True)
        response = requests.request("GET", url, headers=headers, data=payload)
        rates_response = json.loads(response.text)
        ts = rates_response["timestamp"]
        runtime_ts = DT.datetime.utcfromtimestamp(ts/1000).isoformat()
        for rate in rates_response["data"]:
            rate["timestamp"] = runtime_ts
            coin_rates.append(rate)  
    return coin_rates   
def get_asset_data():
    url = "https://api.coincap.io/v2/assets"
    payload={}
    headers = {}
    response = requests.request("GET", url, headers=headers, data=payload)
    asset_data = json.loads(response.text)
    coins = []
    for asset in asset_data["data"]:
        coins.append(asset["id"])
    return coins
coin_list = get_asset_data()
coin_rates_list = get_coin_prices(coin_list)
json_object = json.dumps(coin_rates_list, indent=4)
storage_account_name = "<name>"
storage_account_key = "<key>"
container_name = "<container name>"
directory_name = "<directory>"
connect_str = 'DefaultEndpointsProtocol=https;AccountName=<storage account>;AccountKey=<account key from earlier steps>EndpointSuffix=core.windows.net'
blob_service_client = BlobServiceClient.from_connection_string(connect_str)
container_client = blob_service_client.get_container_client(container_name)
local_path = "./asset-price-data"
file_name =  DT.datetime.now().strftime("%Y-%b-%d-%H-%M") + ".json"
upload_file_path = os.path.join(local_path, file_name)
blob_client = blob_service_client.get_blob_client(container=container_name, blob="asset-exchange-rates/"+file_name)
try:
    with open(upload_file_path, "rb") as data:
            blob_client.upload_blob(data)
            print(f"\nUploaded {file_name} as {str(DT.datetime.utcnow())}")
except:
    print("Issue with Upload")
```
This script should generate the output below. Run this script to check if the json files are getting uploaded to the container. Keep a sample file in the directory so that Data Factory can infer the schema later.

![Screenshot (29)](https://user-images.githubusercontent.com/50084105/229036412-35c2aa08-1ad6-42fa-bee9-a781d455eb58.png)

## Setting up Linked Services in Azure Data Factory

### Synapse Linked Service

Go to `Azure Data Factory Studio > Manage > Linked Services`. Click `New` and select `Azure Synapse Analytics`
+ Enter a Name.
+ Select `From Azure subscription` as the Account selection method. Select your subscription, server name, database name and SQL pool.
+ Select `SQL authentication` as the Authentication type.
+ Enter your username and password which was used to create your Synapse workspace.
+ Test the connection.
+ If the test connection works, click `Create`.

### Data Lake Linked Service

Go to `Azure Data Factory Studio > Manage > Linked Services`. Click `New` and select `Azure Data Lake Storage Gen2`
+ Enter a Name.
+ Select `From Azure subscription` as the Account selection method. Select your subscription and Storage account name.
+ Test the connection to `linked service`.
+ If the test connection works, click `Create`.

## Creating the Data Pipeline in Azure Data Factory

+ Go to `Azure Data Factory Studio > Author`. Create a pipeline `+ > Pipeline > Pipeline`.
![image](https://user-images.githubusercontent.com/50084105/229050487-23a1ee9f-642f-4de3-80ca-f30d89fd35a8.png)
 
+ Drag the `Copy data` Activity into the pipeline. Select this activity and give it a name under the `General Tab`.
![image](https://user-images.githubusercontent.com/50084105/229051343-88970f2c-a90a-461f-9d22-44140d5d50b0.png)

### Configure the Source

+ Go the `Source` tab and click `+ New`
![image](https://user-images.githubusercontent.com/50084105/229052735-46e0b71d-7387-424a-8143-01eaec007447.png)
+ Select `Azure Data Lake Storage Gen2` and then select the `JSON` format.
+ Select the linked service you had created earlier and specify the path. It will look like this. Over here `cryptodatalake` is my container and `asset-exchange-rates` is the directory where the data gets uploaded by the script.
![image](https://user-images.githubusercontent.com/50084105/229053575-2684c79c-069a-4f64-8f24-ea0e0633c5c0.png)
+ Select `From connection/store` to import the schema.
![image](https://user-images.githubusercontent.com/50084105/229054223-b2da7089-7bf2-4725-a40e-031a4b37f669.png)
+ Click `OK`
+ Configure these settings under source. Wildcard path ensures data is only ingested from the desired directory. For this batch pipeline, data is supposed to be uploaded to the container at an hourly frequency. This is why I configured the `Filter by last modified` settings between 5 minutes before the runtime to the runtime.
![image](https://user-images.githubusercontent.com/50084105/229054963-b61c68c1-84da-4e52-84dc-afa8a23820d2.png)

### Configure the Sink

+ Go the `Sink` tab and click `+ New`
+ Select `Azure Synapse Analytics` and select the linked service you created earlier.
+ Select the table name.
+ Import the schema from the connection/store. Click OK.
+ Select these options in the `Sink` tab.
![image](https://user-images.githubusercontent.com/50084105/229068101-80845f63-3888-4a6a-bcc0-c320d4eb30dd.png)

### Configure the Mapping

+ Go the `Mapping` tab and click `Import schemas`
+ Configure the mapping according to the screenshot below
![image](https://user-images.githubusercontent.com/50084105/229068875-20df2537-6448-411d-80cf-9c09862fe434.png)

### Configure the Settings

+ Go the `Settings` tab and configure these settings. Note: The copy activity will need a directory to store the data and do it's respective tasks. You will need to create another directory for the copy activity to stage the data. My directory is called `asset-exchange-rates-stg`.
+ Click `Validate all` and then `Publish all` once validation is complete.

![image](https://user-images.githubusercontent.com/50084105/229068961-835cf19a-4820-4031-a694-2cebd54c59e6.png)

### Create a trigger

+ Go to `Trigger > New/Edit`

![image](https://user-images.githubusercontent.com/50084105/229070480-cc7bd995-949b-4311-89d9-0d39bdd583aa.png)

+ Select `+ New` as `Choose trigger`
+ Configure these settings and click `Continue`.

![Screenshot (32)](https://user-images.githubusercontent.com/50084105/229073143-391adad4-aa32-45a6-8312-28fd59fd3253.png)

+ If you have json files in the container already you should get a data preview like this.

![image](https://user-images.githubusercontent.com/50084105/229073495-6a843d44-5914-4bbd-b101-1e3529529608.png)
+ Click `Continue` and then click `OK`.
+ Click `Publish All`


## Running the pipeline

1. Enable the trigger in Azure Data Factory. You can do this by going to 'Manage > Triggers'. If your trigger already has the status `Started`, you can go to step 2.
![Screenshot (28)](https://user-images.githubusercontent.com/50084105/229034613-019be370-769f-4c3a-8bf9-af0bba181932.png)

You should get this notification when publishing is complete.
![image](https://user-images.githubusercontent.com/50084105/229035291-e65befd0-fd0f-4007-88f8-54451e09a64d.png)

The trigger will look like this with the status `Started` once the changes have been published. The trigger will only work after publishing is completed, data uploaded to the Data Lake before publishing is complete will not be ingested.

![image](https://user-images.githubusercontent.com/50084105/229030556-fdf0907a-f86f-4984-90ee-c00801d08f36.png)

2. Run the script that uploads the coincap data in a json file to Data Lake Storage. The file was uploaded at 2023-03-31 05:59 UTC (This is 9:50 AM in my timezone).
![Screenshot (29)](https://user-images.githubusercontent.com/50084105/229036412-35c2aa08-1ad6-42fa-bee9-a781d455eb58.png)

You should be able to see the file in your container
![image](https://user-images.githubusercontent.com/50084105/229037765-6e8d4495-7c7d-4c84-a3a3-190d9e5b3ec9.png)

Go to `Monitor > Trigger Runs`. You should see a pipeline run.
![image](https://user-images.githubusercontent.com/50084105/229036221-3efb4dcd-55f0-40c6-acd7-b9fe1010e424.png)

3. After a few minutes the data will be inserted in the Synapse destination table. You can verify this with the runtime_timestamp field for the latest entries. Note: Runtime was inserted with the UTC timezone.
![image](https://user-images.githubusercontent.com/50084105/229036886-bea3a2f5-3410-4e7f-a26e-bccb9a1e5dea.png)
