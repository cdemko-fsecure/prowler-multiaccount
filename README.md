# prowler-multiaccount
WIP for automated Prowler scanning across multiple accounts in an AWS Organization

This solution is based heavily on work done by the following authors. All credit
goes to them for the original code and CloudFormation templates.
* https://github.com/bkdinoop/learnDepot/tree/master/prowler-multiaccount
* https://github.com/toniblyx/prowler/tree/master/util/org-multi-account

These files are CloudFormation files which will automate the setup of Prowler
for multi-account scanning in an AWS Organization. At a high-level, the following
are configured:
* An ECS cluster in which Fargate tasks (Prowler) will be executed
* A Lambda function which orchestrates the execution of Prowler
* A sample EventBridge event on a cron-job to automatically run the Lambda every day

In its current form, Prowler will send its findings into Security Hub in the account
that's being scanned. This requires some additional configuration in those accounts
that's not covered by this repo. For more information, see here:
https://github.com/toniblyx/prowler#security-hub-integration

If you don't want to write findings back into Security Hub, alter the Prowler command
in the Lambda function.

# How to deploy

* Enable Security Hub in each account to be scanned for the appropriate region
* Enable the Prowler integration into Security Hub
* Deploy prowler-multi-masterrole.yaml to the Organization root/master account (or any account from which `aws organizations list-accounts` can be executed successfully)
* Deploy prowler-multi-scanrole.yaml to all accounts that will be scanned (like through CloudFormation StackSets)
* Deploy prowler-multi-main.yaml into the account that will host Prowler

# Explanations

## prowler-multi-masterrole

This template file creates a role within the Organization master account which grants
permission to list accounts. This will be used by the Prowler Initiator Lambda function
to get this list.

Inputs:
* SecurityAccount - ID of the account where Prowler will reside
* ProwlerMasterRole - The name of this new role

Outputs:
* MasterRoleArn - The ARN of the new role, to be used as an input for prowler-multi-main

## prowler-multi-scanrole

This file creates the role Prowler will use (assume) in each account to be scanned.
It has ViewOnly and SecurityAudit managed policies attached, plus two others that
are recommended by Prowler devs, including the ability to write findings back 
into Security Hub

Inputs:
* SecurityAccount - ID of the account where Prowler will reside
* ProwlerCrossAccountRole - The name of this new role

Outputs:
* ProwlerCrossAccountRole - The name of the new role

## prowler-multi-main

This file does everything else. It builds the following resources:
* ECS Cluster
* Fargate Task Definition
* Lambda function
* EventBridge event
* IAM Roles for ECS execution, Prowler tasks, and Lambda
* Security group for Prowler tasks (only allows outbound HTTP/HTTPS)
* CloudWatch log groups for the Lambda and ECS tasks

Inputs:
* ProwlerImageName - Name for prowler container to be used (Example: public.ecr.aws/y6q1p2v9/prowler:latest)
* SecAuditRole - The name of the role created in prowler-multi-scanrole above
* MasterRoleArn - The ARN of the role created in prowler-multi-masterrole above
* VPC - For Fargate task networking configuration. Choose a valid VPC
* Subnet - For Fargate task networking configuration. Choose a valid subnet with outbound Internet access

# Usage Notes

* The type of Prowler scan that's run can be controlled by the content of the JSON-formatted Event sent to Lambda. If it contains a "group" key, the value will be used as part of the '-g' option to Prowler to specify which group of scans to run.
* The event can also be used to scope scans to groups of accounts based on OU, using the ListAccountsForParent API. By providing a key of "ou" and a value containing a list of OU IDs, you can control which accounts receive what kinds of scans and how often. Please note that the ListAccountsForParent API does not automatically include child OUs. See [the API reference](https://docs.aws.amazon.com/organizations/latest/APIReference/API_ListAccountsForParent.html) for more information.
* An account exclusion list is included with a dummy value in the Lambda environment variables. If you want to use/add to this list, add account numbers that shouldn't be scanned to the Environment variable as just a space-delimited string.
* The Lambda function uses the region it's deployed in for both the region ("-r") and filter ("-f") commandline arguments to Prowler. This also affects which region is used when writing findings to Security Hub (same as region, "-r" option). This works great if you operate in the same region across all accounts but may not work well otherwise. Some flexibility on configuring region may be required but is not currently a priority.

# TODO

* ~~Add template for MasterRoleArn to avoid having to manually configure this~~
* ~~Add variables into prowler-multi-main for AWS::Partition to support other AWS partitions~~
* Standardize on naming conventions across all files
* ~~Convert Fn::Join to Fn::Sub where it makes sense for readability (based on examples from [this blog post](https://theburningmonk.com/2019/05/cloudformation-protip-use-fnsub-instead-of-fnjoin/))~~
* ~~Add proper DependsOn statements where needed~~
* Abstract Prowler command line options to Lambda Env variable or Event contents for a more flexible configuration
* Rate-limiting? Need to research potential impacts of executing 100 parallel ECS tasks in this fashion
* ~~Remove outbound HTTP rule from SG. I don't think this is needed~~
* Add configuration options for region settings (see Usage Notes above)
