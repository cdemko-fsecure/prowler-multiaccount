# prowler-multiaccount
WIP for automated Prowler scanning across multiple accounts in an AWS Organization

All credit to the following authors, all I did was smash these together:
* https://github.com/toniblyx/prowler/tree/master/util/org-multi-account
* https://github.com/bkdinoop/learnDepot/tree/master/prowler-multiaccount

These files are CloudFormation files which will automate the setup of Prowler
for multi-account scanning in an AWS Organization. At a high-level, the following
are configured:
* An ECS cluster in which Fargate tasks (Prowler) will be executed
* A Lambda function which orchestrates the execution of Prowler
* A sample EventBridge event on a cron-job to automatically run the Lambda every day

In its current form, Prowler will send its findings into SecurityHub in the account
that's being scanned. This requires some additional configuration in those accounts
that's not covered by this repo. For more information, see here:
https://github.com/toniblyx/prowler#security-hub-integration

If you don't want to write findings back into Security Hub, alter the Prowler command
in the Lambda function.

# How to deploy

* Enable SecurityHub in each account to be scanned for the appropriate region
* Enable the Prowler integration into SecurityHub
* Create a role in the organization master account which can be assumed by a lambda function in the account where Prowler is deployed. The role needs permissions for `organizations:listAccounts` to list all available accounts in the Organization. (The ARN of this role will be needed when deploying prowler-multi-main, as the MasterRoleArn)
* Deploy prowler-multi-account.yaml to all accounts that will be scanned
* Deploy prowler-multi-main.yaml into the account that will host Prowler

# Explanations

## prowler-multi-scanrole

This file creates the role Prowler will use (assume) in each account to be scanned.
It has ViewOnly and SecurityAudit managed policies attached, plus two others that
are recommended by Prowler devs, including the ability to write findings back 
into Security Hub

Inputs:
* SecurityAccount - ID of the account where Prowler will reside
* ProwlerExecRole - The name of the role Prowler uses to assume this new role you're creating
* ProwlerCrossAccountRole - The name of this new role

## prowler-multi-main

This file does everything else. It builds the following resources:
* ECS Cluster
* Fargate Task Definition
* Lambda function
* EventBridge event
* CloudWatch log groups for the Lambda and ECS tasks
* IAM Roles for ECS execution, Prowler tasks, and Lambda
* Security group for Prowler tasks (only allows outbound HTTP/HTTPS)

Inputs:
* ProwlerImageName - Name for prowler container to be used (Example: public.ecr.aws/y6q1p2v9/prowler:latest)
* SecAuditRole - The name of the role created in prowler-multi-account above (ProwlerScanRole)
* MasterRoleArn - ARN of a role the Lambda function can assume which allows listing of Organization accounts
* VPC - For Fargate task networking configuration. Choose a valid VPC
* Subnet - For Fargate task networking configuration. Choose a valid subnet with outbound Internet access

# Usage Notes

* Currently, the type of Prowler scan that's run can be controlled by the content of the JSON-formatted Event sent to Lambda. If it contains a "group" key, the value will be used as part of the '-g' option to Prowler to specify which group of scans to run.
* An account exclusion list is included with a dummy value in the Lambda environment variables. If you want to use/add to this list, add account numbers that shouldn't be scanned to the Enviroment variable as just a space-delimited string.

# TODO

* Add template for MasterRoleArn to avoid having to manually configure this
* ~~Add variables into prowler-multi-main for AWS::Partition to support other AWS partitions~~
* Standardize on naming conventions across all files
* Convert Fn::Join to Fn::Sub where it makes sense for readability (based on examples from [this blog post](https://theburningmonk.com/2019/05/cloudformation-protip-use-fnsub-instead-of-fnjoin/))
* Add proper DependsOn statements where needed
* Abstract Prowler command line options to Lambda Env variable or Event contents for a more flexible configuration
* Rate-limiting? Need to research potential impacts of executing 100 parallel ECS tasks in this fashion
