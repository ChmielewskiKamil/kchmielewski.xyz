+++
title = 'Golang Noob Questions'
date = 2024-10-07T20:52:57+02:00
draft = true
tags = ["golang"]
author = "Kamil Chmielewski"
description = "Questions that every person new to go will eventually ask. At least I certainly did."
+++

I'm a noob and you are most likely a noob as well. [How not to be a noob](https://youtu.be/-v8pD0d5Bmk?si=kuQPQw0US61cDxOq)? 
Well, you have to ask a question (keep it to yourself) and find an answer for it (bonus points if you do it all by yourself).
While learning golang I certainly had and still have many unanswered questions. Some of which I've managed to answer.
When lurking for answers I've learned one thing: People (and LLMs) like to overcomplicate stuff a lot. Certainly I'm
not going to install an external library just to do a health check on my website. I'm not a web developer but I can guess
that `curl` would be enough to do the job, I just have to figure it out.

<!--more-->

Content here.

### Invalid URLs return 200 OK by default - How to create 404 handler with net/http package ServeMux?

[People like to do crazy things](https://stackoverflow.com/questions/9996767/showing-custom-404-error-page-with-standard-http-package)
while it turns out that ...

### The simplest health check of a golang web application

`curl "http://YOUR:SERVER:IP:HERE:8080/__heartbeat__" -s -o /dev/null -w %{http_code}`

### How to deploy golang binary without Docker?

### How to structure your app with template/html?

[StackOverflow explanation of the problem space](https://stackoverflow.com/a/69244593)
