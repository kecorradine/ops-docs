# Operational Documentation

This repository contains documentation for deploying the release management system and operational documentation.

## Services quick links

* [Jenkins](http://build.ihtsdotools.org) ([Config docs](Jenkins.md))
* [Sonatype Nexus](https://maven.ihtsdotools.org) ([Config docs](Nexus.md))
* [SonarQube](https://sonar.ihtsdotools.org) ([Config docs](SonarQube.md))
* [JIRA](https://jira.ihtsdotools.org)

## Application build requirements

## Overview

The build and release process is broken into a number of distinct stages.
![Release stages](release_stages.png)

### Building

The build process is run by [Jenkins](http://build.ihtsdotools.org) and Maven, with Jenkins triggering the build and controlling progress through the stages, and Maven doing the actual build.

Source is stored at [GitHub](http://github.com/IHTSDO). Most repositories are public, with a few exceptions. Jenkins needs write access to the repositories, especially for the [release process](#releaseprocess).

As was as the usual collection of jars, Debian native packages are produced using the [jdeb Maven plugin](https://github.com/tcurdt/jdeb).

During the build phase, the code is analysed by [SonarQube](https://sonar.ihtsdotools.org) producing a code quality report.

The resulting artifacts are deployed to [Sonatype Nexus](https://maven.ihtsdo.org). Nexus has an [apt plugin](https://github.com/inventage/nexus-apt-plugin), which generated apt metadata from deb packages uploaded to repositories.

![Build sequence](build_sequence.png)

Content
-------

* The [build process](build.md)
* The [release process](release.md)

