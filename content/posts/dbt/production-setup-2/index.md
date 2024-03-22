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

DBT already has very good logging from run results so why bother extracting the information this way and using it for alerting? From my experience if you leave checking logs as a manual task slowly over time you will forget to do so. Therefore it is better to work towards an easily digestable report which can be sent to us on a daily basis alerting us of test passes, warnings and most importantly failures. 

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

## Project Structure

```
project
│   app.py
|   handlers.py  
│
└───utils
│   │   alert.py
│   │
└───templates
│   │   daily-report.md
│   │   exception.md
│   │
│   └───success
│       │   run.md
│       │   test.md
│       │   ...   │
│   └───error
│       │   run.md
│       │   test.md
│       │   ...
|
└───dbt
│   │   ...
```

## Set up a Webhook URL

Discord, Slack and other messaging apps allow messages to channels via webhooks. Each has their own unique syntax for but in general the setup process is very similar. You can setup a discord webhook url easily by following the linked articles.

- [Discord Webhooks](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)
- [Slack Webhooks](https://api.slack.com/messaging/webhooks)

We can post messages to our messaging channel using the `requests` library in python by simply sending a post request. In this article I will use discord as an example and use the [Discord Markdown Syntax](https://support.discord.com/hc/en-us/articles/210298617-Markdown-Text-101-Chat-Formatting-Bold-Italic-Underline-) to format messages. 

```python
import os
import requests
from dotenv import load_dotenv

# WEBHOOK_URL saved in .env file
load_dotenv()

WEBHOOK_URL = os.environ.get("WEBHOOK_URL")

requests.post(WEBHOOK_URL, {"content": "# Hello World :wave:"})
```
<br>

{{< img src="images/hello-world-discord.png" >}}

## Create an Alert class

To crunch the results of the dbt run class into a useful logging message we need to be able to handle complex templating and since we are using DBT why not use the templating language which powers DBT [jinja2](https://jinja.palletsprojects.com/en/3.1.x/). Below is a logging message templated using jinja2 to handle the run results.

```python
import requests
from jinja2 import Environment, FileSystemLoader

class Alert:
    """
    A class for sending alerts to a webhook.
    """
    def __init__(self, webhook_url: str, env: str = 'templates') -> None:
        self.WEBHOOK_URL = webhook_url
        self.env = Environment(loader=FileSystemLoader(env))
    
    def render(self, template: str, **kwargs):
        """
        Render the results of a jinja template with kwargs
        """
        template = self.env.get_template(template)
        return template.render(**kwargs)

    def log(self, template, **kwargs):
        """
        Log the results of a rendered template to a webhook
        """
        output_from_parsed_template = self.render(template, **kwargs)
        return requests.post(self.WEBHOOK_URL, {"content": output_from_parsed_template})  
```

## Examine the run results

We need import two classes from the main DBT CLI that will allow us to invoke DBT and examine the results, these are `dbtRunner` and `dbtRunnerResult`. `dbtRunner` is the class we can use to invoke DBT and `dbtRunnerResult` is the class which is returned as the result. The following code executes `dbt run` and returns the results as a variable named `res`.

```python 
from dbt.cli.main import dbtRunner, dbtRunnerResult

dbt = dbtRunner()

res: dbtRunnerResult = dbt.invoke(['run'])
```

`dbtRunnerResult` is an iterable class which contains all of the metadata from the model run results. We can iterate over `res` to log useful information from the results of our model run.

```python
for line in res.result:
    logging.info(f"{line.status.upper()} {line.node.name} [{line.message}]")
```

```cmd
2023-12-06 08:53:17,753 [INFO] SUCCESS my_first_dbt_model [CREATE TABLE (2.0 rows, 0 processed)]
2023-12-06 08:53:17,754 [INFO] SUCCESS my_second_dbt_model [CREATE VIEW (0 processed)]
```

## Smart logging with jinja2 + python

We can write the following python code to log results to discord 

```python
def handle_test(res: dbtRunnerResult, discord: Alert):
    if res.success:
        warnings = [x for x in res.result if x.status == 'warn']
        passes = [x for x in res.result if x.status == 'pass']
        discord.log('successes/test.md', **{'warnings': warnings, 'passes': passes})
    else:
        failures = [x for x in res.result if x.status == 'fail']
        discord.log('errors/test.md', **{'results': failures})
```

```md
## Test Results {% if (warnings | length) > 0 %}:warning:{% else %}:white_check_mark:{% endif %}

PASS = {{ passes | length }}, WARN = {{ warnings | length }}, FAIL = 0, TOTAL = {{ (passes | length) + (warnings | length) }}

{%- if (warnings | length) > 0 %}
### Test Warnings

{%- for r in warnings %}
- [{{ r.status | upper }} test] {{ r.node.name}}: [{{ r.message }}]
{%- endfor -%}

{%- endif -%}
```
<br>

Now running:

```python
logging.info('Running: dbt test')
res: dbtRunnerResult = dbt.invoke(['test'] + cli_args + target_arg)
handle_test(res, discord)
```

{{< img src="images/dbt-test-logging.png" >}}

## Error Handling

The dbtRunnerResult class does not raise errors or handle results of invocations by itself. Therefore we need to create some functionality to handle the different scenarios for results. If you reference the [Programmatic Invocations](https://docs.getdbt.com/reference/programmatic-invocations) part of the DBT documentation you will see a table like below which explains what the class will return under different scenarios.

| Scenario                                                                                    | CLI Exit Code | success | result            | exception |
|---------------------------------------------------------------------------------------------|---------------|---------|-------------------|-----------|
| Invocation completed without error                                                          | 0             | True    | varies by command | None      |
| Invocation completed with at least one handled error (e.g. test failure, model build error) | 1             | False   | varies by command | None      |
| Unhandled error. Invocation did not complete, and returns no results.                       | 2             | False   | None              | Exception |

We can handle these situations programmatically like so:

```python
LOOKUP_HANDLER = {
    'freshness': handlers.handle_freshness,
    'snapshot': handlers.handle_snapshot,
    'test': handlers.handle_test,
    'run': handlers.handle_run,
    'build': handlers.handle_build
}


def handle_dbt_result(command: str, res: dbtRunnerResult, alert: Alert):
    """
    Handle the result of a dbt command using the lookup of handler functions
    @param command dbt command (e.g. dbt run command is 'run')
    @param res: dbt runner result object
    @param alert: Alert object
    @return if success return rendered report, otherwise raise exception
    """

    if res.success:
        logging.info('Invocation completed without error')
        return LOOKUP_HANDLER[command](res, alert)
    else:
        if res.exception is None:
            LOOKUP_HANDLER[command](res, alert)
            raise 'Invocation completed with at least one handled error (e.g. test failure, model build error)'
        else:
            alert.log('exception.md', **{'result': res})
            raise f'Unhandled error. Invocation did not complete, and returns no results. Exception: {res.exception}'
```

## Add daily report logging to dbt app

```python
# Initialise a alerting client 
WEBHOOK_URL = os.environ.get('WEBHOOK_URL', '')
if WEBHOOK_URL == '':
    logging.warning('No WEBHOOK_URL environment variable, alerts will not be sent.')
discord = Alert(WEBHOOK_URL)

...

# Compile reports from the dbt build results
res: dbtRunnerResult = dbt.invoke(['build'] + cli_args + target_arg)
test_report, run_report = handle_dbt_result('build', res, discord)

...

# Combine test results to create a daily report
response = log_daily_report(run_report, test_report, freshness_report, discord)
```

{{< img src="images/dbt-daily-report.png" >}}

## Conclusion

{{< alert type="info" >}}
Please leave a message if there are any other attributes you find useful or even better open up a PR on this articles repository if you have some suggestions to improve the code. 
{{< /alert >}}


[^1]: <a href="https://www.freepik.com/free-photo/reminder-popup-bell-notification-alert-alarm-icon-sign-symbol-application-website-ui-purple-background-3d-rendering-illustration_24598564.htm#query=alerting&position=1&from_view=search&track=sph&uuid=01b719d3-2c51-4a31-a3e9-3d9dd2fe37b2">Image by mamewmy</a> on Freepik