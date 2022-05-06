+++
title = "1.1 Creating a New Project"
chapter = false
weight = 1
+++


Infrastructure in Pulumi is organized into projects. Each project is a single program that, when run, declares the desired infrastructure for Pulumi to manage.

## Step 1 &mdash; Create a Directory

Each Pulumi project lives in its own directory. Create one now and change into it:

```bash
mkdir iac-workshop-ecs
cd iac-workshop-ecs
```

> Pulumi will use the directory name as your project name by default. To create an independent project, simply name the directory differently.

## Step 2 &mdash; Initialize Your Project

A Pulumi project is just a directory with some files in it. It's possible for you to create a new one by hand. The `pulumi new` command, however, automates the process:

```bash
pulumi new java -y
```

This will print output similar to the following with a bit more information and status as it goes:

```
This command will walk you through creating a new Pulumi project.

Enter a value or leave blank to accept the (default), and press <ENTER>.
Press ^C at any time to quit.

project name: (iac-workshop-ecs)
project description: (A minimal Java Pulumi program with Maven builds)
Created project 'iac-workshop'

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