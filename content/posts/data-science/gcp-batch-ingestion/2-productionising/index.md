---
title: "Productionising the Process"
date: 2023-05-08T10:55:44+07:00
description: Productionising and automating batch ingestion processes
weight: 20
menu:
  sidebar:
    name: "2. Productionising"
    identifier: gcp-batch-ingestion-2
    parent: batchingest
    weight: 20
hero: images/hand-touching-screen-rpa-concept.jpg
tags: ["GCP","Python","Cloud Run", "Docker", "Flask"]
categories: ["Moderate"]
summary: >
  So your code is working fine locally but you don't want to be stuck running manual scripts every day. In this article, I introduce automating the process with Flask, Docker and Cloud Run.
---

## Productionising the Process

In the last article in this series we looked at how to develop a generic ETL process locally using Python and GCP. In this article, we will look at how to turn that code into a web application using Flask, package the code up using Docker, deploy it to the cloud using Cloud Run and schedule it daily using Cloud Scheduler.

If this is the first time you have tried to schedule a python program you might be thinking the same as I did, "Do I really need all of that to schedule code?". I believe over time a lot of the technologies that are involved in this process will start to be hidden away and it will become more easy to do. For example Cloud Functions are an alternative to Cloud Run which means you don't need a Docker file.

That being said it's always good to have at least a basic understanding of what each technology in the process is doing and why it is needed. In this article, I will give an introduction to each technology with practical examples of what each one is doing.

### What does it mean to deploy code to the cloud?

Before I go into each of the individual technologies it is important to think about what is happening when you deploy code to the cloud. When you deploy code to the cloud you are creating a web application with an endpoint which can be invoked by HTTP requests. To do that your application needs to know what to do with http requests, your application needs to be wrapped up as a package so it is downloadable and you will need some service to host your application on.

### The Need for Flask

With that in mind, what's the need for Flask? Flask is the python library which is going to allow your application to handle HTTP requests. And although that is a very simplified explanation of what you can do with Flask, in our case that's all we need it to do.

Flask allows you to define python functions which are triggered by HTTP requests to a URL. You can control which type HTTP requests the function can handle. Generally, the two common HTTP requests are GET in which case your function will generally show a web page and POST where your function will handle some data which is sent to it. In our case, we want our function to be able to handle POST requests with our function parameters.

In our case we only need one URL, however, it is worth noting how to invoke requests to different page paths. For example, say we want to host our web application on `www.example.com`. Since, this is the base URL of our site we can set it us like this:

```python
@app.route("/")
def home():
  return "<h1>This is the Home Page</h1>"
```

Now, if we also wanted to handle requests to a login page we can do so using the syntax:

```python
@app.route("/login")
def home():
  return "<h1>This is the Home Page</h1>"
```

### Why use Docker?

### And Cloud Run?

### What is a Service Account?

[^1]

[^1]: Image by <a href="https://www.freepik.com/free-photo/hand-touching-screen-rpa-concept_23992686.htm#query=automatic&position=0&from_view=search&track=sph">Freepik</a>