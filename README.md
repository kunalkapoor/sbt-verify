# sbt-verify [![Linux Build Status](https://travis-ci.org/josephearl/sbt-verify.svg?branch=master)](https://travis-ci.org/josephearl/sbt-verify) [![Windows Build status](https://ci.appveyor.com/api/projects/status/8id30yqxiecak5qu?svg=true)](https://ci.appveyor.com/project/josephearl/sbt-verify)

An sbt 1.x plugin that verifies the integrity of downloaded dependencies, similar to [gradle-witness](https://github.com/WhisperSystems/gradle-witness).

sbt allows you to easily use remote dependencies without having to check them into your source tree by specifying `libraryDependencies`:

```scala
libraryDependencies += "com.google.code.gson" % "gson" % "2.6.2"
```

The published Maven-compatible artifact for a dependency consists of a POM file and a jar or other artifact, along with SHA1 and MD5 checksum hash values for those files, for example:

```
gson-2.6.2.jar
gson-2.6.2.jar.md5
gson-2.6.2.jar.sha1
gson-2.6.2.pom
gson-2.6.2.pom.md5
gson-2.6.2.pom.sha1
```

When sbt/Ivy/Coursier downloads the artifact it also retrieves the checksum files for each file and verifies that the downloaded file matches the SHA1/MD5 stored in the checksum file.
However this serves only to verify that the file was downloaded correctly - if someone is able to compromise the remote Maven repository they could easily alter the MD5 and SHA1 checksum values the repository advertises as well as the artifact itself.

The sbt-verify allows you to statically specify the SHA1 or MD5 checksum that a dependency should match through the `verifyDependencies` setting:

```scala
scalaVersion = "2.12.8"

libraryDependencies += "com.google.code.gson" % "gson" % "2.6.2"

verifyDependencies in verify ++= Seq(
  "org.scala-lang" % "scala-library" SHA1 "9aae4cb1802537d604e03688cab744ff47b31a7d",
  "com.google.code.gson" % "gson" SHA1 "17484370291d4a8191344ec4930a1c655b1d15e2"
)
```

Running the sbt task `verify` will check that all of the project's dependencies have the correct checksums.
If there's a mismatch, or a checksum is a missing for a file the build is failed, preventing an attacker undetectably modifying artifacts.

## Getting started

Unfortunately sbt-verify is not available as a published artifact since that would create a bootstrapping problem: nothing would verify the integrity of the downloaded sbt-verify, so an attacker could compromise that to always report a successful checksum match. Therefore to use sbt-verify it must be built from source.

Clone the sbt-verify repository, verify the source code, then build and publish it locally:

```scala
sbt publishLocal
```

Add sbt-verify as a plugin in your project's `project/plugins.sbt`:

```scala
addSbtPlugin("uk.co.josephearl" % "sbt-verify" % "0.4.0")
```

## Usage

### Enabling sbt-verify in your project

SBT verify must be explicitly enabled on the projects you wish to use it for.

In `project/plugins.sbt`:
```scala
addSbtPlugin("uk.co.josephearl" % "sbt-verify" % "0.4.0")
```

In `build.sbt`:
```scala
lazy val root = (project in file("."))
  .enablePlugins(VerifyPlugin)
```

### Verifying dependencies

To verify your project's dependencies first list all of your dependencies and specify the algorithm to be used to verify the hashes:

```scala
verifyDependencies in verify ++= Seq(
  "org.scala-lang" % "scala-library" SHA1 "9aae4cb1802537d604e03688cab744ff47b31a7d",
  "com.google.guava" % "guava" SHA1 "6ce200f6b23222af3d8abb6b6459e6c44f4bb0e9"
)

verifyAlgorithm in verify := HashAlgorithm.SHA1
```

Run the `verify` task to verify all downloaded dependencies have the correct hash.

By default the plugin will fail the build if it finds any unverified dependencies, or if any of the verifications (`verifyDependencies`) are unused.

To change the options, for instance to ignore Scala library files, use the `verifyOptions in verify` setting:

```scala
verifyOptions in verify := VerifyOptions(includeScala = false)
```

### Generating verification for dependencies

You can generate a `verify.sbt` file containing the `verifyDependencies` for all dependencies in your project by running the `verifyGenerate` task.

If you want to configure options for generating, set the `verifyOptions in verifyGenerate` in your `build.sbt`. This will generate a corresponding `verifyOptions in verify` in the generated `verify.sbt`.

The `verify.sbt` file is generated in the base directory. You can change this using the `verifyGenerateOutputFile` setting:

```scala
verifyGenerateOutputFile in verifyGenerate := crossTarget.value / "verify.sbt"
```

To change the hashing algorithm used when generating the file from the default (`SHA1`) use the `verifyAlgorithm` setting:

```scala
verifyAlgorithm in verifyGenerate := HashAlgorithm.MD5
```

## Settings

### `verifyOptions` in `verify`, `verifyGenerate`
* *Description:* Set the options to generate or verify dependencies.
* *Accepts:* `VerifyOptions`
* *Default:*
```
VerifyOptions(
    includeBin = true, 
    includeScala = true, 
    includeDependency = true, 
    excludedJars = Nil,
    warnOnUnverifiedFiles = false,
    warnOnUnusedVerifications = false
)
```

### `verifyAlgorithm` in `verify`, `verifyGenerate`
* *Description:* Hash algorithm to use when generating or verifying hashes for dependencies.
* *Tasks:* `verifyGenerate`
* *Accepts:* `HashAlgorithm.{SHA1, MD5}`
* *Default:* `HashAlgorithm.SHA1`

### `verifyDependencies` in `verify`
* *Description:* List of dependencies and hashes to use for `verify`.
* *Accepts:* `Seq(VerifyID)`
* *Default:* `Nil`

### `verifyGenerateOutputFile` in `verifyGenerate`
* *Description:* File to write verifications to when running `verifyGenerate`.
* *Tasks:* `verifyGenerate`
* *Accepts:* `File`
* *Default:* `baseDirectory(_ / "verify.sbt").value`


## Related projects

* [sbt/sbt-pgp](https://github.com/sbt/sbt-pgp) - Validate the PGP signatures of your dependencies
