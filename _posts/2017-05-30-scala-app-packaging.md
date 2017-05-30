---
layout: default
title: Scala App Packaging in SBT
category: post
author: Jae
---

# Scala App Packaging in SBT

Spark을 사용하면서 `spark-submit`을 통해 실행되는 Application을 개발하기보다는 Zeppelin이나, `spark-shell`을 사용해서 작업을 처리하는것이 보통이었고, Spark Application을 개발하더라도 내가 개발한 Application을 다른 사람이 사용해야 하는 경우는 드물었는데 근래에 팀원들이 내가 만든 Spark Application을 사용해야 할 일이 생겼다.

'남들이 쓰기 좋게 만들어 주어야 너를 귀찮게 하지 않는다'는 존경하는 선배님의 말씀을 아로새겨 Spark, 정확히는 Scala Application을 Packaging 하는 방법에 대해서 조사해 보았다.

현재 서식하는 팀에서 빌드 툴로 SBT를 사용하고 있기 때문에 본 포스트는 SBT에서 Scala Application을 Package하는 방법을 주로 다룬다.

## Hello, World

본 포스트는 SBT를 사용한 Scala Application의 Packaging 에 대해서 다루고 있기 때문에 아래 간단한 Scala Application을 작성하였다.

그렇다. 우리가 아는 그놈이다.

```scala
package jae.lee.example

object helloworld {
  def main(args: Array[String]): Unit = {
    println("Hello, world!")
  }
}
```

build.sbt 파일은 아래와 같이 작성했다.

```scala
lazy val commonSettings = Seq(
  name := "test",
  version := "1.0",
  scalaVersion := "2.12.2"
)

lazy val app = (project in file(".")).
  settings(commonSettings: _*)
```

편의를 위해서 앞으로 위의 Scala Application을 'helloworld-app' 이라하고, `build.sbt`가 위치한 경로를 이 'root 경로'라고 한다.

## 일반적인 Packaging

root 경로에서 `sbt package` 명령을 실행하면 `target/scala-2.12/test_2.12-1.0.jar` 경로에 Application이 jar 파일로 배포된다.

```bash
$ sbt pacakge
```

생성된 jar 파일 내부를 살펴보면 Class 파일과 MANIFEST 파일이 존재함을 볼 수 있다.

```bash
$ jar tf test_2.12-1.0.jar
META-INF/MANIFEST.MF
jae/
jae/lee/
jae/lee/example/
jae/lee/example/helloworld$.class
jae/lee/example/helloworld.class
```

배포된 jar 파일은 다음과 같이 scala 명령을 이용해서 실행할 수 있다.

```bash
$ scala -cp target/scala-2.12/test_2.12-1.0.jar jae.lee.example.helloworld
Hello, world!
```

결과적으로 Scala Application은 JVM에서 돌아가므로 java 명령어를 이용해도 동일하게 실행됨을 기대 할 수 있다.

```bash
$ java -cp target/scala-2.12/test_2.12-1.0.jar jae.lee.example.helloworld
Exception in thread "main" java.lang.NoClassDefFoundError: scala/Predef$
	at jae.lee.example.helloworld$.main(helloworld.scala:5)
	at jae.lee.example.helloworld.main(helloworld.scala)
Caused by: java.lang.ClassNotFoundException: scala.Predef$
	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:335)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 2 more
```

한번에 될리가 없다....
scala 명령어를 사용하면 scala-library.jar를 자동으로 classpath에 추가하지만, java 명령어는 그렇지 않기 때문에 `scala/Predef` Class를 찾을 수 없다는 에러가 발생한다.
다음과 같이 scala-library.jar를 포함시켜주면 "Hello, World!"를 만날 수 있다.

```scala
$ java -cp target/scala-2.12/test_2.12-1.0.jar:$SCALA_HOME/lib/scala-library.jar jae.lee.example.helloworld
Hello, world!
```

#### TruobleShooting

아래 두 실행 명령은 여전히 `NoClassDefFoundError`가 발생한다.

```bash
$ java -cp $SCALA_HOME/lib/scala-library.jar -jar target/scala-2.12/test_2.12-1.0.jar
$ java -cp $SCALA_HOME/lib/scala-library.jar:target/scala-2.12/test_2.12-1.0.jar -jar target/scala-2.12/test_2.12-1.0.jar
```

