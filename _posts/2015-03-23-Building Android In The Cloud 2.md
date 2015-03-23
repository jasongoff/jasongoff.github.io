---
layout: post
title: "Building Android In The Cloud Part 2 - Jenkins"
date: 2015-03-23 16:42:00
comments: true
---

This is Part 2 in a series focusing on building Android apps using Maven and Continuous Integration.
My [previous post]({% post_url 2015-03-23-Building Android In The Cloud 1 %}) looked at setting up Maven and overcoming the Android dependency issues.

In this post, I'm going to discuss how to get Android builds working in Jenkins.  This isn't a detailed Jenkins How-To, and assumes some prior experience.

## Setting up Jenkins
After [installing Jenkins](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins), there are a number of additional configuration steps needed to get Maven-based Android builds working successfully.

#### Install Plugins
Firstly, I need to install the plugins that Jenkins needs.  In the Manage Jenkins - Manage Plugins, I search for *Android* and install the [Android Emulator Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Android+Emulator+Plugin).  Amongst other features, this plugin will run an emulator if required, detect and install any required Android SDKs, and can run monkey and other post-build tasks.  It's worth reading up on what this plugin is capable of, but for the purposes of this post I need it in order to find and install the correct SDKs.

#### Set up the Jenkins Environment
From the Manage Jenkins screen, I select Configure System and set the Android SDK path.  I set this to the location on my development machine where I keep the SDK. I also check the box so that Jenkins will use the SDK tools to download additional Android components if required.  
In this example that is unlikely to occur as I have already built the project using IntelliJ pointing to the same SDK path.  This will have more of a bearing when setting up Jenkins on a dedicated build server.

#### Create a Maven Build Project
The next step is to create a Maven Project.  I give the project a name and set up a few defaults such as number of builds to keep, the details for the source code management tool, a build schedule, and the like.

In the Build Step, I configure the Maven goals I would like the project to execute, in this case ``clean package``.

And that's it.  Triggering a build will result in the Maven goals being executed and the Android APK package will appear in the ``target`` folder of the Jenkins workspace.

![Jenkins Build Output]({{ site.baseurl }}/images/jenkins-build-output.png)

This works because my local development machine already has the Maven dependencies installed in the local repository (see my [previous post]({% post_url 2015-03-23-Building Android In The Cloud 1 %}) for more details).  When configuring Jenkins on a dedicated build server, I will have to ensure the SDK dependencies are installed prior to running the build.  More on that to come.
