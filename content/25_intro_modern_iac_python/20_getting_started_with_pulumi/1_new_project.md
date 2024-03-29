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
Pulumi will use the directory name as your project name by default. To create an independent project, simply name the directory differently.
{{% /notice %}}

## Step 2 &mdash; Initialize Your Project

A Pulumi project is just a directory with some files in it. It's possible for you to create a new one by hand. The `pulumi new` command, however, automates the process:

```bash
pulumi new python -y
```

This will print output similar to the following with a bit more information and status as it goes:

```text
Created project 'iac-workshop'
Created stack 'dev'
Saved config
Installing dependencies...
Finished installing dependencies

Your new project is ready to go!
```

This command has created all the files we need, initialized a new stack named `dev` (an instance of our project), and installed the needed package dependencies from PyPi.

## Step 3 &mdash; Inspect Your New Project

Our project is comprised of multiple files:

* **`__main__.py`**: your program's main entrypoint file
* **`requirements.txt`**: your project's Python dependency information
* **`Pulumi.yaml`**: your project's metadata, containing its name and language
* **`venv`**: a [virtualenv](https://pypi.org/project/virtualenv/) for your project

Run `cat __main__.py` to see the contents of your project's empty program:

```python
"""A Python Pulumi program"""

import pulumi
```

Feel free to explore the other files, although we won't be editing any of them by hand aside from `requirements.txt`.
