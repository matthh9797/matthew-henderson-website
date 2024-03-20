---
title: "Add Discord monitoring to your DBT deployment"
date: 2022-04-16T09:09:54+01:00
description: Adding monitoring to your production DBT environment
menu:
  sidebar:
    name: GCP Production Setup 2
    identifier: dbt-production-setup-2
    parent: dbt
    weight: 30
hero: images/hero.jpg
tags: ["DBT","Production", "GCP", "Cloud Run", "Flask"]
categories: ["Intermediate"]
summary: >
  Setting up an alerting process is important to catch failed tests and models runs in your DBT process. In this article I will show you how to send DBT run results to Discord.
---

All the code for this series of articles can be found in the template repository linked below and you can also read this series of articles on my Medium.

- Code: [github.com/matthh9797/dbt-cloud-run-template](https://github.com/matthh9797/dbt-cloud-run-template)
- You can also read this article on Meidum: [medium.com/@matthh9797/finally-a-better-way-to-deploy-dbt-on-google-cloud](medium.com/@matthh9797/finally-a-better-way-to-deploy-dbt-on-google-cloud) 

In the previous article in this series we looked at using a simplified setup to deploy DBT as a Flask application which was tested locally and deployed to Cloud Run. In this article we look at extending this setup to have more useful logging and alerting sent to Discord.

{{< img src="images/dbt-deployment.png" >}}

## Pre-requisites

There are several ways to recreate this article but to follow along line for line you will need: [^1]

 - The [gcloud CLI](https://cloud.google.com/sdk/docs/install) installed and configured.
 - [Anaconda](https://www.anaconda.com/products/distribution) installed and configured.
 - You have a billable GCP project setup.
 - [Docker](https://www.docker.com/)
 - [VS Code](https://code.visualstudio.com/)
 - [VS Code Extension - Cloud Code](https://cloud.google.com/code) (For local development)
 - [Discord](https://discord.com/)

## Getting Started

Read and setup the code from the previous article in this series: [Finally, a better way to deploy DBT on Google Cloud!]({{< ref "posts/dbt/production-setup-2/index.md" >}})

- When you invoke dbt a class is returned which can be iterated on to get results
- For example to extract all of the run results with a warning and all of the run results with a success we can run the following code 
- 


## Examine DBT Run Results

Using programmatic invokactions we can import two classes from the main DBT CLI that will allow us to invoke DBT and examine the results, these are `dbtRunner` and `dbtRunnerResult`. `dbtRunner` is the class we can use to invoke DBT and `dbtRunnerResult` is the class which is returned as the result. DBT can be invoked using python with the following code which returns the results class as a variable called `res`.

```python 
from dbt.cli.main import dbtRunner, dbtRunnerResult

dbt = dbtRunner()

res: dbtRunnerResult = dbt.invoke(['run'])
```

We can iterate over `res` to find the results of each model run in our dbt pipeline. There are many different attributes which are returned in the `RunResult` but below is an example a few important ones for logging.

{{< alert type="info" >}}
Please leave a message if there are any other attributes you find useful or even better open up a PR on this articles repository if you have some suggestions to improve the code. 
{{< /alert >}}

```python
for line in res.result:
    logging.info(f"{line.status.upper()} {line.node.name} [{line.message}]")
```

Result:

```cmd
2023-12-06 08:53:17,753 [INFO] SUCCESS my_first_dbt_model [CREATE TABLE (2.0 rows, 0 processed)]
2023-12-06 08:53:17,754 [INFO] SUCCESS my_second_dbt_model [CREATE VIEW (0 processed)]
```

## Set Up Alerting with Discord/Slack

DBT already has very good logging from run results so why bother extracting the information this way and using it for alerting? From my experience if you leave checking logs as a manual task slowly over time you will forget to do so. Therefore it is better to work towards an easily digestable report which can be sent to us on a daily basis alerting us of test passes, warnings and most importantly failures. 

To set this up we can use a messaging channel like discord or slack. Note that although each company has it's own syntax, generally the webhook to message channels can be set up in a similar way for each. The process to create a webhook is very simple and you can follow either of the following guides for discord or slack:

- [Discord Webhooks](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)
- [Slack Webhooks](https://api.slack.com/messaging/webhooks)

## Posting Messages to Discord with Python and Jinja

We can post messages to our messaging channel using the `requests` library in python by simply sending post requests and using the [Discord Markdown Syntax](https://support.discord.com/hc/en-us/articles/210298617-Markdown-Text-101-Chat-Formatting-Bold-Italic-Underline-) to format our messages. Note I've saved my webhook URL as an environment variable, you can load environment variables locally with the package `dotenv`.

```python 
import os
import requests

WEBHOOK_URL = os.environ.get("WEBHOOK_URL")

requests.post(WEBHOOK_URL, {"content": "# Hello World :wave:"})
```

Result:

{{< img src="images/hello-world-discord.png" >}}

Now we have our webhook up and running we can use it to send reports of our DBT workflows. There are different ways to do this but since we use jinja for DBT why not use it to template our logging messages. We are going to create two templates, one for the scenario where all models run successfully and another for the scenario when a model fails and breaks our workflow.

Below is a template message for all tests running successfully, for models which are successful we simply count them and for models with warnings we print a more descriptive message about what caused the warning. We pass in two lists of results one for all passed models and another for all models with a warning.

```md
## Run Results {% if (warnings | length) > 0 %}:warning:{% else %}:white_check_mark:{% endif %}

PASS = {{ passes | length }}, WARN = {{ warnings | length }}, FAIL = 0, TOTAL = {{ (passes | length) + (warnings | length) }}

{%- if (warnings | length) > 0 %}
### Run Warnings

{%- for r in warnings %}
- [{{ r.status | upper }} test] {{ r.node.name }}: [{{ r.message }}]
{%- endfor -%}

{%- endif -%}
```

## Handle the DBT Results

When we invoke DBT programmatically with python like so `res: dbtRunnerResult = dbt.invoke(['run'])` we are returned a `dbtRunnnerResult` class which can be iterated to find the 

## Add DBT Runner Result Handlers 

```
def handle_run(res: dbtRunnerResult, WEBHOOK_URL: str):
    warnings = [x for x in res.result if x.status == 'warn']
    passes = [x for x in res.result if x.status == 'success']
    if res.success:
        return render_template(env, 'successes/run.md', **{'warnings': warnings, 'passes': passes})
    else:
        failures = [x for x in res.result if x.status == 'error']
        log_template_to_discord(env, 'errors/run.md', WEBHOOK_URL, **{'results': failures})
```

## Adding DBT Runner Result Handlers

The dbtRunnerResult class does not raise errors or handle results of invocations by itself. Therefore we need to create some functionality to handle the different scenarios for results. If you reference the [Programmatic Invocations](https://docs.getdbt.com/reference/programmatic-invocations) part of the DBT documentation you will see a table like below which explains what the class will return under different scenarios.

| Scenario                                                                                    | CLI Exit Code | success | result            | exception |
|---------------------------------------------------------------------------------------------|---------------|---------|-------------------|-----------|
| Invocation completed without error                                                          | 0             | True    | varies by command | None      |
| Invocation completed with at least one handled error (e.g. test failure, model build error) | 1             | False   | varies by command | None      |
| Unhandled error. Invocation did not complete, and returns no results.                       | 2             | False   | None              | Exception |

So we need to create a function which handles these different scenarios. For each DBT command the result set varies and we would like to create different logs per command. To handle this we can create a lookup function to handle each command and a general wrapper function to handle each of the above scenarios. 

```python 
lookup_handler = {
    'freshness': handle_freshness,
    'snapshot': handle_snapshot,
    'test': handle_test,
    'run': handle_run,
    'build': handle_build
}

def handle_dbt_result(res: dbtRunnerResult, command: str, webhook_url: str):
    """Log based DBT result"""

    if res.success:
        logging.info('Invocation completed without error')
        return lookup_handler[command](res, webhook_url)
    else:
        if res.exception is None:
            lookup_handler[command](res, webhook_url)
            raise SystemError('Invocation completed with at least one handled error (e.g. test failure, model build error)')
        else:
            log_template_to_discord(env, 'exception.md', webhook_url, **{'result': res})
            raise SystemError(f'Unhandled error. Invocation did not complete, and returns no results. Exception: {res.exception}')
```







[^1]: <a href="https://www.freepik.com/free-photo/reminder-popup-bell-notification-alert-alarm-icon-sign-symbol-application-website-ui-purple-background-3d-rendering-illustration_24598564.htm#query=alerting&position=1&from_view=search&track=sph&uuid=01b719d3-2c51-4a31-a3e9-3d9dd2fe37b2">Image by mamewmy</a> on Freepik