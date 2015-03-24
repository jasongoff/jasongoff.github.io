---
layout: post
title: "Building Android In The Cloud Part 3 - Into The Cloud"
date: 2015-03-23 16:42:00
comments: true
---

This is Part 3 in a series focusing on building Android apps using Maven and Continuous Integration.
My [previous post]({% post_url 2015-03-23-Building Android In The Cloud 2 %}) looked at setting up a local Jenkins server.

This time around, I'm going to look at creating a cloud-based server to host my Jenkins CI platform.

There are a number of different offerings around, and I'm not advocating any particular one.  After some research I decided to use [Bitnami's](https://bitnami.com) offering of pre-packaged server applications.  They offer the ability to deploy pre-configured applications on either [Google Cloud Platform](https://cloud.google.com) or [Amazon Web Services](http://aws.amazon.com).  Whilst Bitnami is free, if you want to use either of the cloud platforms for hosting, this may incur a modest charge.  

##Setting up the Virtual Server

The platforms differ in the way that they are set up and managed through the Bitnami portal, and both require an account to be created.  Here I'll be using Google Cloud Services.

After creating an account, I could access the Google Launch console.  The console contains a list of all the applications available to deploy to a virtual server, as well as a link to the currently configured Virtual Machines.

![Bitnami Console]({{site.baseurl}}/images/bitnami-console.png)

A search for *Jenkins* revealed that Bitnami currently package v1.605.  I selected the **Launch** option and was presented with the New Virtual Machine configuration screen.

![Bitnami Console]({{site.baseurl}}/images/bitnami-new-vm.png)


Bitnami requires access to the Google account in order to configure and launch the VMs, and on this screen I opted to add a new project rather than use one of my existing Google App Engine projects.  I chose the smallest (cheapest) footprint available: a 10Gb magnetic disk on a *g1-small* instance.  As I live in the UK, I selected Europe as the region to host my server.  Bitnami calculated out the cost of ownership of this server at $17.89 per month.  This figure is for continuous use, but it is very unlikely that my Jenkins server will be running continuously so I won't be paying this much for the service.  The pricing model for Google's offering is fairly complex, and you can find out more [here](https://cloud.google.com/compute/pricing).

Having completed the form I clicked on **Create** and the VM build was underway.  Once complete, the server detail screen was displayed, from where I could log in to the Jenkins instance, launch an SSH console, reboot or shut down the server.

![Server Detail]({{site.baseurl}}/images/bitnami-server-detail.png)

##Configuring Jenkins
Following the Application links from the Bitnami portal, I logged on to the new Jenkins instance.  The first thing to do is to install the Android Emulator plugin (see my [previous post]({% post_url 2015-03-23-Building Android In The Cloud 2 %}) for details).  Once the plugin was installed, I went to the *Manage Jenkins - Configure System* screen and set the following properties:

Leave the **Android SDK Root** empty.  
Make sure **Automatically Install Android Components** is checked.  
Under **Maven Installations** click *Add Maven* and set the name to *Maven*.  

##Creating a Build Project
Exactly as [I did on the local Jenkins install](({% post_url 2015-03-23-Building Android In The Cloud 2 %})), I created a Maven Project build on the new VM Jenkins instance.  

As well as the basic Project info for Git etc., I added the following Pre-Build Steps:

* Install Android Project Prerequisites
* Execute Shell

For the Execute Shell script, I invoked maven to install the required Android dependencies into *tomcat* user's local repository

```
mvn install:install-file \
 -Dfile=$ANDROID_HOME/platforms/android-19/android.jar \
 -DgroupId=com.google.android \
 -DartifactId=android \
 -Dversion=4.4.2 \
 -Dpackaging=jar \
 -DgeneratePom=true
```

With the basic information set up, I kicked off a build to ensure the Git repository links and Android SDK downloading worked correctly.  

The Git link and SDK download all worked perfectly, however the build failed with an error because maven could not run the ``aapt`` utility in the android SDK folder.

```
[INFO] /bin/sh: 1: /opt/bitnami/apps/jenkins/jenkins_home/tools/android-sdk/build-tools/22.0.1/aapt: not found
[ERROR] Error when generating sources.
org.apache.maven.plugin.MojoExecutionException: 
	at com.jayway.maven.plugins.android.phase01generatesources.GenerateSourcesMojo.generateR(GenerateSourcesMojo.java:576)
...
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 11.661 s
[INFO] Finished at: 2015-03-24T13:03:33+00:00
[INFO] Final Memory: 14M/41M
[INFO] ------------------------------------------------------------------------
Waiting for Jenkins to finish collecting data[ERROR] Failed to execute goal com.jayway.maven.plugins.android.generation2:android-maven-plugin:3.8.0:generate-sources (default-generate-sources) on project androidmaven: MojoExecutionException: ANDROID-040-001: Could not execute: Command = /bin/sh -c cd "/opt/bitnami/apps/jenkins/jenkins_home/jobs/Maven Android/workspace" && /opt/bitnami/apps/jenkins/jenkins_home/tools/android-sdk/build-tools/22.0.1/aapt package -m -J '/opt/bitnami/apps/jenkins/jenkins_home/jobs/Maven Android/workspace/target/generated-sources/r' -M '/opt/bitnami/apps/jenkins/jenkins_home/jobs/Maven Android/workspace/AndroidManifest.xml' -S '/opt/bitnami/apps/jenkins/jenkins_home/jobs/Maven Android/workspace/res' --auto-add-overlay -I /opt/bitnami/apps/jenkins/jenkins_home/tools/android-sdk/platforms/android-19/android.jar, Result = 127 -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
[JENKINS] Archiving /opt/bitnami/apps/jenkins/jenkins_home/jobs/Maven Android/workspace/pom.xml to uk.me.goff.androidmaven/androidmaven/1.0-SNAPSHOT/androidmaven-1.0-SNAPSHOT.pom
channel stopped
Finished: FAILURE
```

##The Unexecutable Executable
I used the Bitnami Portal, and clicked on the **Launch SSH Console** button, which opened up a terminal session in my browser.  From there I navigated to the location where Jenkins downloads its tools, and into the Android SDK folder

```
cd /opt/bitnami/apps/jenkins/jenkins_home/tools/android-sdk/build-tools/22.0.1
```

An ``ls`` of the folder revealed that the `aapt` utility was present, 

```
drwxr-xr-x 4 tomcat tomcat     4096 Mar 24 13:02 .
drwxr-xr-x 3 tomcat tomcat     4096 Mar 24 13:02 ..
-rwxr-xr-x 1 tomcat tomcat  1264873 Mar 24 13:02 aapt
-rwxr-xr-x 1 tomcat tomcat   268935 Mar 24 13:02 aidl
```

but when I tried to execute it, I got an error.

```
./aapt
-bash: ./aapt: No such file or directory
```

After much googling, I discovered that on 64-bit Linux implementations, I needed to install 32-bit libraries as `aapt` is a 32-bit binary.

```
sudo apt-get install lib32stdc++6 lib32z1
```

And finally, the project compiled successfully.

```
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 5 source files to /opt/bitnami/apps/jenkins/jenkins_home/jobs/Maven Android/workspace/target/classes
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 24.515 s
[INFO] Finished at: 2015-03-24T13:30:44+00:00
[INFO] Final Memory: 22M/54M
[INFO] ------------------------------------------------------------------------
[JENKINS] Archiving /opt/bitnami/apps/jenkins/jenkins_home/jobs/Maven Android/workspace/pom.xml to uk.me.goff.androidmaven/androidmaven/1.0-SNAPSHOT/androidmaven-1.0-SNAPSHOT.pom
channel stopped
Finished: SUCCESS
```

So there it is, Android projects building on a cloud-based Jenkins server.

In my next post, I'll be adding in unit testing with Roboelectric.


