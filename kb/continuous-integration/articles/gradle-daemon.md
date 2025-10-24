---
title: Gradle build and daemon issues
description: These are some Gradle issues you might encounter with Harness CI.
redirect_from:
  - /kb/continuous-integration/articles/leverage-service-dependencies-in-gradel-daemon-to-improve-build-performance
---

This article addresses some Gradle issues you might encounter with [Harness Continuous Integration (CI)](https://developer.harness.io/docs/continuous-integration/get-started/overview).

## Out of memory errors with Gradle

If a Gradle build experiences out of memory errors, add the following to your `gradle.properties` file:

```
-XX:+UnlockExperimentalVMOptions -XX:+UseContainerSupport
```

Your Java options must use [UseContainerSupport](https://eclipse.dev/openj9/docs/xxusecontainersupport/) instead of `UseCGroupMemoryLimitForHeap`, which was removed in JDK 11.

## Configure service dependencies in Gradle builds

You can use [Background steps](https://developer.harness.io/docs/continuous-integration/use-ci/manage-dependencies/background-step-settings) to manage long-running service dependencies for Gradle builds. You can also launch services ad-hoc by using commands or flags (such as `--daemon`) in your steps.

### Enable the Gradle daemon in builds

To enable the Gradle daemon in your Harness CI builds, include the `--daemon` option when running Gradle commands in your build scripts (such as in [Run steps](https://developer.harness.io/docs/continuous-integration/use-ci/run-step-settings) or in build arguments for [Build and Push steps](https://developer.harness.io/docs/category/build-and-push)). This option instructs Gradle to use the daemon process.

Optionally, you can [use Background steps to optimize daemon performance](#manage-gradle-daemon-dependencies-to-improve-performance).

### Manage Gradle daemon dependencies to improve performance

In Harness CI builds, each step runs in a separate container. If your pipeline has multiple steps that utilize Gradle through CLI, each step initiates a separate Gradle daemon (unless you explicitly set `--no-daemon`).

To optimize the build process and improve performance, use a [Background step](https://developer.harness.io/docs/continuous-integration/use-ci/manage-dependencies/background-step-settings) to create a single Gradle daemon that can be used by all subsequent steps in the stage. This reduces the overhead of daemon startup and enhances the efficiency of the entire pipeline.

```yaml
              - step:
                  type: Background
                  name: Gradle_Daemon
                  identifier: Gradle_Daemon
                  spec:
                    connectorRef: YOUR_DOCKER_CONNECTOR_ID
                    image: gradle
                    shell: Sh
                    entrypoint:
                      - gradle
                    envVariables:
                      GRADLE_USER_HOME: /harness/.gradle
                      GRADLE_OPTS: "-Dorg.gradle.jvmargs=\"-Xms1024m -Xmx2048m\""
                    resources:
                      limits:
                        memory: 1G
```

## Parallel Gradle builds fail with "Currently in use by another Gradle instance" or "Lock artifact cache"

For Gradle, there is a limitation regardless of Harness.   Users can’t run Gradle in two or more different processes in parallel.  This results in a failure due to some format of locking.  This also occurs locally if running two or more Gradle processes in parallel utilizing shared resources.

Some sample of errors include:

```
Timeout waiting to lock artifact cache (/common/user/.gradle/caches/modules-2). It is currently in use by another Gradle instance.
Owner PID: 1XXXX
Our PID: 1XXXX
Owner Operation: resolve configuration ':classpath’
Our operation: resolve configuration ':classpath’
Lock file: /common/user/.gradle/caches/modules-2/modules-2.lock
```

```
Timeout waiting to lock checksums cache. It is currently in use by another Gradle instance.
```

Gradle is reliant on having unique structures due to their locks in three areas
* GRADLE_USER_HOME
* Build Cache Directory
* Project directory

It is possible to hit various conflicts, but it all comes down to one important thing - pods/steps in the same stage share disk causing lock file(s) or directory with another Gradle process.

Users need to be careful of instances where they are creating Gradle operations in **steps** or **step groups** where there are instances of matrix and parallelism strategies, or parallel processes.   This can cause multiple pods and or containers to run in parallel or overlap, all sharing the same disk space.  This then leads the Gradle runs to fail in some of the steps.

Note that this applies to Run steps, or Harness Test Intelligence step alike (e.g the Java agent in Harness Test Intelligence).

Of particular note, setting GRADLE_USER_HOME as a unique folder *does not help* as the lock file is being set in the root folder of the project: /harness/.gradle/noVersion/buildLogic.lock and Gradle does not support changing this location.

To solve this, Harness recommends considering one of the following three options
* Separate stages (can be run in parallel) 
* Separate the Git folders used for building
* Having a preliminary step to pre populate the dependency cache for Gradle in a shared folder, like [shown in this Stackoverflow article](https://stackoverflow.com/questions/74339765/how-can-i-make-one-single-gradle-cache-for-multiple-projects)



For more information and discussion on this Gradle error, go to:

* [Gradle Forums - Timeout waiting to lock checksums cache in Kubernetes pods](https://discuss.gradle.org/t/timeout-waiting-to-lock-checksums-cache-in-kubernetes-pods/44169)
* [StackOverflow - It is currently in use by another Gradle instance](https://stackoverflow.com/questions/21523508/it-is-currently-in-use-by-another-gradle-instance)

## Gradle version not compatible with Test Intelligence.

If you use Java with Gradle, Test Intelligence assumes `./gradlew` is present in the root of your project. If not, TI falls back to the Gradle tool to run the tests. As long as your Gradle version has test filtering support, it is compatible with Test Intelligence.

Add the following to your `build.gradle` to make it compatible with Test Intelligence:

```
// This adds HARNESS_JAVA_AGENT to the testing command if it's
// provided through the command line.
// Local builds will still remain same as it only adds if the
// parameter is provided.
tasks.withType(Test) {
  if(System.getProperty("HARNESS_JAVA_AGENT")) {
    jvmArgs += [System.getProperty("HARNESS_JAVA_AGENT")]
  }
}

// This makes sure that any test tasks for subprojects don't
// fail in case the test filter does not match.
gradle.projectsEvaluated {
        tasks.withType(Test) {
            filter {
                setFailOnNoMatchingTests(false)
            }
        }
}
```
