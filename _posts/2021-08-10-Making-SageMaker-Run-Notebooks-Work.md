---
layout: post
title: "Scheduling AWS SageMaker Notebooks 📔⏰"
subtitle: "It's all about the IAMs"
date: 2021-08-10
---

> This post covers the process I followed to get [sagemaker-run-notebooks](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#using-the-cli-provided-by-the-convenience-package) working. I hope it helps others stumbling down this path.

<!--more-->

This is a tutorial walking through my process to make the scheduling tool for notebooks on AWS SageMaker work with the SageMaker Dashboard - as described [here](https://aws.amazon.com/blogs/machine-learning/scheduling-jupyter-notebooks-on-sagemaker-ephemeral-instances/) and [here](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#using-the-cli-provided-by-the-convenience-package). (Also an excuse to write a blog post.)

*I knew almost nothing about AWS concepts like roles and policies when I first tried to follow the [instructions](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#using-the-cli-provided-by-the-convenience-package) for the `sagemaker-run-notebook` library. I still don't know much, so use the adivce below at your own risk. Also, I might have missed an essential step which would have made life easier, and the guide below unnecessary. If so, shoot me a comment and I'll add to the blog for future searchers.*

There are four main steps:
1. Setup SageMaker
2. Setup the Environment for Scheduled Execution - key point: do this outside of the SageMaker Dashboard
3. Setup the SageMaker JupyterLab extension -  key point: do this within the SageMaker Dashboard
4. Add additional permissions to Sagemaker-ExecutionRole to create scheduled runs

## Setup SageMaker
I was setting up SageMaker for the first time. Upon setup, you'll be given the option to create an execution role. I chose the default, which created an AmazonSageMaker-ExecutionRole in the IAM menu. The role needs additional permissions for scheduling to work (and for s3 access).

## Setup The Environment
The [Quickstart guide](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#using-the-cli-provided-by-the-convenience-package) tells you to setup your environment first. Don't be like me and skip this step. Do it first!

To setup the environment, enter the AWS CloudShell - the button in red below. If you don't see this button in your AWS account, your Administrator may need to grant you [access](https://docs.aws.amazon.com/cloudshell/latest/userguide/sec-auth-with-identities.html) (or you could ask them to follow this guide and take the day off 😎).

![cloud_shell_button]({{ '/assets/images/cloud_shell_button.jpg' | relative_url }})
{: style="width: 600px; max-width: 100%;"}
*Fig. 1 The AWS CloudShell Button*

From here, you can run the three commands in the Quick Start.

### 1.
```sh
$ pip3 install https://github.com/aws-samples/sagemaker-run-notebook/releases/download/v0.18.0/sagemaker_run_notebook-0.18.0.tar.gz
```
*Note the change to pip3. This was necessary for me.*

### 2.
```sh
$ run-notebook create-infrastructure
```

### 3.
```sh
$ run-notebook create-container
```

Hopefully this works for you, but it might not...

In my case, I found most errors were due to *insufficient permissions* granted to the account executing the commands. Specifically, I received an error mentioning `ROLLBACK_COMPLETE` several times. This came from AWS CloudFormation and it means the stack creation failed in some way.

If you get this error, navigate to CloudFormation, and click the stack called `sagemaker-run-notebook` with the status ROLLBACK_COMPLETE in red. Under events, you can see the events which failed. In my case the four role creations failed: `LambdaExecutionRole, BasicExecuteNotebookRole, ExecuteNotebookClientRole, ContainerBuildRole`.

The solution was to grant rights to create roles to the role that was executing the commands above. 🤷‍♂️ Yeah... confusing.

Basically, you'll see an error that `SageMaker-ExecutionRole... is not authorized to perform: ... on resource ...` in the CloudFormation events. 

![error_message_1]({{ '/assets/images/error_message_1.jpg' | relative_url }})
{: style="width: 600px; max-width: 100%;"}

To fix this you need to give the SageMaker-ExecutionRole rights to perform the necessary action on the necessary resource:
1. Navigate to IAM in the AWS search bar.
2. Click Roles on the left menu and find the SageMaker-Execution role mentioned in the error.
3. Click Attach Policies and add the necessary policies to the SageMaker-Execution role.
  - Either add an AWS managed policy or create your own.
  - In my case, adding the `AWSCloudFormationFullAccess` managed policy allowed the CloudFormation process to complete. The Quick Start guide mentions this, but I didn't realize the SageMaker-ExecutionRole needed to be granted the rights.
4. Navigate back to CloudFormation and delete the `sagemaker-run-notebook` stack with the `ROLLBACK_COMPLETE` status.
5. Jump back to the cloud shell and rerun the command which gave the `ROLLBACK_COMPLETE` error.
6. Keep adding policies until the commands work without errors. 🔨 
  - You should see a `sagemaker-run-notebook` with a green CREATE_COMPLETE status in the CloudFormation stacks menu and a `RunNotebook` function in the Lambda functions menu.

## Setup the JupyterLab Extension
1. Follow the [guide](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#in-sagemaker-studio).
2. [Optional] Add additional policies to access s3 buckets. The default SageMaker-Execution role cannot access all s3 buckets. Especially important if your notebook writes some output to a file.
3. Add policies to schedule the notebooks.
  - I found the SageMaker-ExecutionRole needed extra policies for EventsBridge and Lambda to schedule notebooks. I ended up creating a policy with full access over both of these:
  ```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "events:*",
                "lambda:*"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
  ```
 
Now you *should* be able to schedule notebooks on SageMaker. Test it with the Run Now and Schedule buttons. 

![test_notebooks]({{ '/assets/images/test_notebooks.jpg' | relative_url }})
{: style="width: 600px; max-width: 100%;"}

Also, the cron settings are a bit confusing - here's the [guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html#CronExpressions).
