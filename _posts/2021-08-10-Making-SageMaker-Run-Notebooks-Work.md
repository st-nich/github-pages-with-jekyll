---
layout: post
title: "Scheduling SageMaker Notebooks 📔⏰"
subtitle: "It's all about the IAMs"
date: 2021-08-10
---

> This post covers the process I followed to get [sagemaker-run-notebooks](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#using-the-cli-provided-by-the-convenience-package) working. I hope it helps others stumbling down this path.

<!--more-->

This is a tutorial walking through my process to make the `sagemaker-run-notebook` scheduling work with the SageMaker Dashboard - as described [here](https://aws.amazon.com/blogs/machine-learning/scheduling-jupyter-notebooks-on-sagemaker-ephemeral-instances/) and [here](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#using-the-cli-provided-by-the-convenience-package). (Also an excuse to write a blog post.)

*I knew almost nothing about AWS concepts like roles and policies when I first tried this. I still don't know much, so use the adivce below at your own risk. Also, I might have missed an essential step which would have made life easier, and the guide below unnecessary. If so, shoot me a comment and I'll add to the blog for future searchers.*

There are four main steps:
1. Setup SageMaker
2. Setup the Environment for Scheduled Execution - key point: do this outside of the SageMaker Dashboard
3. Setup the SageMaker JupyterLab extension -  key point: do this within the SageMaker Dashboard
4. Add additional permissions to Sagemaker-ExecutionRole to create scheduled runs

## Setup SageMaker
I was setting up SageMaker for the first time. Upon setup, you'll be given the option to create an execution role. I chose the default, which created an AmazonSageMaker-ExecutionRole in the IAM menu. The role needs additional permissions for scheduling to work (and for s3 access). Those are added below.

## Setup The Environment
The [Quick Start guide](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#using-the-cli-provided-by-the-convenience-package) tells you to setup your environment first. Don't be like me and skip this step. Do it first!

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

To fix this you need to give the SageMaker-ExecutionRole rights to perform the necessary action on the necessary resource.
1. Navigate to IAM in the AWS search bar.
2. Click Roles on the left menu and find the SageMaker-Execution role mentioned in the error.
3. Click Attach Policies and add the necessary policies to the SageMaker-Execution role.
  - Either add an AWS managed policy or create your own.
  - In my case, adding the `AWSCloudFormationFullAccess` managed policy allowed the CloudFormation process to complete. The Quick Start guide mentions this, but I didn't realize the SageMaker-ExecutionRole needs the policy (rather than the User which I was logged in as).
4. Navigate back to CloudFormation and delete the `sagemaker-run-notebook` stack with the `ROLLBACK_COMPLETE` status.
5. Jump back to the cloud shell and re-run the command which gave the `ROLLBACK_COMPLETE` error.
6. Keep adding policies until the commands work without errors. 🔨 
  - You should see 1) a `sagemaker-run-notebook` with a green CREATE_COMPLETE status in the CloudFormation stacks menu and 2) a `RunNotebook` function in the Lambda functions menu.

## Setup the JupyterLab Extension
Follow the [guide](https://github.com/aws-samples/sagemaker-run-notebook/blob/master/QuickStart.md#in-sagemaker-studio).

## Test the Extension
### Run Now
Find your favorite notebook, enter and try the Run Now button. If it works, you should see a green box below the Run Now button. If it fails, an error will show (likely saying that the ExecutionRole does not have authorization to do something). 

My notebook wrote to an s3 bucket, so I needed to add additional permissions to the ExecutionRole to access the bucket.
3. [Optional] Add additional policies to access s3 buckets. The default SageMaker-ExecutionRole cannot access all s3 buckets. Especially important if your notebook pulls in data or writes to an s3 location via boto.

### Create Schedule
Try to create a schedule. To do this you'll need to fill in the Image, Instance, Rule Name, and Pattern fields.
- Image: `notebook-runner`
- Instance: `ml.t3.medium` or whatever you choose
- Rule Name: Make up a name. 
- Schedule: A [cron schedule](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html#CronExpressions) for the rule.

I found the SageMaker-ExecutionRole needed extra policies for EventsBridge and Lambda to schedule notebooks and ended up creating a policy with full access over both of these:
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

Now you *should* be able to schedule notebooks on SageMaker. Test it out Schedule buttons. 

![test_notebooks]({{ '/assets/images/test_notebooks.jpg' | relative_url }})
{: style="width: 600px; max-width: 100%;"}
