---
title: "Finally, a better way to deploy DBT on Google Cloud!"
date: 2022-04-16T09:09:54+01:00
description: Setting up a Production GCP environment for DBT
menu:
  sidebar:
    name: Production Setup GCP
    identifier: dbt-production-setup
    parent: dbt
    weight: 20
hero: images/dbt-gcp-logo.png
tags: ["DBT","Production", "GCP", "Cloud Run", "Flask"]
categories: ["Intermediate"]
---

## Pre-requisites

There are several ways to recreate this article but to follow along line for line you will need:

 - The [gcloud CLI](https://cloud.google.com/sdk/docs/install) installed and configured.
 - [Anaconda](https://www.anaconda.com/products/distribution) installed and configured.
 - You have a billable GCP project setup.
 - [Docker](https://www.docker.com/)
 - [VS Code](https://code.visualstudio.com/)
 - [VS Code Extension - Cloud Code](https://cloud.google.com/code) (For local development)

## Introduction

For the last year or so I've been looking for a good way to productionise DBT pipelines on Google Cloud Platform but I've been frustrated by the solutions I've found on the web. Either, they seem for too complicated or they involve too many programming languages. I am not totally against using multiple programming langauges in one project but since DBT is a python tool it would surely make sense to use a python solution for hosting DBT. 

To my delight, I recently stumbled upon a new release of [programmatic invocations](https://docs.getdbt.com/reference/programmatic-invocations) on the DBT docs which allows you to run DBT via a python programme. This means that we can deploy DBT into production with a regular Python Flask programme with Cloud Run. Finally, a better way to deploy DBT on Google Cloud!

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

Now create a new python flask project, using Cloud Code by clicking 'Create New Cloud Run App', this will create a project with a folder structured for a Flask application. Otherwise copy the folders/files from the repository for this article. Test the app by starting a Docker Daemen (you can do this by opening Docker Desktop) and check the Hello World app is working by clicking 'Run App on Local Cloud Run Emulator' on the Cloud Code extension.

If your app is working properly you should be able to view the home page at [http://localhost:8080/](http://localhost:8080/). You can view detailing logs on the VS Code OUTPUT tab by clicking 'Cloud Run: Run/Debug Locally - Detailed' including HTTP requests.

{{< img src="images/cloud-run-hello-world-app.png" >}}

## Add DBT to your Flask App 

You can follow the steps in my other article [Local Environment Setup For DBT With GCP Using Conda]({{< ref "posts/dbt/local-setup/index.md" >}}) to set up your local dbt project. Optionally, I like to rename my DBT project folder to a folder named dbt using `mv {{YOUR_PROJECT_NAME}} dbt`.

{{< alert type="info" >}}
Git Commit Checkpoint: git commit -m "dbt init"
{{< /alert >}}

For production deployment create a file called `profiles.yml` inside you dbt project directory and copy the project configuration from the default dbt profiles at `~/.dbt/profiles.yml`. Add a new target called `local` which is authenticated by a service key called `tempkey.json` in the root directory.

```yml
cloud_run_template:
  target: dev
  outputs:
    local:
      dataset: YOUR_DATASET
      job_execution_timeout_seconds: 300
      job_retries: 1
      location: EU
      method: service-account
      keyfile: tempkey.json
      priority: interactive
      project: YOUR_PROJECT
      threads: 4
      type: bigquery
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
```

To create the temporary service account key run the following replacing SVC_ACCT_EMAIL with your service account for DBT. If you haven't created one check out this article [Local Environment Setup For DBT With GCP Using Conda]({{< ref "posts/dbt/local-setup/index.md" >}}). Take a note of the key id so you can delete it after you have finished developing.

```cmd  
gcloud iam service-accounts keys create tempkey.json --iam-account=SVC_ACCT_EMAIL
```

Now test dbt is working by running:

```cmd  
dbt run --profiles-dir dbt --project-dir dbt --target local
```

## Add DBT Handler to App

In the Dockerfile after `COPY . . ` add a line `RUN dbt deps --profiles-dir dbt --project-dir dbt` to download any dbt packages required for your project. You can test this by adding a file called `packages.yml` to you DBT project and adding the following to download dbt utils:

```yml  
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1 # Update to current version if required
```

Now add an endpoint to your app to run a certain set of DBT commands programmaticly. The following snippet takes a POST request and reads the optional parameter `target` which defaults to dev, then runs dbt test and dbt run.

```python
@app.route('/daily', methods=['POST'])
def daily():
    """DBT Daily Runner"""
    try:
        json = request.get_json(force=True) # https://stackoverflow.com/questions/53216177/http-triggering-cloud-function-with-cloud-scheduler/60615210#60615210

        target = escape(json['target']) if 'target' in json else 'dev'

        dbt = dbtRunner()

        target_arg = ['--target', target]

        cli_args = ['--project-dir', 'dbt', '--profiles-dir', 'dbt']

        res: dbtRunnerResult = dbt.invoke(['test'] + target_arg + cli_args)
        res: dbtRunnerResult = dbt.invoke(['run'] + target_arg + cli_args)

        ok = 'DBT run successfully'
        return ok
    
    except Exception as e:
        return e
```

You will also need to add the following imports:

```python 
from flask import Flask, render_template, request, escape
from dbt.cli.main import dbtRunner, dbtRunnerResult
```

Test your app locally by again running the app on the cloud run emulator. To debug the app, run in debug mode and add checkpoints. You can test your endpoint using curl or an app like [https://reqbin.com/](https://reqbin.com/). Send a post request to [http://localhost:8080/daily](http://localhost:8080/daily) with the message `{"target": "local"}`. View the detailed logs to view the DBT output. If DBT runs successfully, you will get the success message returned:

```
DBT run successfully
```

To get simple logs, just add the following to the top of your app. You can test this works by adding a logging message and viewing the detailed logs.

```cmd 
import google.cloud.logging

client = google.cloud.logging.Client()
client.setup_logging()
```

Now we have finished local development we can delete our temporary key by running the following by replacing the SVC_ACCT_KEY_ID with the ID you noted down at the start of the session. Note in production the DBT authentication with run with OAuth 2.0 using the service account you deploy it with.

```bash 
gcloud iam service-accounts keys delete SVC_ACCT_KEY_ID --iam-account=SVC_ACCT_EMAIL
```

Now our app is working locally you can deploy it to Cloud Run by clicking 'Deploy to Cloud Run'.

## Conclusion

We have just tested and deploy a simple Flask application to Cloud Run which can be invoked with Cloud Scheduler to run DBT in production. I believe this deployment is better because it is more simple than others I have found on the internet and because it uses Flask which keeps everything in Python. This deployment can also be easily extended to include logging and monitoring which I plan to show in a future article.

