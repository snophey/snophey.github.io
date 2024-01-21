---
layout: post
title:  "Compiling a program that uses native libraries with GraalVM"
date:   2024-01-21 10:50:03 +0100
categories: graalvm serverless java quarkus gcp 
---

There are numerous articles on the web that present the advantages of compiling your Java application to a native binary using GraalVM.
Many benchmarks out there show, that the startup time of a native binary is [much faster](https://shinesolutions.com/2021/08/30/improving-cold-start-times-of-java-aws-lambda-functions-using-graalvm-and-native-images/) than that of a JVM based application.
This is especially important for serverless applications, where the startup time of the application is directly related to the latency of the first request.
However, a downside of using GraalVM is that it can sometimes be tricky to compile your application to a native binary and have it work correctly.
This is due to all the optimizations that GraalVM does under the hood, which can sometimes lead to unexpected behavior.
A lot of these issues are discussed and documented in this [Quarkus tutorial](https://quarkus.io/guides/writing-native-applications-tips).

In this post, we will look at a case that is not covered by the aforementioned article, namely the case where your Java application relies
on some native libraries that are called via JNI.

## The problem

I recently had to do some scientific computing, and for this I needed the excellent [OR-Tools](https://developers.google.com/optimization) library from Google.
This is a C++ library, but it also has a Java wrapper, and being most comfortable with Java, I decided to use it.
The idea of the task was to have an API that would receive some input, do some optimization with OR-Tools and return the result.
Because optimization can be a computationally intensive task, it is best run on a machine with a lot of CPU power.
Modern solvers are able to use multiple CPU cores to parallelize parts of the optimization process, so to a certain level,
the API can benefit from having more CPU cores available to it.

The problem with this approach is that it is prohibitively expensive to have a virtual machine with a beefy multi-core CPU running all the time.
For this reason, I decided to deploy the API to Google Cloud Run, which is a Google's serverless offering.
Upon a request, Cloud Run will spin up a container running my API, and assign a certain amount of CPU cores to it (4 cores in my case).

Running the API in this fashion is very cost effective (especially since Google Cloud, like all major cloud providers, has a generous amount of
free CPU and GB-seconds available),
but it also means that we will incur the dreaded cold start times whenever a new container is spun up for the API.
Java is known to have a relatively slow startup time, so it was clear to me that eventually I would have to compile my API to a native binary
with GraalVM to get the best performance.
I have opted to use [Quarkus](https://quarkus.io/) as my framework of choice, since it has excellent support and
documentation for GraalVM and native compilation.
However, as I later found out, there are some caveats when compiling an application that uses JNI to call native libraries.

## Stumbling towards a solution

On the surface, compiling a Quarkus application with GraalVM is very straightforward. We install a GraalVM distribution with sdkman,
point our `GRAALVM_HOME` environment variable to it, and then trigger the actual build process:

```bash
sdk install java 21.0.2-graalce
# make sure to point GRAALVM_HOME to the place where sdkman installed GraalVM
export GRAALVM_HOME=$HOME/.sdkman/candidates/java/21.0.2-graalce
./mvnw package -Dnative
```

### Missing logging adapters

The first hiccup that I encountered was the build failing with the following error:

```
Error: Discovered unresolved type during parsing: io.grpc.netty.shaded.io.netty.util.internal.logging.Log4J2Logger. This error is reported at image build time because class io.grpc.netty.shaded.io.netty.util.internal.logging.Log4J2LoggerFactory is registered for linking at image build time by command line and command line.
```

Luckily, this is well documented issue, and the quarkus documentation has a [section](https://quarkus.io/guides/logging#logging-apis)
on addressing it. We simply need to include the relevant logging library adapter in our dependencies.
In my case, adding the log4j and log4j2 adapters to my pom.xml solved the issue:

```xml
<dependency>
    <groupId>org.jboss.logmanager</groupId>
    <artifactId>log4j-jboss-logmanager</artifactId>
</dependency>
<dependency>
    <groupId>org.jboss.logmanager</groupId>
    <artifactId>log4j2-jboss-logmanager</artifactId>
</dependency>
```

### Initializing classes on runtime

Upon adding the logging adapters, the build was able to progress further, but it eventually failed during the "analysis" phase with the following error:

```
...
Caused by: org.graalvm.compiler.java.BytecodeParser$BytecodeParserError: org.graalvm.compiler.debug.GraalError: com.oracle.svm.core.util.UserError$UserException: Class initialization of com.google.ortools.linearsolver.MPSolver$ResultStatus failed. Use the option 

    '--initialize-at-run-time=com.google.ortools.linearsolver.MPSolver$ResultStatus'

 to explicitly request initialization of this class at run time.
...
```

Whenever possible, GraalVM will try to initialize classes at build time, so that when the application is started, this step is already done.
It is part of what makes the compiled binary so fast to start up. However, in this case, GraalVM reports that it is unable to initialize the
mentioned class at build time. Rather helpfully, it also suggests a solution to this problem, namely to point the `--initialize-at-run-time` option to the class that failed to initialize, so we do just that:

```
quarkus.native.additional-build-args=--initialize-at-run-time=com.google.ortools.linearsolver.MPSolver$ResultStatus
```

At this point, the analysis phase of the build succeeds and we are presented with a nice summary.
Notice how GraalVM reports that it detected some types, fields and methods that are registered for JNI access.
This is already encouraging, because we do use JNI to call the OR-Tools library.

```
[2/8] Performing analysis...  [***********]                                                             (96.5s @ 2.04GB)
   21,193 reachable types   (87.4% of   24,237 total)
   33,679 reachable fields  (61.4% of   54,813 total)
  128,185 reachable methods (62.7% of  204,581 total)
    6,729 types, 2,253 fields, and 28,681 methods registered for reflection
       81 types,    88 fields, and    62 methods registered for JNI access
        4 native libraries: dl, pthread, rt, z
```

Once the build has finished, we are ready to stumble upon the next issue.

### Missing native libraries

The build is finished, we execute our freshly built application, send a request to it, and... the application crashes with a `NullPointerException` just as it is about to call our native library:

```
java.lang.NullPointerException: Resource ortools-linux-x86-64/ was not found in ClassLoader jdk.internal.loader.ClassLoaders$AppClassLoader@c818063
```

We remember that the Quarkus documentation states that [resources are not automatically included in the native binary](https://quarkus.io/guides/writing-native-applications-tips#including-resources),
and the native libraries that we depend on are also resources, so it is unsurprising that we get this error.
Luckily, this is a problem we can easily solve by adding a single line to our `application.properties` file:

```
quarkus.native.resources.includes=ortools-linux-x86-64/**
```

We rebuild our application, and this time it starts up successfully and is able to start solving our optimization problems.
We are confident that the native C++ code is being called, which implies that the libraries native libraries are being loaded correctly,
but we are still not out of the woods yet.
It is time for the final boss...

### Some class defintions are missing

Our application now runs up to a certain point, but then a new error is thrown:

```
java.lang.NoClassDefFoundError: com/google/ortools/linearsolver/MPVariable
        at org.graalvm.nativeimage.builder/com.oracle.svm.core.jni.functions.JNIFunctions.FindClass(JNIFunctions.java:359)
```

At this point, we are really scratching our heads. Our application code clearly makes use of this `MPVariable` class, so why is it not found?
The Quarkus documentation is of no help here, so we have nothing left to do but to try and understand what GraalVM does under the hood
in order to get an idea of where the problem might be.

Looking into the GraalVM documentation, we find a key bit on information in a [seemingly unrelated section](https://www.graalvm.org/latest/reference-manual/native-image/dynamic-features/JNI/#reflection-metadata) on reflection metadata

> The `native-image` builder must know beforehand which items will be looked up in case they might not be reachable otherwise and therefore would not be included in a native image. Moreover, native-image must generate call wrapper code ahead-of-time for any method that can be called via JNI. Therefore, specifying a concise list of items that need to be accessible via JNI guarantees their availability and allows for a smaller footprint.

So to guarantee that a class is available at runtime, instead of relying on GraalVM to detect it during the analysis phase, we need to explicitly specify it. We could do this by writing a `jni-config.json` file, where we include the classes and methods that we will require
at runtime, but this is a tedious and error prone process. Luckily, there is a better way: we can use GraalVM's [tracing agent](https://www.graalvm.org/latest/reference-manual/native-image/metadata/AutomaticMetadataCollection/#tracing-agent)
to automatically discover the classes that are required at runtime and have it generate the `jni-config.json` file for us.

To this end, we first compile our application to a regular boring JVM jar file, and then we run it with the tracing agent enabled:

```bash
$GRAALVM_HOME/bin/java -agentlib:native-image-agent=config-output-dir=./src/main/resources -jar target/quarkus-app/quarkus-run.jar
```

Now we issue a request to our API that will trigger the native library call, and then we stop the application. If we look in the `src/main/resources` directory, we will see that a `jni-config.json` file has been generated for us. This file is the result of the tracing agent
analyzing the application at runtime and discovering which classes and methods are required for the application to run correctly.
In it, we find a mention of the precise class we were missing earlier:

```json
[
{
  "name":"[Lcom.google.ortools.linearsolver.MPVariable;"
},
{
  "name":"com.google.ortools.linearsolver.MPVariable",
  "methods":[{"name":"<init>","parameterTypes":["long","boolean"] }]
},
...
```

With this metadata file in hand, all that is left to do is to point GraalVM to it during the build process, so we update
the `quarkus.native.additional-build-args` property in our `application.properties` file:

```
quarkus.native.additional-build-args=--initialize-at-run-time=com.google.ortools.linearsolver.MPSolver$ResultStatus,-H:JNIConfigurationFiles=jni-config.json
```

We rebuild our application, and this time we notice that the analysis phase has registered more types, fields and methods
for JNI access when compared to our previous attempt:

```
[2/8] Performing analysis...  [***********]                                                             (98.8s @ 2.49GB)
   21,199 reachable types   (87.4% of   24,244 total)
   33,698 reachable fields  (61.4% of   54,839 total)
  128,205 reachable methods (62.6% of  204,732 total)
    6,730 types, 2,253 fields, and 28,681 methods registered for reflection
       89 types,    96 fields, and    66 methods registered for JNI access
        4 native libraries: dl, pthread, rt, z
```

This is a good sign, and indeed, when we run our native app, it is finally able to complete the optimization task and return the result.

## Conclusion

In this post, we have looked at how to compile a non-trivial Java application that uses native libraries to a native binary with GraalVM.
I have found that the process of building native binaries with GraalVM is not straightforward when the your application has dependencies
on native libraries. This is something that is not covered in the Quarkus documentation, presumably because it is not a common use case.
However, I was happy to find that GraalVM itself has good documentation on the subject, and that there is tooling available to help us
in cases where the static analysis of GraalVM falls short.
