
## Running maddi on these projects

Prereqs: JDK 26 (`JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-26.jdk/Contents/Home`).
The openjdk front-end needs the javac `--add-exports`; keep them in `MADDI_EXPORTS`:

```
MADDI_EXPORTS="--add-exports jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED \
 --add-exports jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED \
 --add-exports jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED \
 --add-exports jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED \
 --add-exports jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED"
```

Analyse **prep only** for now (`-DanalysisSteps=prep` / `--analysis-steps prep`); modification is still too slow.

### Maven projects (e.g. Jenkins) — the maddi Maven plugin

Install the plugin once (in the maddi repo): `./gradlew :maddi-mvnplugin:publishToMavenLocal`. Then, from anywhere
in the reactor, select the module with `-pl`:

```
cd jenkins
env MAVEN_OPTS="$MADDI_EXPORTS -Xmx6G" mvn \
  -pl cli generate-test-sources io.codelaser:maddi-mvnplugin:0.8.2:run -DanalysisSteps=prep
```

Notes: run `generate-test-sources` first (maddi analyses main *and* test) so every generated/added source root
is registered — this includes `build-helper-maven-plugin` `add-test-source` roots bound to that phase (e.g.
langchain4j-core pulls in `../langchain4j-test/src/main/java` there; plain `generate-sources` misses it);
`jmods` defaults to `java.se` (the whole JDK on the classpath) — override with `-Djmods=java.base` for a leaner run;
source directories are emitted absolute, so the goal no longer needs to run from the module directory;
for a multi-module reactor, `mvn install -pl <module> -am -DskipTests` first so the plugin can resolve sibling jars;
`...:write-input-configuration` (instead of `:run`) just writes `target/inputConfiguration.json` and exits.

### Gradle projects (e.g. Fernflower) — derive a config from the compile log

No Maven plugin; capture the Gradle compiler arguments and feed them to maddi's `--compile-log`:

```
cd fernflower
./gradlew --no-build-cache --rerun-tasks compileJava compileTestJava --debug 2>&1 \
  | grep -a 'Compiler arguments:' > /tmp/compile.log
env JAVA_TOOL_OPTIONS="$MADDI_EXPORTS -Xmx6G" maddi \
  --compile-log /tmp/compile.log --write-input-configuration inputConfiguration.json
env JAVA_TOOL_OPTIONS="$MADDI_EXPORTS -Xmx6G" maddi \
  --input-configuration inputConfiguration.json --analysis-steps prep
```

(`maddi` = the openjdk runner, `org.e2immu.analyzer.run.openjdkmain.Main`; the `--compile-log` path adds the
`java.se` jmod closure automatically. See `maddi-run-openjdk/running-examples.md` in the maddi repo.)

## Status: Built

Note: each line with 4 number is the output of `cloc` for that language.

### ActiveMQ

```
mvn clean install -DskipTests
```

Java                          4697         127230         242504         588820


### Camel

```
./mvnw install -DskipTests
```

675 modules!

Java                         26710         474618        1221995        2539205

### Gradle

```
./gradlew compileTestJava
```

Groovy                             6566         181162         116505         854036
Java                              11416         125948         285637         581293
Kotlin                             5710          84679         212282         356044

### ElasticSearch

```
./gradlew compileTestJava
```

Java                         32084         760564         757554        5118018

### Fernflower

```
./gradlew compileTestJava
```

Java                           501          10396           3092          57685

### Google Guava

```
mvn install -DskipTests
```

Java                          1926          60575         133092         350565

### Hadoop

```
./mvnw install -DskipTests
```
takes ages, especially the shaded builds.

```
./mvnw test-compile
```

Java                         12618         403631         830231        2809199


### Hazelcast

```
./mvnw clean install -DskipTests
```

Java                         11576         287855         362382        1495977


### Hibernate ORM

```
./gradlew compileTestJava
```

Java                           23401         422303         533589        1846364

### Hive

```
mvn install -DskipTests
```
succeeds on almost all projects (54/57)

Java                          9532         387854         486579        2454554

### Jenkins

```
mvn install -DskipTests -Dlicense.disableCheck
```

Java                          2028          44345         111027         217109

### Keycloak

```
./mvnw clean install -DskipTests
```

Java                          8299         195924         200218         794721

### Langchain4j

```
mvn clean install -DskipTests
```

Java                          2864          59420          42168         272317

### Pulsar

```
./gradlew compileTestJava -PskipJavaVersionCheck
```

Java                          4755         117863         189611         803604


### Quarkus

```
./mvnw clean install -DskipTests
```

Java                           22298         299007         206293        1427748


### Timefold-solver

```
./mvnw  clean  install -DskipTests
```

Java                            3620          53062          34337         257969


### Tomcat

```
ant
```

Java                                   2807          95637         210854         377480


### Trino

use the Temurin build, in bash:
```
JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-26.jdk/Contents/Home ./mvnw clean install -DskipTests=true -Duse.jdk17=true
```

Java                         11182         234865         223771        1605465

### Wildfly

Add commons-collections license
```
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <licenses>
        <license>
            <name>Apache License, Version 2.0</name>
            <url>https://www.apache.org/licenses/LICENSE-2.0.txt</url>
        </license>
    </licenses>
</dependency>
```
to the license files
```
find . -name "*licenses.xml"
```
then
```
./mvnw clean install -DskipTests
```

Java                                  10094         128067         144293         604068


## Status: Not yet built/Cannot parse/Wrong JDK/...


### Cassandra

Ant build system


### Druid

???


JavaScript frontend



### Neo4J

ignore, contains a lot of Scala


### Spring

??


### Netty

Needs 'cmake', install via brew.
```
brew install autoconf automake libtool
./mvnw clean install -DskipTests
```
still, fails on cmake?

### Sonarqube

```
./gradlew -Dorg.gradle.java.home=$(/usr/libexec/java_home -v 21) --stop
./gradlew -Dorg.gradle.java.home=$(/usr/libexec/java_home -v 21) compileTestJava
```


