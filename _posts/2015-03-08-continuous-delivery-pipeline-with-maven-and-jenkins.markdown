---
layout: post
title:  "Setting up a continuous delivery pipeline with jenkins and maven"
date:   2015-03-08 18:38:38
---
If you are working with Java chances are, the tools you end up using are maven as a build tool and Jenkins as a continuous integration server. You might have read a few things about the maven release plugin. However, you believe that releases should be done often and that each build should be a release candidate. So you would like for each push to result in a new version of the component. You would also prefer to avoid polluting your VCS history with commits by the Jenkins User. If all of this sounds reasonable, then you are like me. Requirements are set, now it's just a matter of making it work. Should be easy, eh?
<p>
Let's start with describing the logical flow. Continuous delivery pipeline begins with a set of unit tests followed by deploying to your artifact repository, running some wider scope tests, maybe testing with other parts of the system, proceeding with going to pre-production environment and then, finally into production.
<p>
In order to avoid changing the code for every check in, you can make local changes in jenkins and discard them after the build is done, passing the deployed version to a next job in the pipeline. This way the artifacts are deployed to the artifact repo with the correct version, but the version in the pom file stays intact. So, in the pom file, you can specify the version as a SNAPSHOT, a version that will actually never be deployed to the artifact repository in this case:

{% highlight xml %}
<groupId>com.foocompany</groupId>
<artifactId>foobaz</artifactId>
<version>1.0-SNAPSHOT</version>
{% endhighlight %}

Then, before executing mvn deploy, you have to set the following script to be run in the jenkins job:

{% highlight bash %}
mvn versions:set -DnewVersion=1.0.${BUILD_NUMBER} -DgenerateBackupPoms=false
{% endhighlight %}

What happens when this is run is that the pom file gets changed locally. For example, if BUILD_NUMBER is 567, then:

{% highlight diff %}
<groupId>com.foocompany</groupId>
<artifactId>foobaz</artifactId>
-<version>1.0-SNAPSHOT</version>
+<version>1.0.567</version>
{% endhighlight %}

Now, instead of committing and pushing this, just run the maven deploy task, publishing the artifacts with the just created version.
<p>
Assuming that next jobs in the pipeline will not need access to the VCS, this should be enough. They should not need it and should fetch the deployed artifacts from the artifact repository instead. As the last step, just add the "Trigger parametrized build on other projects" post-build action to your jenkins job configuration, passing the newly created artifact version along as a parameter. For instance you for the example case presented above you could pass:

{% highlight properties %}
CI_BUILD_VERSION=1.0.${BUILD_NUMBER}
{% endhighlight %}

Following builds in the pipeline would fetch and install the artifacts with the version specified in the CI_BUILD_VERSION variable.
<p>
All done. Jenkins does not commit to upstream but each check in results in a new artifact version deployed to the artifact repository, tested and potentially released.
