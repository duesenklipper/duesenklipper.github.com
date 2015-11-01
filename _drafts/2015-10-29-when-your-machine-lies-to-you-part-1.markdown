---
layout: post
title:  "When your machine lies to you, Part 1: Tomcat, JMX, RMI, and classloading"
categories: java tomcat rmi debugging fail jmx everybodylies
---
To quote the great debugging genius Dr House:

> Everybody lies.

We would like the tools we use to have useful, non-lying debugging mechanisms. Unfortunately, these
tools are built by programmers. Programmers, being a subset of Everybody, also lie, and thus the tools they build often
have a somewhat fleeting affair with what we think of as truth.

In what I hope (fear?) will become a series, I'd like to recount various occasions when my tools lied to me, or at least
tried to make the truth very non-obvious. Since I spend the majority of my working time somewhere in JVM-land, much of
this will be about Java.

If you're coming here to see the solution without the possibly entertaining story of how I got there,
[here's the tl;dr](#tldr).

## The case of the disabled classloader

Recently I started to work on a somewhat old J2EE project that had gone through a number of phases. It had some old
RMI-based parts, lots of SOAP webservices, a small web UI, a big Oracle database, and other unpleasant things.

After taking some time to get everything set up locally I deployed everything into Tomcat and started it from the
command line with `$ catalina.sh run`. It worked.

But I would like to deploy and run things from my IDE, in this case IDEA. So, set up the Tomcat instance in IDEA, tell
it to deploy the appropriate `WAR`s and launch:

    Caused by: java.lang.ClassNotFoundException: foo.customer.SomethingSomethingRmiInterface
     
That's weird. This class is definitely present.

    Caused by: java.lang.ClassNotFoundException: foo.customer.SomethingSomethingRmiInterface
        (no security manager: RMI class loader disabled)
 
Okay... A classloader being disabled would explain not finding a class. But why is there no security manager? Surely
IDEA doesn't tell Tomcat to _not_ load a security manager.

Well, let's make IDEA give `catalina.sh` the `-security` switch so it will set up a security manager for us.

    log4j:WARN No appenders could be found for logger (org.apache.catalina.startup.Catalina).
    log4j:WARN Please initialize the log4j system properly.
    
What? You were logging just fine a minute ago! Okay, let's edit `catalina.policy` so you can actually read your own
`log4j.properties`.

    Class loader creation threw exception
        java.lang.NoClassDefFoundError: Could not initialize class java.util.logging.LogManager

Screw this, you're supposed to be using log4j, not JDK logging. The `-security` switch is not getting me anywhere and I
don't feel like messing around with the logging configuration. Launching from the command line works without that
switch, so you'd think IDEA could launch an RMI-using Tomcat without it too.

My gut feeling says something else is going wrong here. Time to look at what IDEA is doing differently from what I do
on the command line. We both use `catalina.sh` to start Tomcat, and that is a `/bin/sh` script. So let's just add `-x`
to the shebang to see what is really going on.

No screamingly obvious difference to be found, except... Hey. That's not my Tomcat's webapp directory IDEA is using,
it's something under `~/.IntellijIDEA15/`. I hadn't used Tomcat's ability to split its installation directory
(`$CATALINA_HOME`) and its work/deployment directory (`$CATALINA_BASE`) in a long time, so that surprised me for a
moment. Still, could there be a problem with the configuration created by IDEA in its directory?
 
## <a name="tldr"></a>TL;DR
