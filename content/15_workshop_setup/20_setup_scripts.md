+++
title = "Setup Scripts"
chapter = false
weight = 20
+++

{{% notice note %}}
For these steps, you need to be back in the AWS Cloud9 IDE.
{{% /notice %}}

### 1. Clone the workshop scripts

First, we need to get some scripts that will automate the workshop setup.

```
cd ~
git clone https://github.com/dt-demos/modernize-workshop-setup.git
```

### 2. Install workshop tools

Run these commands to call the script that will upgrade the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) to version 2 and install a JSON parsing utility called [jq](https://stedolan.github.io/jq/).

```
cd ~/modernize-workshop-setup/aws
./installWorkshopTools.sh 
```

When this is complete, you should see a message like this.  Verify you see aws-cli/2.x.y in the output

```
=============================================
Installing workshop tools COMPLETE
jq  --version: jq-1.5
aws --version: aws-cli/2.0.17 Python/3.7.3 Linux/4.14.177-107.254.amzn1.x86_64 botocore/2.0.0dev21
=============================================
```

{{% notice warning %}}
If the version is not 2.x, **DO NOT PROCEED**. Go back and confirm the steps on this page.
{{% /notice %}}
