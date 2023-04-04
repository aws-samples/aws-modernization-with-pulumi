+++
title = "1.1 Creating a New Project"
chapter = false
weight = 1
+++


Infrastructure in Pulumi is organized into projects. Each project is a single program that, when run, declares the desired infrastructure for Pulumi to manage.

## Step 1 &mdash; Create a Directory

Each Pulumi project lives in its own directory. Create one now and change into it:

```bash
mkdir iac-workshop
cd iac-workshop
```

{{% notice note %}}
Pulumi will use the directory name as your project name by default. You can change this during the project initiation process, but we'll stick with the default for now.
{{% /notice %}}

## Step 2 &mdash; Initialize Your Project

A Pulumi project is just a directory with some files in it. It's possible to create a new one by hand, but  `pulumi new` command automates the process:

```bash
pulumi new typescript -y
```

This will print output similar to the following with a bit more information and status as it goes:

```text
Created project 'iac-workshop'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
Created stack 'dev'

Installing dependencies...
```

This output will be followed by some `npm` output. Once that's done, the command has created all the files we need, initialized a new stack named `dev` (an instance of our project), and installed the needed package dependencies from npmjs.

## Step 3 &mdash; Inspect Your New Project

Our project is comprised of multiple files:

* **`Pulumi.yaml`**: your project's metadata, containing its name and language
* **`index.ts`**: your program's main entrypoint file
* **`node_modules/`**: your program's npm dependencies
* **`package.json` and `package-lock.json`**: your project's npm dependency information
* **`tsconfig.json`**: your project's TypeScript configuration

Run `cat index.ts` to see the contents of your project's empty program:

```typescript
import * as pulumi from "@pulumi/pulumi";
```

Feel free to explore the other files, although we won't be editing any of them by hand.
