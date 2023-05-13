---
title: "GCP Batch Ingestion 1 - Local Development"
date: 2023-05-13T10:03:58+07:00
description: An introduction to locally developing code for a batch ingestion process
weight: 10
menu:
  sidebar:
    name: "1. Local Development"
    identifier: gcp-batch-ingestion-1
    parent: batchingest
    weight: 10
hero: images/cover.jpg
tags: ["GCP","Python"]
categories: ["Easy"]
summary: >
  An introduction to locally developing code for a batch ingestion process using python and the GCP API. Introducing the motivation behind this series of articles and the code concepts.
---
## Tackling Automatic Batch Data Ingestion with GCP

- Code: [github.com/matthh9797/gcp-ingest-template](https://github.com/matthh9797/gcp-ingest-template)

I remember right at the start of my career, when I bought the book [Data Science on the Google Cloud Platform](https://www.google.co.id/books/edition/Data_Science_on_the_Google_Cloud_Platfor/F7tmEAAAQBAJ?hl=en&gbpv=1&printsec=frontcover) and opened the first chapter, titled "Ingesting Data into the Cloud", I thought to myself *"This part will be easy, I already know how to do this"*. Right? Wrong. What I did know then is how to develop single use case solutions locally and as it turned out, generalising, productionising and automating the process had lots of new concepts and technologies for me to try get my head around.[^1]

So, it's two and a half years later and on and off, I've kept on coming back to that first chapter with a different question. Why do I need a storage bucket? What is a service account? Why do I need Flask? Why the f#@! is a docker container? ... and slowly I started to get my head around some of these concepts. So, I wanted to present this series of articles and supporting repository, as a note to anyone else who is in the same position as I was two and a half years ago.

## Introducing the Repository and the Data

In this article I will introduce a process for automating batch data ingestion into GCP. You can use the code in the repository mentioned above as a starting point for any project where data ingestion is required as a first step by clicking on the "Use this template" button on GitHub. I will step through the code, giving a sort of dummies guide to what it is doing. Meaning, I will try to keep the explanations as simple as possible and avoid direct quotes from documentation and industry jargon.

For the first couple of articles in this series I am going to use the fantasy.premierleague API as an example.[^2] I have chosen this API because it provides several tables which are a good structure to introduce batch ingestion concepts, since they are small enough that you can just overwrite the BigQuery table every day (oh yeah, and because I love football). In this article I will look at setting up your code locally. In the second article, I will discuss productionising and automating your code. In the third article, I will look at how you can tackle data ingestion when your source data is too big or it does not make sense to load the whole source data every day, using financial data from Yahoo Finance.

## Setting up a Base Class

> The full code for this section can be found here: [github.com/matthh9797/gcp-ingest-template/ingest/ingest/base.py](https://github.com/matthh9797/gcp-ingest-template/blob/main/ingest/ingest/base.py). I will not go through the code line for line but only highlight the important parts.

So let's get on with the code. I have chosen to write the code as a generic process which can be extended to specific use cases. The reason for writing code this way is that when it comes to repeating the process, instead of wasting time and effort trying to reproduce what you did last time, you can focus on continuous improvement of the generic process. [^3]

The code for a data ingestion process can be wrapped up into 3 functions:

- `extract()` - Download the data from source and re-structure it into a dictionary of dataframes, the keys being the table name and the values being the data.
- `transform()` - An optional, in the middle step will be defining your data transformations as pandas logic (e.g. add a download_at field or unique_id)
- `load()` - Load your data into BigQuery.

I will define a base class called `Ingest` to wrap up these functions with a `run()` method which will run each function and pass it to the next one. The class is initialised with a config dictionary which contains all of the process parameters. The structure of the class looks like this:

```python
# /ingest/ingest/base.py
class Ingest:
    def __init__(self, config: dict) -> None:
        self.config = config
   ...
    def extract(self) -> dict:
      ...
      return df_dict_raw   

    def transform(self, df_dict_raw: dict) -> dict:
      ...
      return df_dict_transformed

    def load(self, df_dict_transformed: dict, gcp_connector: GcpConnector) -> None:
      ...
    
    def run(self, env: str, overrides: dict = None) -> None:
      ...
```

In this case, I think of the base class as a blueprint for the process with a bunch of default methods. The default methods can be overwritten for specific instances of the class, depending on the needs and structure of your data source. I will show an example of this in Article 3 when we overwrite the default extract and transform methods with an instance of the base class for Yahoo Finance.

I always try to create a central `config.yaml` file with all of the parameters for my code which will be read into python and used to initialise the class. This means you don't need to hard code any values which makes it easier to write generalised and reproducible code.

### Extract

The first step to any ingestion process is to download the data from source. If your source is an API, that will mean sending a request to a URL (API Endpoint) to retrieve the data. In general, there are two ways to do this using python:

1. Using a package like `requests` to interact directly with the endpoint (Articles 1 and 2).
2. Using a python library which has been created as an interface to interact with the API (Article 3 using `yfinance`).

In either case the purpose of the extract method will be to download the data tables from source and restructure them into a dictionary of dataframes which can be passed to our transform function.

We can do this by setting up two sub-methods:

- `download()` -  Downloads the data from the API endpoint.
- `to_df_dict()` - Parses the json output of the download function to a dictionary of python dataframes.

Now our `extract()` method will combine these two sub-methods. In the case of the base class the default method will be to retrieve data using the `requests` library, since this is the most generic way to interact with API's using python.

### Transform

You might have heard of ELT and ETL. In most cases, I prefer ELT (i.e. load the raw data into BigQuery and do the data transformations in there). However, sometimes there is a need to add some fields before the table is loaded into BigQuery. For example, you may want to add a `downloaded_at` field which could be used to debug your process if something goes wrong, particularly when you have a process where rows are appended onto your table. You may also want to add a `unique_id` to make sure your process is not creating duplicate rows.

In this case, we would like to create some functions which clean up our tables. Personally, I like to create a transform function for each table I'd like to transform and name it `transform_{NAME_OF_TABLE}`, then we can loop over our dictionary of dataframes applying each function to each table using the below python syntax.

```python
transform_dict = {
  'table_1': transform_table_1,
  'table_2': transform_table_2
}

for k, f in transform_dict.items():
    df_dict_transformed[k] = f(df_dict_raw[k])
```

Of course, we cannot assume the number of tables or the name of the tables in a generic process. So, the default method for transform will be to do nothing (i.e. return the input from the `extract()` method).

### Load

The load function is how we will load our data to BigQuery. There are quite a few different ways we might want to do this, I will go into more detail about these in Article 3 of this series. However, for now we want to create a generic function that will take the dictionary of dataframes and upload them to BigQuery tables using some parameters passed to an upload function.

```python
for dataframe_name, dataframe in df_dict_transformed.items():
    gcp_connector.upload(dataframe = dataframe, table_id = dataframe_name, **upload_kwargs)
```

In the above code, the `df_dict_transformed` dictionary of dataframes is the output of our `transform()` method. The `gcp_connnector` is an instance of the `GcpConnector` class which we will define to make it easier to interact with GCP. The upload_kwargs will be passed into our class using a config file, more on that below.

`GcpConnector` is a class I have defined to make connecting and uploading tables to GCP easier. The class is initialised with a config file with parameters for connecting to GCP, similar to the `Ingest` class. When the class is initialised it will create BigQuery and storage clients using the environment passed in by your config file.

```python
# /ingest/gcp/gcp.py
class GcpConnector:
    def __init__(self, auth_config = None) -> None:
      ...
      self.bq_client = bigquery.Client()      
      self.storage_client = storage.Client()    

    def upload(self, ... )
      ....  
```

The class contains an upload function which wraps around several options for uploading data into GCP. I will go into more details on this function and the several upload options you can consider in Article 3 but for now I will describe two options for your upload function. The two options I would like to consider are uploading directly from a dataframe or uploading via cloud storage.

Uploading directly from a dataframe saves a lot of work and I have often found that this method works really well in a variety of cases and helps to avoid the numerous difficulties which can arise when saving your data to a file. However, in a production environment I think it is always better to go with saving a json or csv file to cloud storage, then uploading this file to BigQuery. The main reason for this is debugging the process when something goes wrong. It is convenient to be able to download the data that was uploaded on a certain date.

A simplified version of the upload function is below. We have created a file name which is the upload date concatenated with the table id (e.g. 20230201:elements). Then, we create a function which uploads the dataframe to this file in Cloud Storage and returns the location of the file. Finally, we can pass this location into another function which uploads the file to BigQuery.

```python
class GcpConnector
  ...
  def upload(self, dataframe, table_id, dataset_id, bucketname, ...):
    ...
    uploaded_at = datetime.now()
    blobname = f'{uploaded_at.strftime("%Y%m%d")}:{table_id}'
    # this function uploads your dataframe to cloud storage
    gcslocation = upload_dataframe_to_bucket(storage_client, dataframe, bucketname, blobname, file_type)
    ...
    # this function uploads the cloud storage file to bigquery
    upload_bucket_to_table(bq_client, gcslocation, table_ref, job_config)
```

### Run

Now we have created our main methods, we can create a `run()` method to run the process. The simplified version of the full code looks like this:

```python
def run(self, env: str, overrides: dict = None) -> None:
  df_dict_raw = self.extract()
  df_dict_transformed = self.transform(df_dict_raw)
  self.load(df_dict_transformed, gcp_connector)
```

The `env` variable controls whether your environment is dev or prod which will dictate how the program connects to GCP. The `overrides` variable allows you to pass in a dictionary which overrides the config that the class has been initialised with. This is useful when you need to do one-off manual runs with different configurations.

## Creating an FPL Instance

> The full code for this section can be found here: [github.com/matthh9797/gcp-ingest-template/tree/fantasy_premierleague/ingest](https://github.com/matthh9797/gcp-ingest-template/tree/fantasy_premierleague/ingest). I will not go through the code line for line but only highlight the important parts.

Now the base class is setup, let's create an instance of the class for ingesting fantasy.premierleague API data into GCP and setup a configuration file for the process. Since the fantasy.premierleague API has endpoints we can interact with it directly using `requests` and we do not have to overwrite any of the default methods of our base class. So, we can just set up our `FplIngest` class as a child of the `Ingest` class, inheriting all of the default methods.

```python
# /ingest/ingest/__init__.py
from .base import Ingest


class IngestFpl(Ingest):
    pass
```

### Configuration

I have learned two great frameworks so far in my career, DBT for analytics engineering and Hugo for static websites. Central to the reason these frameworks are great is generic code and well-structured config files. These two frameworks have taught me to always use config files to set up my code.

So let's set up the configuration for our class. The config file will have two main sections for configuration. The API config which will control the endpoint and tables to download from your API and the GCP config which will control how you connect and upload data to GCP.

```yaml
# /ingest/config.yaml
env: dev

api:
  baseurl: https://fantasy.premierleague.com/api
  endpoints:
    - name: bootstrap-static
      tables: 
        - elements
    - name: fixtures
  
gcp:
  key_file: <NAME_OF_KEYFILE> # Must be place in your downloads folder
  upload:
    dataset_id: fpl_raw
    bucketname: fpl-staging
```

In the above code we specify that we would like to download the elements table from the `https://fantasy.premierleague.com/api/bootstrap-static` endpoint and the fixtures table from the `https://fantasy.premierleague.com/api/fixtures` endpoint. Note the `tables` list parameter, is required in cases where the endpoint returns multiple tables. The tables will be uploaded to the `fpl_raw` dataset in GCP via the `fpl-staging` storage bucket. You will also need to specify a key_file for connecting to GCP locally but I will add more details on that in Article 2.

### Testing Locally

> You can check out the full notebook here: [ingest/notebook.ipynb](https://github.com/matthh9797/gcp-ingest-template/blob/fantasy_premierleague/ingest/notebook.ipynb)

Now we have set up our config, we can look at the data in our development notebook. Of course, in practice there were several iterations of downloading the data and inspecting it before setting this up properly.

{{< img src="images/inititalise-fpl-class.png" >}}

So our class has been initialised by reading our `config.yaml` file into a python dictionary and initialising our class with it. Now we can run the extract method and look at the resulting object. The extract method returns a dictionary of dataframes, the keys being the names of the tables and the values of the data in the form of a dataframe.

{{< img src="images/extract-data.png" >}}

The elements table is a fact table which has a row per Premier League player. By fact table I mean one row per player with the most recent stats as opposed to a dimension table which would have all of the historical gameweek data for each player.

{{< img src="images/elements-table.png" >}}

The fixtures table returns Premier League fixtures and results data. If the game has been played there is a nested table with match stats. As I mentioned above, this is great data to play about with as it has a variety of structures and the data is high quality and totally free.

{{< img src="images/fixtures-table.png" >}}

So, we have successfully downloaded and restructured our data in a local notebook, now lets upload it into a BigQuery table by using the `run()` method. I am going to override the default config which will run daily by adding two extra tables to download from the API, element_types and teams. These tables are immutable during the season so there is not need to add them to our daily runner.

{{< img src="images/run-fpl-ingest.png" >}}

And success. Our API tables have been uploaded into BigQuery via Cloud Storage and are ready to be analysed. You may have noticed in the above prints a comment about schema files being detected and used for upload. Schema files are used to tell BigQuery about the field data types, descriptions etc. Again, I will discuss this further in Article 3.

{{< img src="images/fpl_raw.png" align="center" >}}

## Next Steps ...

In this article, we have looked at setting up a data ingestion process locally. The process has been set up using generic classes so that we can repeat our logic easily in the future. As mentioned in the introduction, everything can be working perfectly on your local machine but it does not mean it is going to be easy to productionize and automate the process. In the next article we will look at how you can use Flask, Cloud Run, Cloud Scheduler and Docker to schedule daily runs of our code.

[^1]: Image by <a href="https://www.freepik.com/free-photo/html-css-collage-concept-with-person_36295463.htm#query=python%20code&position=11&from_view=search&track=ais">Freepik</a>
[^2]: Article for more information on the Fantasy Premier League API: [www.medium.com/@frenzelts/fantasy-premier-league-api-endpoints](https://medium.com/@frenzelts/fantasy-premier-league-api-endpoints-a-detailed-guide-acbd5598eb19)
<!-- [^3]: I believe setting up code this way is always a good idea because it drives efficiency and continuous improvement, however, in the quick results driven world we live in, it can be difficult to find an employer who appreciates the long term benefits of this. -->
