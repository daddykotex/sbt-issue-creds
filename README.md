# sbt issue

I have an issue with sbt under the following conditions:

1. version 1.5.5
2. using a private repository (`~/.sbt/repositories`) along with `-Dsbt.override.build.repos=true`
3. sbt has never started on the machine

## TLDR

1. You're using the default sbt root dir (`~/.sbt`) and (`~/.sbt/1.0/` for settings and `~/.sbt/1.0/plugins` for plugins)
2. You have a blank install (configuration is there, but sbt has never started)
3. The directory `~/.sbt/1.0/plugins` exists
4. `sbt` will fail to start


```bash
docker run -it --entrypoint /bin/bash --link nexus:nexus sbt-works
root:/reproduction#> tree ~/.sbt
/root/.sbt
├── 1.0
│   └── credentials.sbt
└── repositories

1 directory, 2 files
root:/reproduction#> sbt clean > /dev/null && echo Woo
[info] [launcher] getting org.scala-sbt sbt 1.5.5  (this may take some time)...
[info] [launcher] getting Scala 2.12.14 (for sbt)...
Woo
root:/reproduction#> sbt clean > /dev/null || echo "WOOPS" # see the stack below
WOOPS
root@6fdc41813aa7:/reproduction#
```

## Requirements: a private repository

1. start nexus: `docker run -d -p 8081:8081 --name nexus sonatype/nexus3@sha256:f7c805f51a44dc55163dc05525c96dc845c6cff58572c3cc02af00dfb9a111ba`
1. allow a min or two to start
1. once started, grab the admin password: `docker exec -it nexus /bin/bash -c 'cat /nexus-data/admin.password'`
1. head on to the nexus to complete setup: http://localhost:8081
  1. sign in using user: `admin` and the password grabbed in the step above
  2. set a new password to `admin`
  3. disable anonymous access

At that point we have a private repo running. The steps above (to start the nexus container) only need to be done once (except if you stop/rm the container).

*Note: the repository, by default, does not know where to find sbt-plugins. if you want to do that, you'll need to configure a proxy to `https://repo.scala-sbt.org/scalasbt/sbt-plugin-releases/` *

## To reproduce

I've encoded the steps to reproduce inside docker files. The reason is that reproduction is easy when the setup is clean. When you try to reproduce on your machine, you pollute all sorts of directories with cache that may make the problem go away.

1. the `Dockerfile.base` gives you the minimum to run `sbt` using only on repository (the nexus we started above) (only the initial jar downloaded by the launcher comes from else where)
2. the `Dockerfile.broken` makes one modification on top of the base image that breaks sbt. ** it creates a plugin directory **

To reproduce build the two image:

1. `docker build -t sbt-works -f Dockerfile.base . && docker build -t sbt-broken -f Dockerfile.broken .`
2. confirm `sbt-works` is fine by running `docker run -it --link nexus:nexus sbt-works run`
2. confirm `sbt-broken` is **broken** by running `docker run -it --link nexus:nexus sbt-broken run`

For some reason, when the `plugins` folder exists, credentials are not used when resolving some artifacts for the build, and we get the following exceptions:

