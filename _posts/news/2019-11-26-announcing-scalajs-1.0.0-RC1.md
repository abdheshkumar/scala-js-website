---
layout: post
title: Announcing Scala.js 1.0.0-RC1
category: news
tags: [releases]
permalink: /news/2019/11/26/announcing-scalajs-1.0.0-RC1/
---


We are thrilled to announce the release of Scala.js 1.0.0-RC1!

This release candidate is intended for testing purposes by as many users as possible, and as a synchronization point with library authors so that they can start upgrading in preparation for the final release.
If no critical issue is found until the end of January 2020, this RC will become the final release.

**We encourage all users who are able to do so to test their projects with this RC, and report any issue as soon as possible.**

As the change in "major" version number witnesses, this release is *not* binary compatible with 0.6.x, nor with the previous milestones of the 1.x series.
Libraries need to be recompiled and republished using this RC to be compatible.
Moreover, this release is not entirely source compatible with 0.6.x either.

These release notes contain cumulative changes with respect to 0.6.31.
Compared to 1.0.0-M8, the following changes are noteworthy:

* Drop support for sbt 0.13.x
* Drop support for Scala 2.11.{0-11} (2.11.12 is supported)
* `x eq y` now more closely matches the JVM behavior: `+0.0 ne -0.0` and `NaN eq NaN`
* Small tweaks in the JSEnv API
* The Scala.js linker is now loaded by reflection into the sbt plugin, which solves issues with binary incompatible transitive dependencies such as Guava.

We would also like to remind readers of the following important change that happened in 1.0.0-M5 and 1.0.0-M7, respectively:

* Drop compatibility with sbt-crossproject v0.4.x and earlier (v0.5.0 or later is required)
* With the default module kind `NoModule`, top-level exports are now exposed to JavaScript as top-level `var`s, rather than assigned as properties of the global object

<!--more-->

