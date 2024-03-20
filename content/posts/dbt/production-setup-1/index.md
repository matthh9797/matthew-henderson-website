---
title: "Finally, a better way to deploy DBT on Google Cloud!"
date: 2022-04-16T09:09:54+01:00
description: Setting up a Production GCP environment for DBT
menu:
  sidebar:
    name: GCP Production Setup 1
    identifier: dbt-production-setup-1
    parent: dbt
    weight: 20
hero: images/production.jpg
tags: ["DBT","Production", "GCP", "Cloud Run", "Flask"]
categories: ["Intermediate"]
summary: >
  For the last year or so I've been looking for a good way to productionise DBT pipelines on Google Cloud Platform but I've been frustrated by the solutions I've found on the web.
---

All the code for this article can be found in the template repository linked below and you can also read this artlice on my Medium.

- Code: [github.com/matthh9797/dbt-cloud-run-template](https://github.com/matthh9797/dbt-cloud-run-template)
- You can also read this article on Meidum: [medium.com/@matthh9797/finally-a-better-way-to-deploy-dbt-on-google-cloud](medium.com/@matthh9797/finally-a-better-way-to-deploy-dbt-on-google-cloud) 

For the last year or so I've been looking for a good way to productionise DBT pipelines on Google Cloud Platform but I've been frustrated by the solutions I've found on the web. Either, they seem far too complicated or they involve multiple programming languages. I am not totally against using multiple programming langauges in one project but since DBT is a python command line tool it would surely make sense to use a python solution to host it. 

To my delight, I recently stumbled upon a new release, as of DBT 1.5 you can invoke DBT using a python programme, see [programmatic invocations](https://docs.getdbt.com/reference/programmatic-invocations). This means that we can create a relatively simple Flask application which can be deployed as a Cloud Run service to host our DBT pipelines. Finally, a better way to deploy DBT on Google Cloud!

{{< img src="images/dbt-deployment.png" >}}

## Pre-requisites

There are several ways to recreate this article but to follow along line for line you will need: [^1]

 - The [gcloud CLI](https://cloud.google.com/sdk/docs/install) installed and configured.
 - [Anaconda](https://www.anaconda.com/products/distribution) installed and configured.
 - You have a billable GCP project setup.
 - [Docker](https://www.docker.com/)
 - [VS Code](https://code.visualstudio.com/)
 - [VS Code Extension - Cloud Code](https://cloud.google.com/code) (For local development)

## Getting Started

Let's create a new conda environment with dbt-bigquery and Flask.

```bash
conda create -n dbt-cloud-run pip
conda activate dbt-cloud-run
pip install dbt-bigquery
pip install google-cloud-logging
pip install Flask
```

## Hello World Cloud Run App

Now create a new python flask project, using Cloud Code by clicking 'Create New Cloud Run App', this will create a project with a folder structured for a Flask application. Test the app by starting a Docker Daemen (you can do this by opening Docker Desktop) and check the Hello World app is working by clicking 'Run App on Local Cloud Run Emulator' on the Cloud Code extension.

If your app is working properly you should be able to view the home page at [http://localhost:8080/](http://localhost:8080/). You can view detailing logs on the VS Code OUTPUT tab by clicking 'Cloud Run: Run/Debug Locally - Detailed' including HTTP requests.

{{< alert type="info" >}}
You can remove the folders/files for the Getting Started app after testing. However, personally I like to keep them for a simple way to check if the app is running.
{{< /alert >}}

{{< img src="images/cloud-run-hello-world-app.png" >}}

## Add DBT to your Flask App 

{{< alert type="warning" >}}
**Update (2023-11-14)** This section has been updated from the orginal version of the article to be more secure, instead of creating a temporary service account, we can use the [gcp auth addon](https://minikube.sigs.k8s.io/docs/handbook/addons/gcp-auth/) to connect with GCP locally.
{{< /alert >}}

You can follow the steps in my other article [Local Environment Setup For DBT With GCP Using Conda]({{< ref "posts/dbt/local-setup/index.md" >}}) to set up your local dbt project. I like to rename my DBT project folder to a folder named dbt.

{{< alert type="info" >}}
I like to name the folder containing my DBT project `dbt`, note, the name of this folder does not affect your DBT project configuration.
{{< /alert >}}

For production deployment create a file called `profiles.yml` inside you dbt project directory and copy the project configuration from the default dbt profiles at `~/.dbt/profiles.yml`.

```yml
YOUR_DBT_PROJECT:
  target: dev
  outputs:
    dev:
      dataset: YOUR_DATASET
      job_execution_timeout_seconds: 300
      job_retries: 1
      location: EU
      method: oauth
      priority: interactive
      project: YOUR_PROJECT
      threads: 4
      type: bigquery
    prod:
      dataset: YOUR_DATASET
      job_execution_timeout_seconds: 300
      job_retries: 1
      location: EU
      method: oauth
      priority: interactive
      project: YOUR_PROJECT
      threads: 4
      type: bigquery
```

{{< alert type="info" >}}
Usually, you will have at least one more profile, for example, `prod` for your production data 
{{< /alert >}}

## Authenticate with GCP from your Local Container

To authenticate your local cloud code container with GCP first login with your gcloud credentials and set your project.

```cmd
gcloud auth application-default login
gcloud config set project YOUR_PROJECT
```

Next, enable the [GCP Auth addon](https://minikube.sigs.k8s.io/docs/handbook/addons/gcp-auth/) in your VS Code workspace setttings.

```yaml
{
    "cloudcode.useGcloudAuthSkaffold": true
}
```

## Add DBT Deployment Scripts

In `app.py` add the following imports to the top of your script and set up google cloud logging, this will ensure that your DBT logs are sent to Google Cloud Logging.

```python
import os
import logging
import json
import os

from flask import Flask, request, escape, render_template
import google.cloud.logging
from dbt.cli.main import dbtRunner, dbtRunnerResult


client = google.cloud.logging.Client()
client.setup_logging()
```

Now add an endpoint to your app to run a certain set of DBT commands programmatically. The following snippet takes a POST request and reads the optional parameter `target` which defaults to dev, then runs `dbt source freshness` and `dbt build`. 

{{< alert type="info" >}}
You might want to buld on this by, for example, adding another endpoint called weekly which is invoked every week and add the `--full-refresh` flag to fully refresh any incremental models
{{< /alert >}}

```python
@app.route('/daily', methods=['POST'])
def daily():
    """DBT Daily Runner."""

    try:

        json = request.get_json(force=True) # https://stackoverflow.com/questions/53216177/http-triggering-cloud-function-with-cloud-scheduler/60615210#60615210
        target = escape(json['target']) if 'target' in json else 'prod'

        # initialize
        dbt = dbtRunner()

        # create CLI args as a list of strings
        cli_args = ["--project-dir", "dbt", "--profiles-dir", "dbt"]
        target_arg = ['--target', target]
 
        logging.info('Running: dbt source freshness')
        res: dbtRunnerResult = dbt.invoke(['source', 'freshness'] + cli_args + target_arg)

        logging.info('Running: dbt build')
        res: dbtRunnerResult = dbt.invoke(['build'] + cli_args + target_arg)

        ok = 'DBT Run Successfully'
        logging.info(ok)
        return ok     
    
    except Exception as e:
        logging.exception(e)
        return e
```

Amend your dockerfile by using the image that [dbt labs official image](https://github.com/dbt-labs/dbt-bigquery/pkgs/container/dbt-bigquery) and add a line to run `dbt deps` after `COPY . .`.

```Dockerfile
# Python image to use.
FROM ghcr.io/dbt-labs/dbt-bigquery:1.5.6

...

# Download dbt dependencies
RUN dbt deps --profiles-dir dbt --project-dir dbt
```

To test the package downloads are working you can add a package to the `dbt/packages.yml` file like so.

```yml  
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1 # Update to current version if required
```

## Test the Deployment Locally

To test if the deployment is working locally, you can test your application locally by running it again on the Cloud Run Emulator. When the app is running you can test it by invoking the local endpoint, I like using [https://reqbin.com/](https://reqbin.com/) for this. Send a post request to [http://localhost:8080/daily](http://localhost:8080/daily) with the message `{"target": "dev"}`. If your application has run successfully you will get the success message returned. You can check on the detailed logs that logging is being sent to the google-cloud-logging API.

```cmd
DBT Run Successfully 
```

{{< alert type="info" >}}
It's not often the case that everything works first time, for debugging, I would recommend using the 'Debug App on Local Cloud Run Emulator' option on cloud code with breakpoints.
{{< /alert >}}

## Deploy Your App into Production

Now our app is working locally you can deploy it to Cloud Run by running the following bash script. If you have a Windows machine you may need to run this with the GCP Cloud Shell. Now that your cloud run service is deployed you can easily set up Cloud Scheduler to invoke it on a daily basis. For more information on that I would recommend checking out [Data Science on GCP](https://github.com/GoogleCloudPlatform/data-science-on-gcp/tree/edition2/02_ingest).

```bash
SERVICE_ACCOUNT=YOUR_SERVICE_ACCOUNT
SERVICE_NAME=YOUR_SERVICE_NAME # e.g. dbt-daily
REGION=YOUR_REGION # e.g. europe-west2

gcloud builds submit \
  --tag gcr.io/$(gcloud config get-value project)/${SERVICE_NAME}

gcloud run deploy ${SERVICE_NAME} --region $REGION \
    --image gcr.io/$(gcloud config get-value project)/${SERVICE_NAME} \
    --service-account ${SERVICE_ACCOUNT}@$(gcloud config get-value project).iam.gserviceaccount.com \
    --platform managed \
    --no-allow-unauthenticated
```

## Conclusion

We have just tested and deployed a simple Flask application to wrap up a DBT pipeline and run it in production. I believe this deployment is an improvement on other popular solutions I have found on the internet because, 1, it is more simple, 2, it is a python application. One of the main advantages of it being a python application is that in my opinion most people who know DBT are likely to be familiar with python as opposed to GO, for example. This means we can easily extend this application to handle the results and add things like monitoring. I plan to add more detail on this in the future so stay tuned!

[^1]: <a href="https://www.freepik.com/free-photo/motion-blur-automatic-train-moving-inside-tunnel-tokyo-japan_10824403.htm#query=automatic&position=0&from_view=search&track=sph">Image by tawatchai07</a> on Freepik