---
layout: post
title:  "When your machine lies to you, Part 1: Tomcat, JMX, RMI, and classloading"
categories: java tomcat rmi debugging fail jmx everybodylies
---

To quote the great debugging genius Dr House:

> Everybody lies.

We would like the tools we use to have useful, non-lying debugging mechanisms.
Unfortunately, these tools are built by programmers. Programmers, being a
subset of Everybody, also lie, and thus the tools they build often
have a somewhat fleeting affair with what we think of as truth.

Sometimes, these lies are accidental, or just a matter of not thinking things
through. Sometimes, it seems like the lies are almost intentional.

## A conspiracy of database and ORM

Some years ago, I was given the task of building a small [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)
application that was to serve as one of the back-ends for another, larger
application. At first glance, it was nothing out of the ordinary: JBoss, some
web services as the front interface, a DB2 database in the back, and JPA in
the form of Hibernate inbetween, and just a bit of validation and business
logic. Not overly exciting, but fairly quick to implement, and the basic
functions were working and available to the consuming application in a
short time.

Some time later, the team building one of the consumers of our webservices
started complaining about data disappearing from their objects.