Please report any issues [on GitHub](https://github.com/scala-js/scala-js/issues).

## Preparations before upgrading from 0.6.x

Before upgrading to 1.0.0-RC1, **we strongly recommend that you upgrade to Scala.js 0.6.31 or later**, and address all deprecation warnings.
Since Scala.js 1.0.0-RC1 removes support for all the deprecated features in 0.6.x, it is easier to see the deprecation messages guiding you to the proper replacements.

In particular, make sure that you explicitly use [`sbt-crossproject`](https://github.com/portable-scala/sbt-crossproject#migration-from-scalajs-default-crossproject) instead of the default `crossProject` implementation.
The old `crossProject` is deprecated in 0.6.31, but it is easy to overlook deprecations in the build itself.

Additionally to the explicitly deprecated things, make sure to use `scalaJSLinkerConfig` instead of the following sbt settings:

* `scalaJSSemantics`
* `scalaJSModuleKind`
* `scalaJSOutputMode`
* `emitSourceMaps`
* `relativeSourceMaps`
* `scalaJSOptimizerOptions`

Finally, if you are still using sbt 0.13.x, you will have to upgrade to sbt 1.2.1 or later (as of this writing, the latest release is 1.3.4).

## Upgrade to 1.0.0-RC1 from 0.6.31 or later

As a first approximation, all you need to do is to update the version number in `project/plugins.sbt`:

{% highlight scala %}
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "1.0.0-RC1")
{% endhighlight %}

In addition, if you use some of the components that have been moved to separate repositories, you will need to add some more dependencies in `project/plugins.sbt`:

If you use `scalajs-stubs`:

* Change its version number to `"1.0.0-RC1"` (irrespective of the version of Scala.js itself, and hence of the `scalaJSVersion` constant)

If you use `jsDependencies` (or rely on the `jsDependencies` of your transitive dependencies):

* Add `addSbtPlugin("org.scala-js" % "sbt-jsdependencies" % "1.0.0-RC1")` in `project/plugins.sbt`
* Add `.enablePlugins(JSDependenciesPlugin)` to Scala.js `project`s
* Add `.jsConfigure(_.enablePlugins(JSDependenciesPlugin))` to `crossProject`s

If you use the Node.js with jsdom environment:

* Add `libraryDependencies += "org.scala-js" %% "scalajs-env-jsdom-nodejs" % "1.0.0-RC1"` in `project/plugins.sbt`

If you use the PhantomJS environment:

* Add `addSbtPlugin("org.scala-js" % "sbt-scalajs-env-phantomjs" % "1.0.0-RC1")` in `project/plugins.sbt`

Finally, if your build has

{% highlight scala %}
scalacOptions += "-P:scalajs:sjsDefinedByDefault"
{% endhighlight %}

you will need to remove it (Scala.js 1.x always behaves as if `sjsDefinedByDefault` were present).

This should get your build up to speed to Scala.js 1.0.0-RC1.
From there, you should be able to test whether things go smoothly, or whether you are affected by the breaking changes detailed below.

## Breaking changes

This section discusses the backward incompatible changes, which might affect your project.

### sbt 0.13.x is not supported anymore

You will not be able to use Scala.js 1.0.0-RC1 with sbt 0.13.x.
You will have to upgrade to sbt 1.2.1 or later.
As of this writing, the latest version is 1.3.4.

### Scala 2.10.x is not supported anymore, nor building on JDK 6 and 7

The title says it all: you cannot use Scala.js with

{% highlight scala %}
scalaVersion := "2.10.x" // for any x
{% endhighlight %}

anymore.
Note that you can still use the sbt plugin with sbt 0.13.17+, even though it runs itself on 2.10.x.
Only the Scala.js code itself (not the build) cannot use Scala 2.10.x.

In addition, building Scala.js code on top of JDK 6 or 7 is not supported anymore either.

Finally, a severe regression in Scala 2.12.0 upstream, affecting `js.UndefOr`, forced us to drop support for Scala 2.12.0 (see [#3024](https://github.com/scala-js/scala-js/issues/3024)).
Scala 2.12.1+ is supported.

### Scala 2.11.{0-11} are not supported anymore

Only 2.11.12 is still supported in the 2.11.x series.

### Access to the global scope instead of the global object

This is the only major breaking change at the language level.
In Scala.js 1.x, `js.Dynamic.global` and `@JSGlobalScope` objects refer to the global *scope* of JavaScript, rather than the global *object*.
Concretely, this has three consequences, which we outline below.
Further information can be found in [the documentation about the global scope in Scala.js]({{ BASE_PATH }}/doc/interoperability/global-scope.html).

#### Members can only be accessed with a statically known name which is a valid JavaScript identifier

For example, the following is valid:

{% highlight scala %}
println(js.Dynamic.global.Math)
{% endhighlight %}

but the following variant, where the name `Math` is only known at run-time, is not valid anymore:

{% highlight scala %}
val mathName = "Math"
println(js.Dynamic.global.selectDynamic(mathName))
{% endhighlight %}

The latter will cause a compile error.
This is because it is not possible to perform dynamic lookups in the global scope.
Similarly, accessing a member whose name is statically known but not a valid JavaScript identifier is also prohibited:

{% highlight scala %}
println(js.Dynamic.global.`not-a-valid-JS-identifier`)
{% endhighlight %}

#### Global scope objects cannot be stored in a separate `val`

For example, the following is invalid and will cause a compile error:

{% highlight scala %}
val g = js.Dynamic.global
{% endhighlight %}

as well as:

{% highlight scala %}
def foo(x: Any): Unit = println(x)
foo(js.Dynamic.global)
{% endhighlight %}

This follows from the previous rule.
If the above two snippets were allowed, we could not check that we only access members with statically known names.

The first snippet can be advantageously replaced by a renaming import:

{% highlight scala %}
import js.Dynamic.{global => g}
{% endhighlight %}

#### Accessing a member that is not declared causes a `ReferenceError` to be thrown

This is a *run-time* behavior change, and in our experience the larger source of breakages in actual code.

Previously, reading a non-existent member of the global object, such as

{% highlight scala %}
println(js.Dynamic.global.globalVarThatDoesNotExist)
{% endhighlight %}

would evaluate to `undefined`.
In Scala.js 1.x, this throws a `ReferenceError`.
Similarly, writing to a non-existent member, such as

{% highlight scala %}
js.Dynamic.global.globalVarThatDoesNotExist = 42
{% endhighlight %}

would previously *create* said global variable.
In Scala.js 1.x, it also throws a `ReferenceError`.

A typical use case of the previous behavior was to *test* whether a global variable was defined or not, e.g.,

{% highlight scala %}
if (js.isUndefined(js.Dynamic.global.Promise)) {
  // Promises are not supported
} else {
  // Promises are supported
}
{% endhighlight %}

This idiom is broken in Scala.js 1.x, and needs to be replaced by an explicit use of `js.typeOf`:

{% highlight scala %}
if (js.typeOf(js.Dynamic.global.Promise) != "undefined")
{% endhighlight %}

The `js.typeOf` "method" is magical when its argument is a member of a global scope object.

### Scala.js emits ECMAScript 2015 code by default

Now that ES 2015 has been supported by major JS engines for a while, it was time to emit ES 2015 code by default.
The ES 2015 output has several advantages over the older ES 5.1 strict mode output:

* `Throwable`s, by virtue of properly extending JavaScript's `Error`, have an `[[ErrorData]]` internal slot, and therefore receive proper debugging info in JS engines, allowing better display of stack traces and error messages in interactive debuggers.
* Static fields and methods in JS classes are properly inherited. See https://github.com/scala-js/scala-js/issues/2771.
* The generated code is shorter.

To revert to emitting ES 5.1 strict mode code, use the following sbt setting:

{% highlight scala %}
scalaJSLinkerConfig in ThisBuild ~= { _.withESFeatures(_.withUseECMAScript2015(false)) }
{% endhighlight %}

### With the default module kind `NoModule`, top-level exports are exported as top-level `var`s

In Scala.js 0.6.x, top-level exports such as

{% highlight scala %}
@JSExportTopLevel("Foo")
object Bar
{% endhighlight %}

were exported to JavaScript by being assigned to properties of the global object, for example as if by:

{% highlight javascript %}
window.Foo = <the object Bar>; // or global.Foo, etc.
{% endhighlight %}

In Scala.js 1.x, they are exported as top-level JavaScript `var`s instead, as if by:

{% highlight javascript %}
var Foo = <the object Bar>;
{% endhighlight %}

This ensures that code generated by Scala.js is completely generic with respect to which variables hold the global object, and hence works in any compliant JavaScript environment.

However, it also means that the generated .js file must be interpreted as a proper *script* for the variables to be visible by other scripts.
This may have compatibility consequences.

### `js.UndefOr[A]` is now an alias for `A | Unit`

Instead of defining `js.UndefOr[+A]` as its own type, it is now a simple type alias for `A | Unit`:

{% highlight scala %}
type UndefOr[+A] = A | Unit
{% endhighlight %}

The `Option`-like API is of course preserved.

We do not expect this to cause any significant issue, but it may impact type inference in subtle ways that can cause compile errors for previously valid code.
You may have to adjust some uses of `js.UndefOr` due to these changes.

### `x eq y` now matches more closely matches the JVM behavior

In Scala.js 0.6.x, `x eq y` always corresponds to JavaScript's `x === y`.
This is almost always correct, but causes issues when comparing `+0.0` with `-0.0` or `NaN` with itself.

In Scala.js 1.x, `x eq y` has been adapted to more closely match the JVM behavior, resulting in more portable code.
It is equivalent to the old behavior except in the following cases:

* `+0.0 eq -0.0` is now `false`
* `NaN eq NaN` is now `true`

The new behavior corresponds to JavaScript's [`Object.is(x, y)`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is) function.

### `testHtml` replaces both `testHtmlFastOpt` and `testHtmlFullOpt`

The separation of `testHtmlFastOpt` and `testHtmlFullOpt`, which were independent of the value of `scalaJSStage`, caused significant unfixable issues in 0.6.x.
In Scala.js 1.x, both are replaced by a single task, `testHtml`.
It is equivalent to the old `testHtmlFastOpt` if the value of `scalaJSStage` is `FastOptStage` (the default), and to `testHtmlFullOpt` if it is `FullOptStage`.
This makes it more consistent with other tasks such as `run` and `test`.

### Java system properties are not loaded from `__ScalaJSEnv` nor sbt's `javaOptions` anymore

In 0.6.x, the sbt plugin automatically extracted `-D` options in `javaOptions`, and transferred them to the Java system properties API inside Scala.js, using a generated .js file filling in the magic variable `__ScalaJSEnv`.
Scala.js 1.x does not support `__ScalaJSEnv` anymore, therefore `-D` options in `javaOptions` are now ignored.

Use your own mechanism to transfer data from the build to your Scala.js code, for example source code generation.

### A unique, simple sbt setting to control what JS files are `run` or `test`ed: `jsEnvInput`

By default, only the .js file generated by `fastOptJS` or `fullOptJS` is given to the selected `jsEnv` to be `run` or `test`ed.
In 0.6.x, there were several non-obvious task keys to modify this behavior: `resolvedJSDependencies`, `loadedJSEnv` and related.
Moreover, the `JSEnv`s decided on their own, based on unclear heuristics, whether to treat the files as modules or not, and as what kind of module.

Scala.js 1.x consolidates all of that into one simple task key `jsEnvInput` of type `Seq[org.scalajs.jsenv.Input]`, where an `Input` is an ADT with the following possible alternatives:

* `Input.Scripts(script)`: a JavaScript file to load as a script
* `Input.ESModule(module)`: a JavaScript file to load as an ECMAScript module (some JS envs may require a specific extension such as `.mjs` for this to work, due to limitations of the underlying engines)
* `Input.CommonJSModule(module)`: a JavaScript file to load as a CommonJS module

Not all `JSEnv`s support all kinds of `Input`s.

The default value of `jsEnvInput` is a single `Input` whose type is derived from the `ModuleKind`, and which contains the output of `fastOptJS` or `fullOptJS`.

### `scalajs-javalib-ex` was removed

The artifact `scalajs-javalib-ex` is removed in 1.x.
It only contained a partial implementation of `java.util.ZipInputStream`.
If you were using it, we recommend that you integrate a copy of [its source code from Scala.js 0.6.x](https://github.com/scala-js/scala-js/tree/0.6.x/javalib-ex/src/main/scala/java/util/zip) into your project.

### `js.use(x).as[T]` was removed

The use cases for `js.use(x).as[T]` have been dramatically reduced by non-native JS classes (previously known as Scala.js-defined JS classes).
This API seems virtually unused on the visible Web.
Moreover, it was the only macro in the Scala.js standard library.

We have therefore removed it from the standard library, and it is not provided anymore.
On demand, we can republish it as a separate library, if you need it.

### The Tools API has been split into 3 artifacts and its packages reorganized

This only concerns consumers of the Tools API, i.e., tools that build on top of the Scala.js linker, such as ScalaFiddle.
In Scala.js 0.6.x, all the tools were in one artifact `scalajs-tools`.
This artifact has been split in two in Scala.js 1.x:

* `scalajs-logging`: tiny logging API
* `scalajs-linker-interface`: the extra stable interface for the linker API
* `scalajs-linker`: the implementation of the linker API

The `scalajs-linker` artifact can be loaded via reflection, while developing against the stable APIs in `scalajs-linker-interface`.

In addition, the packages have been reorganized as follows:

* `org.scalajs.core.ir`            -> `org.scalajs.ir`
* `org.scalajs.core.tools.io`      -> gone (replaced by standard `java.nio.file.Path`-based APIs, and some abstractions in `org.scalajs.linker`)
* `org.scalajs.core.tools.logging` -> `org.scalajs.logging`
* `org.scalajs.core.tools.linker`  -> `org.scalajs.linker` and `org.scalajs.linker.interface`

Additionally, the linker API has been refactored to be fully asynchronous in nature.

## Enhancements

There are very few enhancements in Scala.js 1.0.0-RC1.
Scala.js 1.0.0 is focused on simplifying Scala.js, not on adding new features.
Nevertheless, here are a few enhancements.

### Scala.js can access `require` and other magical "global" variables of special JS environments

The changes from global *object* to global *scope* mean that magical "global" variables provided by some JavaScript environments, such as `require` in Node.js, are now visible to Scala.js.
For example, it is possible to dynamically call `require` as follows in Scala.js 1.x:

{% highlight scala %}
val pathToSomeAsset = "assets/logo.png"
val someAsset = js.Dynamic.global.require(pathToSomeAsset)
{% endhighlight %}

We still recommend to use `@JSImport` and `CommonJSModule` for statically known imports.

### Declaring inner classes in native JS classes

Some JavaScript APIs define classes inside objects, as in the following example:

{% highlight javascript %}
class OuterClass {
  constructor(x) {
    this.InnerClass = class InnerClass {
      someMethod() {
        return x;
      }
    }
  }
}
{% endhighlight %}

allowing use sites to instantiate them as

{% highlight javascript %}
const outerObject = new OuterClass(42);
const innerObject = new outerObject.InnerClass();
console.log(innerObject.someMethod()); // prints 42
{% endhighlight %}

In Scala.js 0.6.x, it is very awkward to define a facade type for `OuterClass`, as illustrated [in issue #2398](https://github.com/scala-js/scala-js/issues/2398).
Scala.js 1.x now allows to declare them very easily as inner JS classes:

{% highlight scala %}
@js.native
@JSGlobal
class OuterClass(x: Int) extends js.Object {
  @js.native
  class InnerClass extends js.Object {
    def someMethod(): Int = js.native
  }
}
{% endhighlight %}

which in turns allows for the following call site:

{% highlight scala %}
val outerObject = new OuterClass(42);
val innerObject = new outerObject.InnerClass();
console.log(innerObject.someMethod()); // prints 42
{% endhighlight %}

### Nested non-native JS classes expose sane constructors to JavaScript

It is now possible to declare non-native JS classes inside outer `class`es or inside `def`s, and use their `js.constructorOf` in a meaningful way.
For example, one can define a method that creates a new JavaScript class every time it is invoked:

{% highlight scala %}
def makeGreeter(greetingFormat: String): js.Dynamic = {
  class Greeter extends js.Object {
    def greet(name: String): String =
      println(greetingFormat.format(name))
  }
  js.constructorOf[Greeter]
}
{% endhighlight %}

Assuming there is some native JavaScript function like

{% highlight javascript %}
function greetPeople(greeterClass) {
  const greeter = new greeterClass();
  greeter.greet("Jane");
  greeter.greet("John");
}
{% endhighlight %}

one could call it from Scala.js as:

{% highlight scala %}
val englishGreeterClass = makeGreeter("Hello, %s!")
greetPeople(englishGreeterClass)
val frenchGreeterClass = makeGreeter("Bonjour, %s!")
greetPeople(frenchGreeterClass)
val japaneseGreeterClass = makeGreeter("%sさん、こんにちは。")
greetPeople(japaneseGreeterClass)
{% endhighlight %}

resulting in the following output:

{% highlight text %}
Hello, Jane!
Hello, John!
Bonjour, Jane!
Bonjour, John!
Janeさん、こんにちは。
Johnさん、こんにちは。
{% endhighlight %}

In Scala.js 0.6.x, the above code would compile but produce incoherent results at run-time, because `js.constructorOf` was meaningless for nested classes.

## Bugfixes

Amongst others, the following bugs have been fixed since 0.6.31:

* [#2800](https://github.com/scala-js/scala-js/issues/2800) Global `let`s, `const`s and `class`es cannot be accessed by Scala.js
* [#2382](https://github.com/scala-js/scala-js/issues/2382) Name clash for `$outer` pointers of two different nesting levels (fixed for Scala 2.10 and 2.11; 2.12 did not suffer from the bug in 0.6.x)
* [#3085](https://github.com/scala-js/scala-js/issues/3085) Linking error after the optimizer for `someInt.toDouble.compareTo(double)`

See the full list of issues [fixed in 1.0.0-M1](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M1+is%3Aclosed), [in 1.0.0-M2](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M2+is%3Aclosed), [in 1.0.0-M3](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M3+is%3Aclosed), [in 1.0.0-M4](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M4+is%3Aclosed), [in 1.0.0-M5](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M5+is%3Aclosed), [in 1.0.0-M6](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M6+is%3Aclosed), [in 1.0.0-M7](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M7+is%3Aclosed), [in 1.0.0-M8](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-M8+is%3Aclosed) and [in 1.0.0-RC1](https://github.com/scala-js/scala-js/issues?q=is%3Aissue+milestone%3Av1.0.0-RC1+is%3Aclosed) on GitHub.

## Cross-building for Scala.js 0.6.x and 1.x

If you want to cross-compile your libraries for Scala.js 0.6.x and 1.x (which you definitely should), here are a couple tips.

### Dynamically load a custom version of Scala.js

Since the version of Scala.js is not decided by an sbt setting in `build.sbt`, but by the version of the sbt plugin in `project/plugins.sbt`, standard cross-building setups based on `++` cannot be applied.
We recommend that you load the version of Scala.js from an environment variable.
For example, you can do this in your `project/plugins.sbt` file:

{% highlight scala %}
val scalaJSVersion =
  Option(System.getenv("SCALAJS_VERSION")).getOrElse("0.6.31")

addSbtPlugin("org.scala-js" % "sbt-scalajs" % scalaJSVersion)
{% endhighlight %}

You can then launch

{% highlight bash %}
$ SCALAJS_VERSION=1.0.0-RC1 sbt
{% endhighlight %}

from your command line to start up your build with Scala.js 1.0.0-RC1.

### Extra dependencies for JS environments

You can further build on the above `val scalaJSVersion` to dynamically add dependencies on `scalajs-env-phantomjs` and/or `scalajs-env-jsdom-nodejs` if you use them:

{% highlight scala %}
// For Node.js with jsdom
libraryDependencies ++= {
  if (scalaJSVersion.startsWith("0.6.")) Nil
  else Seq("org.scala-js" %% "scalajs-env-jsdom-nodejs" % "1.0.0-RC1")
}

// For PhantomJS
{
  if (scalaJSVersion.startsWith("0.6.")) Nil
  else Seq(addSbtPlugin("org.scala-js" % "sbt-scalajs-env-phantomjs" % "1.0.0-RC1"))
}
{% endhighlight %}

In both cases, you can then use the source-compatible API in `build.sbt` to select your JS environment of choice.

### Extra dependencies for `jsDependencies`

Similarly, you can conditionally depend on `jsDependencies` as follows:

{% highlight scala %}
// For jsDependencies
{
  if (scalaJSVersion.startsWith("0.6.")) Nil
  else Seq(addSbtPlugin("org.scala-js" % "sbt-jsdependencies" % "1.0.0-RC1"))
}
{% endhighlight %}

In that case, you should unconditionally keep the

{% highlight scala %}
enablePlugins(JSDependenciesPlugin)
{% endhighlight %}

on the relevant projects.
Scala.js 0.6.20 and later define a no-op `JSDependenciesPlugin` to allow for this scenario.

### Conditional application of `-P:scalajs:sjsDefinedByDefault`

In Scala.js 1.x, the flag `-P:scalajs:sjsDefinedByDefault` has been removed.
However, if you have non-native JS types in your codebase, you need this flag in Scala.js 0.6.x.

Add the following setting to your `build.sbt` to conditionally enable that flag:

{% highlight scala %}
scalacOptions ++= {
  if (scalaJSVersion.startsWith("0.6.")) Seq("-P:scalajs:sjsDefinedByDefault")
  else Nil
}
{% endhighlight %}
