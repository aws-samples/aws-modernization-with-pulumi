+++
title = "1.1 Creating a New Project"
chapter = false
weight = 1
+++


Infrastructure in Pulumi is organized into projects. Each project is a single program that, when run, declares the desired infrastructure for Pulumi to manage.

## Step 1 &mdash; Create a Directory

Each Pulumi project lives in its own directory. Create one now and change into it:

```bash
mkdir iac-workshop-webservers
cd iac-workshop-webservers
```

> Pulumi will use the directory name as your project name by default. To create an independent project, simply name the directory differently.

## Step 2 &mdash; Initialize Your Project

A Pulumi project is just a directory with some files in it. It's possible for you to create a new one by hand. The `pulumi new` command, however, automates the process:

```bash
pulumi new java
```

This will print output similar to the following with a bit more information and status as it goes:

```
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

project name: (iac-workshop-webservers)
project description: (A minimal Java Pulumi program with Maven builds)
Created project 'iac-workshop-webservers'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
stack name: (dev)
Created stack 'dev'

Your new project is ready to go! ✨

To perform an initial deployment, run 'pulumi up'
```

This command initializes a new Pulumi stack named `dev` (an instance of our project) and generates a [Maven](https://maven.apache.org/) project template. 
You can also use [Gradle](https://gradle.org/) build tool if you prefer by simply typing
```bash
pulumi new java-gradle -y
```

## Step 3 &mdash; Inspect Your New Project

Our project is composed of multiple files:

* **`src/main/java/myproject/App.java`**: your program's main entrypoint file
* **`pom.xml`**: Maven xml file describing your project
* **`Pulumi.yaml`**: your project's metadata, containing its name and language

Run `cat src/main/java/myproject/App.java` to see the contents of your project's empty program:

```java
package myproject;

import com.pulumi.Pulumi;
import com.pulumi.core.Output;

public class App {
    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            ctx.export("exampleOutput", Output.of("example"));
        });
    }
}
```

## Step 4 &mdash; Add AWS Dependencies

Before we continue, we need to include AWS dependencies.
You can do it by editing your `pom.xml` file. Find the `dependencies` section and add the following entries:
```xml
<dependency>
    <groupId>com.pulumi</groupId>
    <artifactId>aws</artifactId>
    <version>5.3.0</version>
</dependency>
<dependency>
    <groupId>com.pulumi</groupId>
    <artifactId>aws-native</artifactId>
    <version>0.16.1</version>
</dependency>
```

Also, ensure that the `properties` section references Java 17 as we will be using constructs available on newer versions.

Your `pom.xml` file should look like the following:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.pulumi</groupId>
    <artifactId>iac-workshop-webserver</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <encoding>UTF-8</encoding>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <maven.compiler.release>17</maven.compiler.release>
        <mainClass>myproject.App</mainClass>
        <mainArgs/>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.pulumi</groupId>
            <artifactId>pulumi</artifactId>
            <version>0.1.0</version>
        </dependency>
        <dependency>
            <groupId>com.pulumi</groupId>
            <artifactId>aws</artifactId>
            <version>5.3.0</version>
        </dependency>
        <dependency>
            <groupId>com.pulumi</groupId>
            <artifactId>aws-native</artifactId>
            <version>0.16.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.2.2</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <mainClass>${mainClass}</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <mainClass>${mainClass}</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-my-jar-with-dependencies</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.0.0</version>
                <configuration>
                    <mainClass>${mainClass}</mainClass>
                    <commandlineArgs>${mainArgs}</commandlineArgs>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-wrapper-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <mavenVersion>3.8.5</mavenVersion>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

```

## Step 5 &mdash; Configure an AWS Region

Finally, set up your Pulumi configurations:

```bash
$ pulumi config set aws:region us-west-2
```

## Step 6 &mdash; Configure an AWS Profile

As with the first module, if you are using an alternative AWS profile, tell Pulumi which one to use by one of the following options:

* Using an environment variable: `export AWS_PROFILE=<profile name>`
* Using configuration: `pulumi config set aws:profile <profile name>`