```
david@FRANCOED1 ~/D/repo> docker run -it --link nexus:nexus sbt-broken run
[info] [launcher] getting org.scala-sbt sbt 1.5.5  (this may take some time)...
[info] [launcher] getting Scala 2.12.14 (for sbt)...
[info] Updated file /reproduction/project/build.properties: set sbt.version to 1.5.5
[info] welcome to sbt 1.5.5 (Temurin Java 1.8.0_312)
[info] loading global plugins from /root/.sbt/1.0/plugins
[info] Updating
[info] Resolved  dependencies
[warn]
[warn] 	Note: Unresolved dependencies path:
[error] sbt.librarymanagement.ResolveException: Error downloading org.scala-sbt:sbt:1.5.5
[error]   Not found
[error]   Not found
[error]   not found: /root/.ivy2/localorg.scala-sbt/sbt/1.5.5/ivys/ivy.xml
[error]   unauthorized: http://nexus:8081/repository/maven-central/org.scala-sbt/sbt/1.5.5/ivys/ivy.xml (Sonatype Nexus Repository Manager)
[error]   unauthorized: http://nexus:8081/repository/maven-central/org/scala-sbt/sbt/1.5.5/sbt-1.5.5.pom (Sonatype Nexus Repository Manager)
[error] Error downloading org.scala-lang:scala-library:2.12.14
[error]   Not found
[error]   Not found
[error]   not found: /root/.ivy2/localorg.scala-lang/scala-library/2.12.14/ivys/ivy.xml
[error]   unauthorized: http://nexus:8081/repository/maven-central/org.scala-lang/scala-library/2.12.14/ivys/ivy.xml (Sonatype Nexus Repository Manager)
[error]   unauthorized: http://nexus:8081/repository/maven-central/org/scala-lang/scala-library/2.12.14/scala-library-2.12.14.pom (Sonatype Nexus Repository Manager)
[error] 	at lmcoursier.CoursierDependencyResolution.unresolvedWarningOrThrow(CoursierDependencyResolution.scala:258)
[error] 	at lmcoursier.CoursierDependencyResolution.$anonfun$update$38(CoursierDependencyResolution.scala:227)
[error] 	at scala.util.Either$LeftProjection.map(Either.scala:573)
[error] 	at lmcoursier.CoursierDependencyResolution.update(CoursierDependencyResolution.scala:227)
[error] 	at sbt.librarymanagement.DependencyResolution.update(DependencyResolution.scala:60)
[error] 	at sbt.internal.LibraryManagement$.resolve$1(LibraryManagement.scala:59)
[error] 	at sbt.internal.LibraryManagement$.$anonfun$cachedUpdate$12(LibraryManagement.scala:133)
[error] 	at sbt.util.Tracked$.$anonfun$lastOutput$1(Tracked.scala:73)
[error] 	at sbt.internal.LibraryManagement$.$anonfun$cachedUpdate$20(LibraryManagement.scala:146)
[error] 	at scala.util.control.Exception$Catch.apply(Exception.scala:228)
[error] 	at sbt.internal.LibraryManagement$.$anonfun$cachedUpdate$11(LibraryManagement.scala:146)
[error] 	at sbt.internal.LibraryManagement$.$anonfun$cachedUpdate$11$adapted(LibraryManagement.scala:127)
[error] 	at sbt.util.Tracked$.$anonfun$inputChangedW$1(Tracked.scala:219)
[error] 	at sbt.internal.LibraryManagement$.cachedUpdate(LibraryManagement.scala:160)
[error] 	at sbt.Classpaths$.$anonfun$updateTask0$1(Defaults.scala:3678)
[error] 	at scala.Function1.$anonfun$compose$1(Function1.scala:49)
[error] 	at sbt.internal.util.$tilde$greater.$anonfun$$u2219$1(TypeFunctions.scala:62)
[error] 	at sbt.std.Transform$$anon$4.work(Transform.scala:68)
[error] 	at sbt.Execute.$anonfun$submit$2(Execute.scala:282)
[error] 	at sbt.internal.util.ErrorHandling$.wideConvert(ErrorHandling.scala:23)
[error] 	at sbt.Execute.work(Execute.scala:291)
[error] 	at sbt.Execute.$anonfun$submit$1(Execute.scala:282)
[error] 	at sbt.ConcurrentRestrictions$$anon$4.$anonfun$submitValid$1(ConcurrentRestrictions.scala:265)
[error] 	at sbt.CompletionService$$anon$2.call(CompletionService.scala:64)
[error] 	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
[error] 	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
[error] 	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
[error] 	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
[error] 	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
[error] 	at java.lang.Thread.run(Thread.java:748)
[error] (update) sbt.librarymanagement.ResolveException: Error downloading org.scala-sbt:sbt:1.5.5
[error]   Not found
[error]   Not found
[error]   not found: /root/.ivy2/localorg.scala-sbt/sbt/1.5.5/ivys/ivy.xml
[error]   unauthorized: http://nexus:8081/repository/maven-central/org.scala-sbt/sbt/1.5.5/ivys/ivy.xml (Sonatype Nexus Repository Manager)
[error]   unauthorized: http://nexus:8081/repository/maven-central/org/scala-sbt/sbt/1.5.5/sbt-1.5.5.pom (Sonatype Nexus Repository Manager)
[error] Error downloading org.scala-lang:scala-library:2.12.14
[error]   Not found
[error]   Not found
[error]   not found: /root/.ivy2/localorg.scala-lang/scala-library/2.12.14/ivys/ivy.xml
[error]   unauthorized: http://nexus:8081/repository/maven-central/org.scala-lang/scala-library/2.12.14/ivys/ivy.xml (Sonatype Nexus Repository Manager)
[error]   unauthorized: http://nexus:8081/repository/maven-central/org/scala-lang/scala-library/2.12.14/scala-library-2.12.14.pom (Sonatype Nexus Repository Manager)
[warn] Project loading failed: (r)etry, (q)uit, (l)ast, or (i)gnore? (default: r)
```