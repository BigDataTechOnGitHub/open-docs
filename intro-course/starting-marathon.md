---
layout: doc
title: Exercise 4 – Starting Marathon
---

In Linux the cron utility manages all of the
processes running on your system.  The Mesosphere stack uses Marathon to manage the processes and services.
Marathon is the technology that plays the role of ``init`` in the analogy of an operating system.
Marathon provides both a simple GUI and an extensive REST API that you can work with if you need
more capability.

In this exercise:

1. Start Marathon.
2. Explore the Marathon GUI.
3. Install an app and start a simple Python web server by using Marathon.
4. Test that the Python web server is live.

Video Lecture
-------------

{% mesos_video Mesos-Intro-Lecture-4 %}

Quick Reference
---------------

Start Marathon by using the ``service`` command:

{{ mesos.code('ex4/start_marathon.sh-session') }}

Go to ``http://192.168.33.10:8080/`` to view the Marathon GUI.
From the GUI, install a new app that Marathon will run.  In this example, we start by using the ``python -m SimpleHTTPServer`` app.:


{{ mesos.code('ex4/test_python_http.sh-session', section='start') }}

Marathon and Mesos give you direct access to the ``stderr`` (standard error) and ``stdout`` (standard out) files for every process.  You can     use ``curl`` to view these files and see that they are the ``SimpleHTTPServer``'s logs, which you would normally see on your terminal when you start it:

{{ mesos.code('ex4/test_python_http.sh-session', section='curl') }}

Further Study
-------------

* Try some other servers that you want to run.  For example, ``Flask``, ``Ruby on Rails``, a database, or a chat server .

