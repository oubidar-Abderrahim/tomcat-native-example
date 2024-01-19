# Run Tomcat Server As Native Image

## Overview  
* This demo provides steps to run Tomcat server along with the web applications in terms of GraalVM native image.  
* The steps reference Tomcat documentation https://tomcat.apache.org/tomcat-11.0-doc/graal.html  
        1. Generate a single executable JAR file, including dependencies for Tomcat server and running applications.  
        2. Generate metadata for native image creation using the agent tool provided by GraalVM for the JAR file.  
        3. Generate the native image using the native build tool provided by GraalVM.  
* The sample application in this demo implements Spring Framework 6.x.

## Demo Environment
* [Tomcat 10.0.27](https://tomcat.apache.org/download-10.cgi)
* [GraalVM for JDK 21](https://www.oracle.com/jp/java/technologies/downloads/#graalvmjava21)
* [Apache Maven 3.6.3](https://maven.apache.org/download.cgi)
* [Apache Ant 1.10.14](https://ant.apache.org/bindownload.cgi)

> **NOTE:** It is assumed that all the above softwares are installed. Here is an example of setting environment variables in the ~/.bashrc file for the installed software on Linux.
```
export CATALINA_HOME=/opt/apache-tomcat-10.0.27
export PATH=$CATALINA_HOME/bin:$PATH

export JAVA_HOME=/usr/lib64/graalvm/graalvm-java21
export PATH=$JAVA_HOME/bin:$PATH

export MVN_HOME=/opt/apache-maven-3.6.3
export PATH=$MVN_HOME/bin:$PATH

export ANT_HOME=/opt/apache-ant-1.10.14
export PATH=$ANT_HOME/bin:$PATH

```

## Contents
* **[1.Deploying the Sample Application to Tomcat](#1-Deploying-the-Sample-Application-to-Tomcat)**

* **[2.Downloading AOT Templates](#2-Downloading-AOT-Templates)**
   
* **[3.Packaging and Native Build](#3-Packaging-and-Native-Build)**

* **[4.Run Tomcat native image in Docker container](#4-Run-Tomcat-native-image-in-Docker-container)**

## 1. Deploying the Sample Application to Tomcat  
Create a web application using Spring Framework, deploy it as a WAR file to Tomcat. Later, we will build this application along with Tomcat server into native-imaged.

```
$ git clone https://github.com/junsuzu/tomcat-native-jp
$ cd spring-framework-tomcat-sample
$ mvn clean package
$ cd target
```

Verify that the springTomcat.war file is generated in the target directory.
```
[opc@instance target]$ ls -la
total 4276
drwxrwxr-x. 7 opc opc     132 Dec 24 06:09 .
drwxrwxr-x. 4 opc opc      46 Dec 24 06:09 ..
drwxrwxr-x. 3 opc opc      17 Dec 24 06:09 classes
drwxrwxr-x. 3 opc opc      25 Dec 24 06:09 generated-sources
drwxrwxr-x. 2 opc opc      28 Dec 24 06:09 maven-archiver
drwxrwxr-x. 3 opc opc      35 Dec 24 06:09 maven-status
drwxrwxr-x. 4 opc opc      54 Dec 24 06:09 springTomcat
-rw-rw-r--. 1 opc opc 4378492 Dec 24 06:09 springTomcat.war
```
Copy the springTomcat.war file to the webapps directory of the Tomcat server, and the WAR file will be automatically deployed, and springTomcat folder will be generated. Start the tomcat server, and verify springTomcat application runs normally with display message of "Hello Spring Framework World".

```
[opc@jms-instance-2 target]$ cd /opt/apache-tomcat-10.0.27/bin
[opc@jms-instance-2 bin]$ ./startup.sh
Using CATALINA_BASE:   /opt/apache-tomcat-10.0.27
Using CATALINA_HOME:   /opt/apache-tomcat-10.0.27
Using CATALINA_TMPDIR: /opt/apache-tomcat-10.0.27/temp
Using JRE_HOME:        /usr/lib64/graalvm/graalvm-java21
Using CLASSPATH:       /opt/apache-tomcat-10.0.27/bin/bootstrap.jar:/opt/apache-tomcat-10.0.27/bin/tomcat-juli.jar
Using CATALINA_OPTS:   
Tomcat started.
[opc@jms-instance-2 bin]$ curl http://localhost:8080/springTomcat/greeting
Hello Spring Framework World
[opc@jms-instance-2 bin]$
```

## 2. Downloading AOT Templates

Follow the instructions in the official Tomcat documentation https://tomcat.apache.org/tomcat-11.0-doc/graal.html to download the Tomcat Stuffed module.
```
git clone https://github.com/apache/tomcat.git

```
All of the successor tasks will be conducted under the stuffed folder. 
```
cp -r tomcat/modules/stuffed ../tomcat-native-jp/

```
> **NOTE:** For reference, the stuffed folder after completing all tasks is stored in the complete folder.  
> **NOTE:** Define the location of 'stuffed' as an environment variable. The following is an example of the definition in ~/.bashrc: 
```
export TOMCAT_STUFFED=/home/opc/project/tomcat-native-jp/stuffed

```

Copy the deployed web applications of springTomcat from the Tomcat server to the stuffed/webapps directory.
```
cp -r $CATALINA_HOME/webapps/springTomcat $TOMCAT_STUFFED/webapps/

```
Similarly copy default web applicatins ROOT and manager.
```
cp -r $CATALINA_HOME/webapps/ROOT $TOMCAT_STUFFED/webapps/
cp -r $CATALINA_HOME/webapps/manager $TOMCAT_STUFFED/webapps/
```

Copy the Java source files under spring-framework-tomcat-sample/src/main/java to the stuffed/webapps directory.  

```
cd spring-framework-tomcat-sample
cp -r src/main/java/* $TOMCAT_STUFFED/webapps/springTomcat/WEB-INF/classes/
```

Copy all files under the Tomcat server's conf directory to the stuffed/conf directory. 
```
cp -r $CATALINA_HOME/conf/* $TOMCAT_STUFFED/conf/
```

## 3. Packaging and Native Build
Modify the pom.xml file under the stuffed directory.

Change the tomcat.version property to the actual version of Tomcat you are using.
Add the spring-framework.version property.
> **NOTE:** Add libraries and plugins as necessary to suit your actual web application.  

```
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <mainClass>org.apache.catalina.startup.Tomcat</mainClass>
    <!--tomcat.version>11.0.0-M14</tomcat.version-->
    <tomcat.version>10.0.27</tomcat.version>
    <!--add for springframework-->
    <spring-framework.version>6.0.2</spring-framework.version>
</properties>
```

Add dependencies for the Spring Framework in the dependencies section.
```
<dependencies>
        <!-- add for springframework -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring-framework.version}</version>
        </dependency>       
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring-framework.version}</version>
        </dependency>
        <!-- add for springframework -->
.....................
</dependencies>
```

Build with maven.
```
cd stuffed
mvn package
```
Build with Ant. Change the "webapp.name" variable to actual web application names.

```
ant -Dwebapp.name=springTomcat -f webapp-jspc.ant.xml
ant -Dwebapp.name=ROOT -f webapp-jspc.ant.xml
ant -Dwebapp.name=manager -f webapp-jspc.ant.xml
```

Build again with Maven.
```
mvn package
```

Run following java command to generate Tomcat embedded code.
```
$JAVA_HOME/bin/java\
        -Dcatalina.base=. -Djava.util.logging.config.file=conf/logging.properties\
        -jar target/tomcat-stuffed-1.0.jar --catalina -generateCode src/main/java
```
> **NOTE:** At runtime, an error message will appear stating that the Tomcat logging library related to Juli cannot be found, but this does not matter with for the purpose of this demo. If you want to avoid error messages, edit logging.properties under the conf directory and comment out the Juli-related library description.

Use Ctrl+C to stop the process and build again with Maven.
```
mvn package
```

Generate metadata for native image by using the agent tool to resolve Java Reflection.
```
$JAVA_HOME/bin/java\
        -agentlib:native-image-agent=config-output-dir=$TOMCAT_STUFFED/target/\
        -Dorg.graalvm.nativeimage.imagecode=agent\
        -Dcatalina.base=. -Djava.util.logging.config.file=conf/logging.properties\
        -jar target/tomcat-stuffed-1.0.jar --catalina -useGeneratedCode
```
All patterns included in the web application must be executed to generate metadata for the native image. The web application we are using this time has a context root and a URL pattern of "greeting" for the servlet to process, so we will launch a separate terminal and execute each of these two patterns.
```
[opc@jms-instance-2 /]$ curl http://localhost:8080/springTomcat/
<html>
<body>
<h2>Hello Spring!</h2>
</body>
</html>
[opc@jms-instance-2 /]$ curl http://localhost:8080/springTomcat/greeting
Hello Spring Framework World
```

Use Ctrl+C to stop the above java process.
Use GraalVM native build tool to build native image.

```
native-image --no-server\
        --allow-incomplete-classpath --enable-https\
        --initialize-at-build-time=org.eclipse.jdt,org.apache.el.parser.SimpleNode,jakarta.servlet.jsp.JspFactory,org.apache.jasper.servlet.JasperInitializer,org.apache.jasper.runtime.JspFactoryImpl\
        -H:+JNI -H:+ReportUnsupportedElementsAtRuntime\
        -H:+ReportExceptionStackTraces -H:EnableURLProtocols=http,https,jar,jrt\
        -H:ConfigurationFileDirectories=$TOMCAT_STUFFED/target/\
        -H:ReflectionConfigurationFiles=$TOMCAT_STUFFED/tomcat-reflection.json\
        -H:ResourceConfigurationFiles=$TOMCAT_STUFFED/tomcat-resource.json\
        -H:JNIConfigurationFiles=$TOMCAT_STUFFED/tomcat-jni.json\
        -jar $TOMCAT_STUFFED/target/tomcat-stuffed-1.0.jar
```
Confirm that the native executable "tomcat-stuffed-1.0" is generated under the stuffed directory.
Start the native image.
```
./tomcat-stuffed-1.0 -Dcatalina.base=. -Djava.util.logging.config.file=conf/logging.properties --catalina -useGeneratedCode
```
Use another terminal to verify that the native image is working.
```
[opc@jms-instance-2 /]$ curl http://localhost:8080/springTomcat/greeting
Hello Spring Framework World
```

## 4. Run Tomcat native image in Docker container

Modify the DockerfileGraal under the stuffed directory and change the base OS image from busy:box to oraclelinux:8-slim.

```
# FROM busybox:glibc
FROM oraclelinux:8-slim
```

Build Dokcer image.
```
docker build -t apache/tomcat-stuffed-native:1.0 -f ./DockerfileGraal .
```
Start the Docker container and check that the Tomcat server and springframework sample are working properly.

```
docker run --name tomcat-native -p 8080:8080 apache/tomcat-stuffed-native:1.0
```
```
[opc@jms-instance-2 tomcat-native-jp]$ curl http://localhost:8080/springTomcat/greeting
Hello Spring Framework World
```

Compare the application execution time when running the native image of the Tomcat server in a container with the execution time when running a conventional Tomcat server. In this demo environment, it is confirmed that the native image runs 20 times faster than traditional Tomcat server.

*native image
```
[opc@jms-instance-2 tomcat-native-jp]$ docker start tomcat-native
tomcat-native
[opc@jms-instance-2 tomcat-native-jp]$ time curl http://localhost:8080/springTomcat/greeting
Hello Spring Framework World

real    0m0.012s
user    0m0.003s
sys     0m0.005s
```

*Tomcat server
```
[opc@jms-instance-2 tomcat-native-jp]$ startup.sh
Using CATALINA_BASE:   /opt/apache-tomcat-10.0.27
Using CATALINA_HOME:   /opt/apache-tomcat-10.0.27
Using CATALINA_TMPDIR: /opt/apache-tomcat-10.0.27/temp
Using JRE_HOME:        /usr/lib64/graalvm/graalvm-java21
Using CLASSPATH:       /opt/apache-tomcat-10.0.27/bin/bootstrap.jar:/opt/apache-tomcat-10.0.27/bin/tomcat-juli.jar
Using CATALINA_OPTS:   
Tomcat started.
[opc@jms-instance-2 tomcat-native-jp]$ time curl http://localhost:8080/springTomcat/greeting
Hello Spring Framework World

real    0m0.273s
user    0m0.004s
sys     0m0.004s
```