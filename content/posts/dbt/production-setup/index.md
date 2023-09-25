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

You can follow the steps in my other article [Local Environment Setup For DBT With GCP Using Conda]({{< ref "posts/dbt/local-setup/index.md" >}}) to set up your local dbt project. Optionally, I like to rename my DBT project folder to a folder named dbt using `mv {{YOUR_PROJECT_NAME}} dbt`.

{{< alert type="info" >}}
Git Commit Checkpoint: git commit -m "dbt init"
{{< /alert >}}

## Testing Deployment Locally



```bash
gcloud auth application-default login
```


