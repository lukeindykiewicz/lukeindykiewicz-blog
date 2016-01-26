+++
date = "2016-01-26T18:27:31+01:00"
draft = true
title = "sbt search maven plugin"

+++

# Sbt plugin to query search.maven.org

## Why

Lots of development tasks can be done without leaving your favorite editor/ide nor console.
Writing Scala code is no difference, sbt gives a lot of power to run code and tests, to package and publish application. There is even plugin for git.

There is one small thing for which you have to leave that environment and go to a browser - to search exact name of dependency for your project.
Unless you have super memo powers you probably have to check what is the group id for akka or latest version for any other package that you would like to include in your project.

To make it even simpler sbt-search-maven-plugin was created. Just type `searchMaven <akka>` and everything is clear. This prints the same results as `search.maven.org`, but without leaving sbt.

## Plugin development

In order to make this happen I had to get some knowledge on how to write sbt plugin. There are three great sources you can check:

* [SBT AutoPlugins Tutorial](http://mukis.de/pages/sbt-autoplugins-tutorial/)
* [testing sbt plugins](http://eed3si9n.com/testing-sbt-plugins)
* [Plugins in sbt documentation](http://www.scala-sbt.org/0.13/docs/Plugins.html)

Theses are all great, but I wanted to create plugin that gets some user input from sbt shell, which is not covered in these references.

## Input plugin

Plugin is generally the same, but main "entry" point for your code is different.
There are some important places to look at:

### build.sbt

The most important part for plugin is

{{< highlight scala >}}
sbtPlugin := true
{{</ highlight >}}

The whole [build.sbt](https://github.com/blstream/sbt-search-maven-plugin/blob/master/build.sbt) for this plugin.

### extends AutoPlugin

The entry point for your code should be placed in an object that extends sbt.AutoPlugin. This in turn should have another object autoImport with definition of task to be added.

{{< highlight scala >}}
object autoImport {
  lazy val searchMaven = InputKey[Unit]("searchMaven", "Search maven")
}
{{</ highlight >}}

Input key makes it possible to get input provided by user from sbt shell. To write some implementation we need one more thing

{{< highlight scala >}}
override lazy val projectSettings = Seq(
  searchMaven := search(complete.DefaultParsers.spaceDelimited("<arg>").parsed, streams.value.log)
)
{{</ highlight >}}

`search` from snippet above is regular Scala function in which we can finally implement our new feature.

`sbt.Keys._` contain a lot of useful things that we can use in our code, like i.e. `scalaVersion` defined in ones project or `streams.value.log`, which we use in our plugin to print results to user

The while file can be found [here](https://github.com/blstream/sbt-search-maven-plugin/blob/master/src/main/scala/com/blstream/sbtsearchmavenplugin/SbtSearchMavenPlugin.scala)

## Additional sbt settings

There are two things that should be mentioned here.

To make your plugin enabled by default you should add such override:
{{< highlight scala >}}
override def trigger = allRequirements
{{</ highlight >}}
This simply says that this plugin will be activated when all required plugins are present. Plugins that this plugin depends on could be defined by overriding def `requires` (it's empty by default)

This plugin doesn't iteract with code in project, but searches for artifacts, so we don't want our code to be run multiple times when it's executed from multimodule project.
To do this additional setting has to be specified:

{{< highlight scala >}}
aggregate in searchMaven := false
{{</ highlight >}}
This says to not run defined task in submodules.

## Testing sbt plugin

### unit tests

Nothing fancy here, just simple unit test for code that's not connected to sbt.

### scripted

Scripted is default mechanism for testing sbt plugins.

continue here

## Make it available for everyone

## Contribution

## Summary
