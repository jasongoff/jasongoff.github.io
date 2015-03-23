---
layout: post
title: "Building Android In The Cloud Part 1 - Maven"
date: 2015-03-23 06:52:00
comments: true
---

For a while I have been building all my Android projects locally, using the standard Ant build scripts inside IntelliJ.  
Recently I decided to try to step up my build process and use Continuous Integration, a dependency management system, and JVM-based unit tests.
This series of articles provides some insight into the method used, and the various failures I encountered on the way.

For the purposes of illustration I will be creating a very simple Android application consisting of 3 activities.

Dependency management will be provided by Maven, Jenkins will be used as the CI server, and JVM unit-testing will be achieved using [Roboelectric](http://robolectric.org).

Before getting everything running in the cloud, I set about getting everything up and running on my local development machine.

My set-up:  
* MacBook Air running OSX Yosemite.  
* IntelliJ 14  
* Tomcat 7  
* Jenkins


## Managing the Dependencies with Maven
I opted to use Maven to manage the dependencies of the project, solely because I have more experience with it than I do with Gradle.  Migrating to Gradle is a challenge for another day.

The first thing to do is create a new Maven project.  Rather than having to set up all the Android folder structures by hand, there is an excellent set of Maven archetypes [available on GitHub](https://github.com/akquinet/android-archetypes).

![Adding Archetype]({{ site.baseurl }}/images/quickstart-archetype.png)

This creates a basic POM file that makes use of the [Android Maven plug-in](http://simpligility.github.io/android-maven-plugin/) and defaults to Android API level 16 (4.1.1.4).

In this example, I want to build against API 19 (4.4.2), so I need to update the Maven POM to reflect this.  While I'm here, I will also update the Android Plug-In version to the latest one.  

```xml
<properties>
	<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	<platform.version>4.4.2</platform.version>
	<android.plugin.version>3.8.0</android.plugin.version>
</properties>
```
Further down in the POM file, I also need to change the target platform version in the Android Maven Plugin configuration to 19.

```xml
<plugin>
	<groupId>com.jayway.maven.plugins.android.generation2</groupId>
	<artifactId>android-maven-plugin</artifactId>
	<configuration>
   		<sdk>
			<platform>19</platform>
		</sdk>
	</configuration>
</plugin>
```
Having done this, I should be able to build the sample application that the archetype creates, but no, the build fails.

```
Downloading: https://repo.maven.apache.org/maven2/com/google/android/android/4.4.2/android-4.4.2.pom
[WARNING] The POM for com.google.android:android:jar:4.4.2 is missing, no dependency information available
Downloading: https://repo.maven.apache.org/maven2/com/google/android/android/4.4.2/android-4.4.2.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 4.903 s
[INFO] Finished at: 2015-03-23T11:56:00+00:00
[INFO] Final Memory: 12M/28M
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal on project mavensample: Could not resolve dependencies for project uk.me.goff.sample:mavensample:apk:1.0-SNAPSHOT: Could not find artifact com.google.android:android:jar:4.4.2 in central (https://repo.maven.apache.org/maven2) -> [Help 1]
[ERROR] 
```

## Problem: No Android Dependencies in Maven Central
Well, that's not strictly true.  There are *some* dependencies in Maven Central, however there's nothing beyond API 16.  Google do not release the Android JARs into Maven Central, ones that are there have been done by others.  
The solution to this particular problem is to install the Android JAR from my local SDK installation into the local Maven repository. So from my shell, I can run the following:

```
mvn install:install-file \
 -Dfile=$ANDROID_HOME/platforms/android-19/android.jar \
 -DgroupId=com.google.android \
 -DartifactId=android \
 -Dversion=4.4.2 \
 -Dpackaging=jar \
 -DgeneratePom=true
``` 
Having deployed the JAR to satisfy the dependencies, the build succeeds.

In my next post, I'll discuss getting this build running in Jenkins.

