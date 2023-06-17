---
title: "Local Environment Setup For DBT With GCP Using Conda"
date: 2022-04-16T09:09:54+01:00
description: Setting up a local DBT GCP environment using conda
menu:
  sidebar:
    name: Local Setup GCP
    identifier: dbt-local-setup
    parent: dbt
    weight: 10
hero: images/dbt-gcp-logo.png
tags: ["DBT","Setup","Conda", "GCP"]
categories: ["Basic"]
---

[This article is also on medium.com](https://medium.com/@matthh9797/setting-up-a-local-environment-for-dbt-bigquery-using-conda-60a207617053)

# Setting up a local environment for DBT (BigQuery) using conda

Recently dbt has been increasing in popularity as a structured and robust framework for writing SQL logic. Although the [dbt documentation](https://docs.getdbt.com/) is a great resource, it is not too specific on local configuration. So here is a quick article on setting up a local dbt-bigquery environment with anaconda that works on both Windows and a Mac …

## Introduction

Before we begin, I assume you already have:

 - The [gcloud CLI](https://cloud.google.com/sdk/docs/install) installed and configured.
 - [Anaconda](https://www.anaconda.com/products/distribution) installed and configured.
 - You have a billable GCP project setup.

In this tutorial we are going to create a conda environment called dbt-demo . Once the tutorial is complete you can create a new conda environment for your own projects by taking a copy:

```cmd
conda env export -n dbt-demo -f /path/to/environment.yml
conda env create -n <NAME_OF_ENVIRONMENT> -f /path/to/environment.yml
```

## Setting up a conda environment

To setup a conda environment for dbt-bigquery run the following code on the anaconda prompt:

```cmd
conda create --name dbt-demo pip
conda activate dbt-demo
pip install dbt-bigquery
```

When the installation is complete run to confirm dbt is working run `dbt --version`. The output should look like:

```cmd
installed version: [installed version of dbt] (e.g. 1.0.4)
latest version: [latest version of dbt] (e.g. 1.0.4)
Up to date!
Plugins:
- bigquery: 1.0.0 - Up to date!
```

## BigQuery Setup

To initialize a new dbt project, it must be connected to a GCP project and BigQuery dataset within that project.

Assuming you have a billable project for GCP already setup, open up the BigQuery tab and create a new dataset for the project called `dbt-demo`.

### Authentication

Once you have created a new dataset you will need to create a service account (unless you would like to use oauth). This is the authentication used to connect locally to GCP.

<img src="Https://drive.google.com/uc?export=view&id=1B72WDTy_6HHUPY0wXzYgQ-i2NXVeuToH">


When you have a service account setup, you will need to create a new json key and download it locally.

<img src="Https://drive.google.com/uc?export=view&id=1B7czVHYioQoLeUXy6XM_CGf_x2i1kB7c">

## Configuring your profile

When you install dbt it will create a hidden configuration file in `~/.dbt/profiles.yml`. For each dbt project you create a new block of code is required within this file with all of the details needed to connect to your data warehouse, in this case, bigquery. The dbt CLI helps to fill out these details by prompting you to choose an option for each of the required configurations.

> **Tip:** You can open up the contents of profiles.yml on a text editor by running `code ~/.dbt/profiles.yml` from your default directory

To create a new dbt project first make a new directory called `dbt-demo` on your computer. Next, open up a anaconda/command prompt and `cd` into the new directory:

```cmd
conda activate dbt-demo
cd /path/to/dbt-demo
```

Once you are in the repository, initialize a new dbt project by running:

```cmd
dbt init dbt_demo
```

When you run this you will be prompted with a number of questions, these will be used to fill out the profiles.yml configuration file. We want the final block in profiles.yml to look like below, so fill out the questions accordingly:

```yml
dbt_demo:
    outputs:
        dev:
            dataset: dbt_demo
            fixed_retries: 1
            keyfile: /Users/<YOUR_USER>/Downloads/<YOUR_KEY_FILE>.json
            location: <YOUR_LOCATION> (e.g. EU)
            method: service-account
            priority: interactive
            project: <YOUR_GCP_PROJECT>
            threads: 4
            timeout_seconds: 300
            type: bigquery
    target: dev
```

When you have filled this in the dbt structure will appear in your repository and you are ready to start using dbt locally. Happy modeling!
