+++
date = "2016-01-26T18:27:31+01:00"
draft = true
title = "sbt search maven plugin"

+++

# Sbt plugin to query search.maven.org

## tl;dr

[show me the code](https://github.com/blstream/sbt-search-maven-plugin)

## Why

Most development tasks can be done without leaving your favorite editor/ide nor console.
Writing Scala code is no different: sbt gives you a lot of power to run code and tests, to package and publish application.

There is one small thing though, forcing you to leave that environment and go to a browser - to find the exact name of dependency for your project.
Unless you have super memo powers you probably have to check the group id for akka, or latest version for any other package that you would like to include in your project.

To make it even simpler, I created *sbt-search-maven-plugin*. Just type `searchMaven something` and everything is clear. This prints the same results as `search.maven.org` without leaving sbt.

## Plugin development

In order to make this happen I had to get some knowledge on how to write an sbt plugin. There are three great sources you can check:

* [SBT AutoPlugins Tutorial](http://mukis.de/pages/sbt-autoplugins-tutorial/)
* [testing sbt plugins](http://eed3si9n.com/testing-sbt-plugins)
* [Plugins in sbt documentation](http://www.scala-sbt.org/0.13/docs/Plugins.html)

These are all great, but I wanted to create plugin that gets some user input from sbt shell, which is not covered in these references.

## Input plugin

Plugin code is similar as in above examples, but main "entry" point for your code is different, as we need InputKey.
There are some important places to look at:

### build.sbt

The most important part for developing a plugin is

{{< highlight scala >}}
sbtPlugin := true
{{</ highlight >}}

The whole [build.sbt](https://github.com/blstream/sbt-search-maven-plugin/blob/master/build.sbt) for this plugin.

### extends AutoPlugin

The entry point for your code should be placed in an object that extends `sbt.AutoPlugin`. This in turn should have another object defined: `autoImport` with definition of task.

{{< highlight scala >}}
object autoImport {
  lazy val searchMaven = InputKey[Unit]("searchMaven", "Search maven")
}
{{</ highlight >}}

Input key makes it possible to get input provided by user from sbt shell. To write some implementation we need one more thing:

{{< highlight scala >}}
override lazy val projectSettings = Seq(
  searchMaven := search(complete.DefaultParsers.spaceDelimited("<arg>").parsed, streams.value.log)
)
{{</ highlight >}}

`search` from snippet above is a regular Scala function in which we can finally implement our new feature.

`sbt.Keys._` contain a lot of useful things that we can use in our code, like `scalaVersion` defined in one's project or `streams.value.log`, which we use in our plugin to print results to user.

The whole file can be found [here](https://github.com/blstream/sbt-search-maven-plugin/blob/master/src/main/scala/com/blstream/sbtsearchmavenplugin/SbtSearchMavenPlugin.scala)

## Additional sbt settings

There are two things that should be mentioned here.

To make your plugin enabled by default you should add such override:
{{< highlight scala >}}
override def trigger = allRequirements
{{</ highlight >}}
This simply says that this plugin will be activated when all required plugins are present. Plugins that this plugin depends on could be defined by overriding def `requires` (it's empty by default)

This plugin doesn't interact with code in project - just searches for artifacts - so we don't want our code to be run multiple times when it's executed inside multimodule project.
To do this additional setting has to be specified:

{{< highlight scala >}}
aggregate in searchMaven := false
{{</ highlight >}}
This stops the task from running in submodules.

### for testing

If you use version without `SNAPSHOT` suffix and don't want to get warnings about deprecation when packaging your plugin, just add  `isSnapshot := true`  to your `build.sbt`.

## Testing sbt plugin

### unit tests

Nothing fancy here, just simple unit test for code that's not related to sbt.

### scripted

Scripted is default mechanism for testing sbt plugins by writing scripts for sbt. Search maven is not building anything,
so we can just prepare basic script to check whether `searchMaven` command is available and executed with success.

Special directory structure has to be used. Tests sit in `sbt-test` in `src`. Under that directory another two directories have to be created for test group and test itself. The full path looks like this:

{{< highlight scala >}}
<projectHome>/src/sbt-test/<testGroup>/<testName>
{{</ highlight >}}

In such path test should be described as:

* `plugins.sbt` inside of project dir that adds plugin to test project
* `test` file that contains script (test scenario)
* `build.sbt` file that describes test build and assertions for test. Assertions are written in form of sbt tasks.

Compare [usage-help-test](https://github.com/blstream/sbt-search-maven-plugin/tree/master/src/sbt-test/test-group/usage-help-test)

## Make it available for everyone

The best description can be found [here](http://www.scala-sbt.org/0.13/docs/Bintray-For-Plugins.html). Screenshots are a bit outdated, but most of the content is still valid.

[build.sbt](https://github.com/blstream/sbt-search-maven-plugin/blob/master/build.sbt) can stay simple.

## Contribution

You can pick a feature from future work section in [readme](https://github.com/blstream/sbt-search-maven-plugin),
implement issue with feature proposal (if any) or fix some bug. Pull requests are very welcome!

## Summary

This blog describes how to write simple sbt plugin, test it and publish it.
