---
title: "Update IAM settings for your Workspace"
chapter: false
weight: 19
---

{{% notice info %}}
Cloud9 normally manages IAM credentials dynamically. This isn't currently compatible with
the some AWS services authentication, so we will disable it and rely on the IAM role instead.

{{% /notice %}}

- Return to your workspace and click the gear icon (in top right corner), or click to open a new tab and choose "Open Preferences"
- Select **AWS SETTINGS**
- Turn off **AWS managed temporary credentials**
- Close the Preferences tab
![c9disableiam](/images/c9disableiam.png)


Let's run the command below, the following actions will take place as we do that: 

:small_blue_diamond: Install jq- jq is a command-line tool for parsing JSON

:small_blue_diamond: Ensure temporary credentials aren’t already in place.

:small_blue_diamond: Remove any existing credentials file.

:small_blue_diamond: Set the region to work with our desired region.

:small_blue_diamond: Validate that our IAM role is valid. 

```sh
sudo yum -y install jq
rm -vf ${HOME}/.aws/credentials
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || echo AWS_REGION is not set
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | 
tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
aws sts get-caller-identity --query Arn | grep Pulumi-Workshop-Admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
```

If the IAM role is not valid, <span style="color: red;">**DO NOT PROCEED**</span>. Go back and confirm the steps on this page. 