[Java Man Page](http://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html)를 읽어보니 `-jar` 옵션을 사용하는 경우 다른 classpath 셋팅은 무시되고 온전히 `-jar` 옵션 뒤에 명시된 jar 파일 하나만을 사용하는듯 하다.


## sbt-assembly Plugin

본 포스트에서는 간단한 예제를 바탕으로 설명하고있기 때문에 필요한 library가 하나 뿐이지만, 보통의 경우 하나의 Application을 개발하기 위해서는 상당히 많은 library를 사용하게된다. 이런 Application을 배포하기 위해서는 위에서처럼, 사용된 library의 jar 파일과 classpath를 함께 제공해 주어야 하는데, 몇 십개씩되는 library를 일일이 관리하는 것은 여간 성가신일이 아닐 수 없다.

> 위대하신 선배 개발자 분들께서 이런 만행을 가만히 두셨을리 없다.

sbt진영에선 `sbt-assembly` plugin을 제공한다.

```scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.4")
```

> 명시된 버전은 2017-05-30 일 기준이다.

`sbt-assembly` plugin을 사용하기 위해서는 위와 같이 `project/plugins.sbt`에 `addSbtPlugin()`을 이용해서 plugin의 이름과 버전을 추가해 주어야한다. 이 후 `assembly`명령을 사용할 수 있다.

```bash
// sbt shell
> assembly
```

성공적으로 assembly 명령이 수행되면 `target/scala-2.12/test-assembly-1.0.jar` 경로에 assembly 된 jar가 배포된다. 배포된 jar 내용을 확인해보면..... 뭐가 많이 들어있는 것을 확인 할 수 있다.

```bash
$ jar tf test-assembly-1.0.jar
META-INF/MANIFEST.MF
jae/
jae/lee/
jae/lee/example/
scala/
scala/annotation/
scala/annotation/meta/
scala/annotation/unchecked/
scala/beans/
scala/collection/
scala/collection/concurrent/
scala/collection/convert/
scala/collection/generic/
<중략>
```

생성된 assembly jar는 일명 fat jar라고도 불리며 Application을 구동하는데 필요한 Class 파일들을 하나의 jar 내부에 모두 가지고 있기 때문에 실행가능한 jar, 즉, Executable jar 라고도 불린다.
Executable jar는 다음과 같이 실행할 수 있다.

```bash
$ java -jar target/scala-2.12/test-assembly-1.0.jar
Hello, world!
```

> build.sbt 의 내용을 수정해서 assembly 하는 동안 test를 생략하거나, Multi Project의 경우 MainClass를 지정해 줄 수 있다.
> 자세한 내용은 [sbt-assembly github page](https://github.com/sbt/sbt-assembly)를 참고하면 된다.


#### 다른 방법으로 Executable jar 만들기

사실 fat jar, Executable jar라는 것이 별것이 없다.
위에서 처럼 fat jar안에 library가 잔뜩 들어있지 않아도 helloworld-app을 실행할 수 있다.

> 아마도 intellij에서 sbt project를 만들 때 이정도는 사용하겠거니하고 기본으로 불러오는 library들을 다 묶은 모양이다.

다음과 같이 `scala-library.jar` 와 `target/scala-2.12/test_2.12-1.0.jar`를 하나의 jar로 합쳐주는 것 만으로 fat jar를 만들 수 있다.

```bash
$ mkdir ~/make-fat-jar
$ cp target/scala-2.12/test_2.12-1.0.jar ~/make-fat-jar
$ cp $SCALA_HOME/lib/scala-library.jar ~/make-fat-jar
$ cd ~/make-fat-jar
$ jar xf scala-library.jar
$ jar xf test_2.12-1.0.jar
$ jar cmf META-INF/MANIFEST.MF assembly.jar ./jae/* ./scala/*
$ java -jar assembly.jar
Hello, world!
```

> 위 코드에서는 jar 파일을 푸는 순서에 따라서 MANIFEST.MF 파일이 helloworld-app의 그것으로 덮어 씌어진다. MANIFEST.MF 파일 안의 `Main-Class`의 값이 `jae.lee.example.helloworld`이기 때문에 helloworld-app은 정상적으로 실행될 수 있다.


## sbt-native-packager Plugin

위에서는 assembly jar를 만들어 Scala Application을 배포하는 방법을 살펴봤다면 이번에는 `sbt-native-packager`를 사용한 배포방법을 알아보자.

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-native-packager" % "1.2.0-M9")
```

> 명시된 버전은 2017-05-30 일 기준이다.

`sbt-native-packager` plugin을 사용하기 위해서는 `sbt-assembly` plugin을 추가했을 때와 동일한 방법으로 `project/plugins.sbt`에 plugin의 이름과 버전을 추가해주고, build.sbt에 `enablePlugins(JavaAppPackaging)`를 추가해준다. 이 후 plugin이 제공하는 다양한 명령들을 사용할 수 있다.

```scala
// build.sbt
lazy val commonSettings = Seq(
  name := "test",
  version := "1.0",
  scalaVersion := "2.12.2"
)

lazy val app = (project in file(".")).
  settings(commonSettings: _*).
  enablePlugins(JavaAppPackaging)
```

`sbt-native-packager`를 사용할 준비가 끝났다면, `sbt stage` 명령을 이용해서 Staging 버전을 생성해 볼 수 있다.

```bash
$ sbt stage
$ ls target/universal/stage/
bin	lib
```

각 폴더의 구성은 다음과 같다.

```bash
bin/
  <app_name>       <- BASH script
  <app_name>.bat   <- cmd.exe script
lib/
   <Your project and dependent jar files here.>
```

`bin` 폴더에는 Application을 실행해주는 script가 생성된다.
`lib` 폴더에는 Application을 실행하는데 필요한 jar 파일들이 존재하게 된다.

생성된 script의 내용을 살펴보면 lib경로에 있는 jar 파일을 classpath로 지정하여 java 명령을 통해서 실행해주는것을 볼 수 있다.

helloworld-app의 실제 산출물은 다음과 같다.

```bash
stage
    ├── bin
    │   ├── test
    │   └── test.bat
    └── lib
        ├── org.scala-lang.scala-library-2.12.2.jar
        └── test.test-1.0.jar
```

script를 실행해보면 정상적으로 실행된다. (야호!)

```bash
./test
Hello, world!
```

> script를 실행할 때 JAVA_OPTS 또는 Application의 arguments와 함께 실행할 수도 있다.
> 자세한 내용은 [Start script options](http://www.scala-sbt.org/sbt-native-packager/archetypes/java_app/index.html#start-script-options)를 참고한다.

#### TroubleShooting

이렇게 만들어진 Spark Application을 실행하면 `spark-submit`을 통해서 실행되지 않기 때문에 log4j.properties 파일이 정상적으로 Load되지 않는다...
이 경우 JAVA_OPTS를 사용해서 log4j.properties 파일을 Spark이 사용할 수 있도록 해줄 수 있다.

```
./spark-application -Dlog4j.configuration=file:$SPARK_HOME/conf/log4j.properties
```

끝으로 운영체제에 맞는 패키지를 생성하려면 `sbt shell`에서 아래 명령어를 사용한다.

* universal:packageBin - Generates a universal zip file
* universal:packageZipTarball - Generates a universal tgz file
* debian:packageBin - Generates a deb
* docker:publishLocal - Builds a Docker image using the local Docker server
* rpm:packageBin - Generates an rpm
* universal:packageOsxDmg - Generates a DMG file with the same contents as the universal zip/tgz.
* windows:packageBin - Generates an MSI

`debian:packageBin` 명령어를 사용하면 `fakeroot`를 사용해서 우분투 패키지를 생성하고, `rpm:packageBin` 명령어를 사용하기 위해서는 `rpmVendor in Rpm` 설정을 채워주어야 한다.
자세한 내용은 [SBT Native Packager](https://github.com/sbt/sbt-native-packager)를 참고한다.

## 참고
* [sbt-assembly github page](https://github.com/sbt/sbt-assembly)
* [SBT Native Packager](https://github.com/sbt/sbt-native-packager)