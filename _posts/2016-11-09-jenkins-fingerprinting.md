---
layout: post
title: Jenkins Fingerprinting
date:   2016-11-09 10:00:00
tags:
- jenkins
- Continuous Integration
---
## What is Jenkins Fingerprinting?
Jenkins Fingerprinting is the way that Jenkins keeps tracking of a file across different build and jobs.
Sometimes you have different jobs using the same produced file and it's hard to track if it's using the proper version of this file.
Fingerprinting is made for it.

*Example:* Imagine that you have the Job A and the Job B.
The Job A creates a build #1 which the result is a lib_xyz.jar.
The Job B depends on the lib_xyz.jar, and start the build #3 (I chose an arbitrary number).
With Fingerprinting you will be able to know if the build #3 used the proper lib_xyz.jar from Job A.

*but, what really is the Fingerprinting?*

The fingerprint of a file is a MD5 checksum.

*Example:*
MD5: 7b84e16c174a048c794cf9483478da52

## How does Fingerprinting works?
The fingerprint is as xml file stored in the $JENKINS_HOME/fingerprints directory on the Jenkins master.

```
fingerprints
└── 7b
    └── 84
        └── e16c174a048c794cf9483478da52.xml
```

And the xml file contains the following data:
* Timestamp
* Job name
* Build number
* MD5 checksum
* Fingerprinted file name.

```
<?xml version='1.0' encoding='UTF-8'?>
<fingerprint>
  <timestamp>2016-11-09 20:15:21.520 UTC</timestamp>
  <original>
    <name>JobA</name>
    <number>7</number>
  </original>
  <md5sum>7b84e16c174a048c794cf9483478da52</md5sum>
  <fileName>auto-versioning-lib-a-1.0.0-SNAPSHOT.jar</fileName>
  <usages>
    <entry>
      <string>JobA</string>
      <ranges>7</ranges>
    </entry>
  </usages>
  <facets/>
</fingerprint>
```

From the Jenkins UI:

![fingerprint one job]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_fingerprint_one_job.png)

If this file is used for more than one build, you will see in the usages node, all the jobs/builds that are using this file.
Something like that:

```
...
<usages>
	<entry>
		<string>JobB</string>
		<ranges>6-7</ranges>
	</entry>
	<entry>
		<string>JobA</string>
		<ranges>7</ranges>
	</entry>
</usages>
...
```

Where ranges stands for the build number, or the range of builds that this file has been used.

From the Jenkins UI:

![fingerprint one job]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_fingerprint_two_jobs.png)

## How to implement it?

### Jenkins Freestyle jobs

From the Jenkins Dashboard

1. New Item

	![Step1]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_freestyle-pipeline_step1.png)

2. Give a name for your job, i.e: JobABC, and choose freestyle job.

	![Step2]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_freestyle_step2.png)

3. In the *Post-build Actions* section, add the option "Record fingerprints of files to track usage"

	![Step3]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_freestyle_step3.png)

4. Add name or a regex to the file that you want to keep tracking with fingerprint.

	If you want to use this file in another job, you can add another build step, the "Archive the artifacts" and fill in the name like the fingerprint.

	![Step4]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_freestyle_step4.png)


### Jenkins Pipeline jobs

From the Jenkins Dashboard

1. New Item

	![Step1]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_freestyle-pipeline_step1.png)

2. Give a name for your job, i.e: JobA_pipeline, and choose pipeline job.


	![Step2]({{ site.url }}/assets/2016-11-09-jenkins-fingerprinting/jenkins_pipeline_step2.png)

3. Add groovy code to Archive & fingerprint the file:

		step([$class: 'ArtifactArchiver', artifacts: 'target/auto-versioning-lib-a-*.jar', fingerprint: true])

	Where artifacts works exactly as freestyle jobs, that is regex or the file name.


4. For better understanding, here is the complete pipeline code:

		node {
		   // Mark the code checkout 'stage'....
		   stage 'Checkout'

		   // Get some code from a GitHub repository
		   git url: 'https://github.com/rodrigozrusso/auto-versioning-lib-a.git'

		   // Get the maven tool.
		   // ** NOTE: This 'M3' maven tool must be configured
		   // **       in the global configuration.           
		   def mvnHome = tool 'M3'

		   // Mark the code build 'stage'....
		   stage 'Build'
		   // Run the maven build
		   sh "${mvnHome}/bin/mvn clean package"

			 stage 'Archive'
			 [$class: 'ArtifactArchiver', artifacts: 'target/auto-versioning-lib-a-*.jar', fingerprint: true]
		}

#### References
* https://wiki.jenkins-ci.org/display/JENKINS/Fingerprint
* For this example it was used the repository: https://github.com/rodrigozrusso/auto-versioning-lib-a
