TheseFoolishThings - SuperPOM
================================

[![Build Status](https://drone.io/bitbucket.org/tidalwave/thesefoolishthings-superpom-src/status.png)](https://drone.io/bitbucket.org/tidalwave/thesefoolishthings-superpom-src/latest)

A feature-rich SuperPOM for building Java projects. It features:

* explicit version configuration for a number of plugins;
* easy configurability by means of pre-defined properties to avoid cut & copy of plugin sections.

A number of profiles, easily activatable, are available for:

* Spring-AOP configuration;
* different kinds of Continous Integration tasks, including a full run of QA tools such as JaCoCo, FindBugs, PMD, etc...
* deploying WARs and locally running them with Tomcat or Jetty;
* creating a Mac OS X bundle for JavaFX applications;
* creating a Mac OS X bundle for NetBeans Platform applications;
* a customized release cycle, including all requirements for the Maven Central such as signing, with a 'transactional' behaviour 
  (all artifacts, both the DSCM and the Maven artifacts are prepared on the local disk, so they can be uploaded in a second moment);


Remember to customise it
------------------------

If you use it, please remember to change the ```description```,```url```, ```organization```, ```developers```, ```license```, etc... 
to override those related to the development of this POM.


Bootstrapping
-------------

In order to build the project, run from the command line:

```mvn -DskipTests```

The project can be opened and built by a recent version of the NetBeans, Eclipse or Idea IDEs.


Documentation
-------------

More information can be found on the [homepage](http://thesefoolishthings.kenai.com) of the project.


Where can I get the latest release?
-----------------------------------
You can download source and binaries from the [download page](${scm.repo.browse.url}).

Alternatively you can pull it from the central Maven repositories:

```xml
<dependency>
    <groupId>it.tidalwave.thesefoolishthings<groupId>
    <artifactId>superpom</artifactId>
    <version>-- version --</version>
</dependency>
```


Contributing
------------

We accept pull requests via BitBucket or GitHub.

There are some guidelines which will make applying pull requests easier for us:

* No tabs! Please use spaces for indentation.
* Respect the code style.
* Create minimal diffs - disable on save actions like reformat source code or organize imports. If you feel the source
  ode should be reformatted create a separate PR for this change.
* Provide TestNG tests for your changes and make sure your changes don't break any existing tests by running
```mvn clean test```.

If you plan to contribute on a regular basis, please consider filing a contributor license agreement. Contact us for
 more information


License
-------
Code is released under the [Apache Licence v2](https://www.apache.org/licenses/LICENSE-2.0.txt).


Additional Resources
--------------------

* [Tidalwave Homepage](http://tidalwave.it)
* [Project Issue Tracker (Jira)](http://services.tidalwave.it/jira/browse/SUP)
* [Project Continuous Integration (hudson)](http://ci.tidalwave.it/ci/view/TheseFoolishThings_SuperPOM)
