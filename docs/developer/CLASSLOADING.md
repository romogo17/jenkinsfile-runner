# Implementation Note

Jenkinsfile Runner embeds Jenkins in a command line tool, then execute a pipeline inside.
In order to allow users to choose Jenkins version as
well as plugins, it creates a classloader hierarchy and different portion of the code
runs in different classloader.

```
       +----------+       +-----------+
       | System   |       | Bootstrap |
       +----+-----+       +-----------+
            ^
       +----+-----+
       | Jenkins  |
       +-+-----+--+
         ^     ^
+--------+-+  ++---------+
| Setup    |  |Plugins   |
+----------+  ++---------+
               ^
              ++---------+
              |Payload   |
              +----------+
```

The `Boot` classloader is what comes from `java -cp` in the wrapper script.
This is the first to execute, but it contains all sorts of jars and missing
others, so it's not suitable for any actual work. We use it only to build
the rest of the classloader hierachy, then jump to the setup.

`System` classloader is the one JavaVM implicitly creates to host JavaSE runtime.

`Jenkins` classloader contains the Jenkins core, which comes from `jenkins.war`
given by a user.

`Setup` classloader contains Jetty, Jenkins unit test harness, and other garbage
that we don't use but Jenkins unit test harness depends on. By segregating this
into a separate classloader, we prevent them from interfering with Jenkins core
and its plugins. This setup module starts Jetty, load Jenkins as a war, and boots
Jenkins like it normally does. Once that's done, the execution jumps to the payload.

`Plugins` classloader is actually a web of classloaders created by Jenkins to load
plugins. This gets created as Jenkins boots.

`Payload` classloader is where the actual work of running Jenkinsfile happens.
This module needs to be able to rely on pipeline plugin families, so it gets loaded
at the very end by depending on the uber classloader that sees all the plugins.

Different code that runs in different classloaders are housed in different Maven modules
to clarify this isolation.
