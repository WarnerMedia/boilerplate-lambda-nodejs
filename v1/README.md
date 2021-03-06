# Lambda Trunk-Based Deployment (v1)

## Table of Contents

- [Folder Structure](#folder-structure)
- [Install/Initial Setup](#installinitial-setup)
- [Local Lambda Function Development](#local-lambda-function-development)
- [GitHub Branch Flow](#github-branch-flow)
- [AWS CodeBuild/CodePipeline Infrastructure](#aws-codebuildcodepipeline-infrastructure)
- [Version Management](#version-management)
- [Frequently Asked Questions (F.A.Q.)](#frequently-asked-questions)
- [TODO List](#todo-list)

# Folder Structure

This section describes the layout of the `v1` version of this project.

- [`build`](build): Everything that build processes would need to execute successfully.
    * For AWS CodeBuild, the needed BuildSpec and related shell scripts are all in this folder.
- [`env`](env): Any environment-related override files.
    * There are a number of JSON files in this folder which are used to override parameters in the various CloudFormation templates (via CodePipeline CloudFormation stages).  This is very useful for making specific changes to different environments.
    * This folder could also contain other environment configuration files for the application itself.
- [`git-hook`](git-hook): Any git hooks that are needed for local development.
    * If the [main setup script](script/cfn/setup/main.sh) is used, it will help set the hooks up.
- [`iac`](iac): Any infrastructure-as-code (IaC) templates.
    * Currently this only contains CloudFormation YAML templates.
    * The templates are categorized by AWS service to try to make finding and updated infrastructure easy.
    * This folder could also contain templates for other IaC solutions.
- [`local`](local): Tools to help with local development and testing of the function(s).
    * There are instructions for [local Lambda function development](#local-lambda-function-development).
- [`node_modules`](node_modules): Node.js modules that the Lambda(s) need to function.
    * There is a CloudFormation parameter (`CodeBuildEnableDepInstall`) in the [setup template](iac/cfn/setup/main.yaml) which can be toggled (`Yes`/`No`) to have the orchestrator automatically install the Node.js modules.
    * There is a second CloudFormation parameter (`CodeBuildCustomInstallCommand`) in the [setup template](iac/cfn/setup/main.yaml) which can be used to set the dependency install command (e.g. `npm install --only=prod`).
    * If the toggle is set to `No`, the orchestrator won't run the install command, so the dependencies will need to be installed locally and the files committed to the repository.  This will allow the orchestrator to run faster.
    * If the toggle is set to `Yes`, the orchestrator will run the install command before creating the ZIP files.  This is more convenient for the engineer, but the orchestrator will take longer to run.
- [`script`](script): General scripts that are needed for this project.
    * This folder includes the [main infrastructure setup script](script/cfn/setup/main.sh), which is the starting point of getting things running.
    * This folder could contain any other general scripts needed for this project (sans the CodeBuild related scripts, which are always in the [build](build) folder).
- [`src`](src): The source files needed for the function (sans the `node_modules` folder).
    * You could house the source code for multiple related functions in this folder if this project has multiple functions to deploy.
- [`test`](test): Any resources needed for doing testing of the application.
    * This project supports [Cucumber.js](https://cucumber.io/docs/installation/javascript/) Behavior-Driven Development (BDD) testing by default.
    * Cucumber.js is a very versatile testing solution which integrates well with CodeBuild reporting.
    * Cucumber tests can be written in [many different programming languages](https://cucumber.io/docs/installation/) including Java, Node.js, Ruby, C++, Go, PHP, Python, etc.
- [`version root`](./): All of the files that are generally designed to be at the base of the project.
    * Node.js `package-lock.json` and `package.json` files.
    * Miscellaneous files, such as the `.shellcheckrc`, and `README.md` file.

# Install/Initial Setup

---
**NOTE**

These setup instructions assume you are working with a non-prod/prod AWS account split and that you have already set up the base infrastructure: [boilerplate-aws-account-setup](https://github.com/warnermedia/boilerplate-aws-account-setup)

The `boilerplate-aws-account-setup` repository will get cloned (uisng the "Use this template" button) for each set of accounts; this allows for base infrastructure changes that are specific to that account.  The changes can be made without impacting the original boilerplate repository.

---

## Prerequisite Setup

### GitHub User Connection

1. This repository is meant to be used as a starting template, so please make use of the "Use this template" button in the GitHub console to create your own copy of this repository.
2. Since you have the base infrastructure in place, in your primary region, there should be an SSM parameter named: `/account/main/github/service/username`
3. This should be the name of the GitHub service account that was created for use with your AWS account.
4. You will want to add this service account user to your new repository (since the repository is likely private, this is required).
5. During the initial setup, the GitHub service account will need to be given admin. access so that it can create the needed webhook (no other access level can create webhooks at this time).
6. Once you try to add the service account user to your repository, someone who is on the mailing list for that service account should approve the request.

### Local AWS CLI Credentials (Needed for AWS CLI Helper Script)

If you want to use the helper script (recommended), you will need to have AWS credentials for both your AWS non-prod and AWS production accounts and have the AWS CLI installed.

1. Instructions for installing the AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
2. WarnerMedia currently only allows for local credentials to be generated from your SSO access.
3. You will need to install and configure the [gimme-aws-creds](https://github.com/WarnerMedia/gimme-aws-creds) tool.
4. You will then use that tool to generate the needed AWS CLI credentials for both the non-prod and production AWS accounts.
5. You will need to keep track of what your AWS profile names are for use with the script.

---
**NOTE**

You will need either DevOps or Admin SSO role in order to have enough permissions to set things up.

---

## Primary Setup

This can be done via the helper script or manually through CloudFormation.  The script takes more setup but then handles more things for you.  The manual CloudFormation upload requires more clicking around but less setup since you don't have to locally configure the AWS CLI, etc.

### Using the Helper Script (AWS CLI)

#### Prepare Credentials

1. Make sure that you know the names of your AWS CLI account profiles for the non-production and production accounts.
2. Retrieve a fresh set of AWS CLI credentials for your non-prod and prod AWS CLI account profiles (using the `gimme-aws-creds` tool).  These credentials generally expire every 12 hours.

#### Prepare Defaults

1. You will want to make a local checkout of the repository that you created.
2. Once you have the local checkout, switch to the `v1` folder.
3. Change to this directory: [`./script/cfn/setup/`](script/cfn/setup/)
3. Locate the following file: [`.setuprc`](script/cfn/setup/.setuprc)
4. Open this file for editing, you will see the initial values that the boilerplate was making use of.
5. Modify all of these values to line up with the values related to your accounts.  It may be useful to look at the `.setuprc` of an existing repository for your account if you are not familiar with the values that you need to fill in.

#### Execute the Script

1. Run the following script: `main.sh`
2. The `main.sh` script will use the `.setuprc` file to set its default values.
3. The script will ask you a number of questions, at which time you will have the option to change some of the default values.
4. Once you have run through all of the questions in the script, it will kick off the CloudFormation process in both the non-prod and production accounts.
5. At this point, CloudFormation will create a setup and infrastructure CodePipeline.  These CodePipelines will set up everything else that your CI/CD process needs.  The following environments will be set up:
    - `dev`
    - `int`
    - `qa`
    - `stage`
    - `prod`
6. If you need to make changes to any of the infrastructure, you can do so via CloudFormation templates located in this folder: `v1/iac/cfn`
7. You can make your changes and merge then into the `main` branch via a pull request.  From this point forward, the CodePipelines will make sure the appropriate resources are updated.

### Manually Setup (AWS Console)

If you don't want to go through the work of setting up the AWS CLI locally, you can manually upload the main setup CloudFormation template.

You may want to look at the helper script to make sure you set all the parameters correctly.

---
**NOTE**

This method is not recommended as the potential for human error or confusion is higher.

---

#### Production (Should Be Done First)

1. You will want to make a local checkout of the repository that you created.
2. Once you have the local checkout, switch to the `v1` folder.
3. Find the following CloudFormation template: [`iac/cfn/setup/main.yaml`](iac/cfn/setup/main.yaml)
4. Log into the AWS Console for your production account.
5. Go to the CloudFormation console.
6. Upload this template and then fill in all of the needed parameter values.
7. Go through all the other CloudFormation screens and then launch the stack.
8. Monitor the stack and make sure that it completes successfully.

#### Non-Prod (Should Be Done Second)

1. Switch to the `v1` folder.
2. Find the following CloudFormation template: [`iac/cfn/setup/main.yaml`](iac/cfn/setup/main.yaml)
3. Log into the AWS Console for your non-prod account.
4. Go to the CloudFormation console.
5. Upload this template and then fill in all of the needed parameter values.
6. Go through all the other CloudFormation screens and then launch the stack.
7. Monitor the stack and make sure that it completes successfully.

#### Initial Kick-Off

1. Once the stacks have created everything successfully, you will need to kick the CodeBuild off.  This can be done in one of two ways:
    - Make a commit to your `main` branch in your repository.
    - In the primary region of each account, locate a CodeBuild project in your primary region named `(your project name)-orchestrator` and then run a build (with no overrides).
2. At this point, CloudFormation will create a setup and infrastructure CodePipeline.  These CodePipelines will set up everything else that your CI/CD process needs.  The following environments will be set up:
    - `dev`
    - `int`
    - `qa`
    - `stage`
    - `prod`
3. If you need to make changes to any of the infrastructure, you can do so via CloudFormation templates located in this folder: `v1/iac/cfn`
4. You can make your changes and merge them into the `main` branch via a pull request.
5. From this point forward, the CodePipelines will make sure the appropriate resources are updated.

# Local Lambda Function Development

There are various ways to develop Lambdas locally, we have opted to stick with AWS solutions for this version of the boilerplate.  Because of this, we will be leveraging the SAM CLI for doing local development.

## Prerequisites

Your local system will need to have the following installed:

1. [docker](https://docs.docker.com/get-docker/)
2. [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)

---
**NOTE**

For Ubuntu Linux, Homebrew is no longer needed, you can install globally using: `sudo pip3 install aws-sam-cli`

If you install this way, you can upgrade with this command: `sudo pip3 install --upgrade aws-sam-cli`

---

## Running the Lambda Function

A simple helper shell script has been put in place to help run SAM CLI.

1. From within the `v1` folder, you can run the following command: [`./local/run.sh`](local/run.sh)
2. This will cause a docker instance to spin up and run the function.
3. The output of the function will be written out to this folder: `./local/output/`
4. The event that is fed into the Lambda is stored in this folder: `./local/event/`
5. There is a small CloudFormation template which SAM CLI uses to determine the runtime and handler, it is located here: [`./local/cfn/function.yaml`](local/cfn/function.yaml)

---
**NOTE**

The SAM CLI is fairly powerful and can be used to simulate a number of Lambda use-cases.  The advanced documentation is located here:

https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-command-reference.html

It should also be noted that we are only leveraging SAM for local testing.  We may eventually have additional version in this repository which have SAM and Serverless Framework deployments.

---

# GitHub Branch Flow

---
**NOTE**

Use of direct commits to the `main` branch is discouraged.  Pull requests should always be used to help give visibility to all changes that are being made.

---

## Development Flow

This repository uses a trunk-based development flow.  You can read up on trunk-based flows on this website:

[https://trunkbaseddevelopment.com](https://trunkbaseddevelopment.com)

## Commenting of Commits

The use of "Conventional Commits" is encouraged in order to help make commit message more meaningful. Details can be found on the following website:

[https://www.conventionalcommits.org](https://www.conventionalcommits.org)

## Primary Branches

1. `main`:
    - This branch is the primary branch that all bug and feature branches will be created from and merged into.
    - For the purposes of this flow, it is the "trunk" branch.
2. `dev`:
    - This is a pseudo-primary branch which should always be considered unstable.
    - It will be automatically created once the first production deployment happens and will be based off of the release that was just deployed.
    - This branch will support an "off-to-the-side" environment for testing changes that may be too risky to deploy into the main flow.  For instance, interactions with a new AWS service that are hard to test locally.
    - All `feature` or `bug-fix` branches can be merged into this branch for testing before being merged into the `main` branch.
    - **NOTE:** The `dev` branch should never be merged into the `main` branch, only `feature` or `bugfix` branches should ever be merged into `main`.
3. `.*hotfix.*`:
    - Branches with the keyword `hotfix` anywhere in the middle of the name will temporarily override the main flow, allowing for a specific hotfix to get pushed through the flow.
    - Once a `hotfix` branch has been deployed to fix the problem, the changes can be copied/cherry-picked back into the `main` branch for deployment with the next full release.
    - All `hotfix` branches should be considered temporary and can be deleted once merged into the `main` branch.
4. `feature`/`bugfix` branches:
    - These branches will be created from the `main` branch.
    - Engineers will use their `feature`/`bugfix` branch for local development.
    - Feature branch names typically take the form of `f/(ticket number)/(short description)`.  For example, `f/ABC-123/update-service-resources`
    - Bug fix branch names typically take the form of `b/(ticket number)/(short description)`.  For example, `b/ABC-123/correct-service-variable`
    - Once a change is deemed ready locally, a pull request should be used to get it merged into the `main` branch.
    - An optional step would be to merge your feature branch into the `dev` branch first to test in the `dev` environment.
    - If you do merge your `feature`/`bugfix` branch into the `dev` branch for testing, once you verify things are good, then you would merge your `feature`/`bugfix` branch into the `main` branch via a pull request.
    - **NOTE:** The `dev` branch should never be merged into the `main` branch, only `feature`/`bugfix` branches should ever be merged into `main`.
    - All `feature`/`bugfix` branches should be considered temporary and can be deleted once merged into the `main` branch.  The pull request will keep the details of what was merged.

## General Flow

### Local Development

1. An engineer would create a `feature`/`bugfix` branch from the local checkout of the local, current `main` branch.
2. The engineer would then make their changes and do in-depth local testing.
3. The engineer should write any needed application/infrastructure tests related to their changes.
4. If this feature isn't fully functional, it is good practice to wrap it in a feature flag so that it can be disabled until it is ready.
5. Once things look good locally, the engineer would push the branch to GitHub.

### Non-Prod Deployment

1. In GitHub, a pull request will be created to the `main` branch.
2. A peer review should be done of the pull request by at least one other engineer.
3. Once the pull request is approved, it will be merged into the `main` branch.
4. The changes will be deployed to the initial integration (`int`) environment.
5. If things are approved in the `int` environment, then a manual approval in the CodePipeline will promote the changes to the Quality Assurance (`qa`) environment.
6. At this point a GitHub pre-release tag will be created and a link to a GitHub changelog will be added to the notes of the pre-release.
7. Once things are approved in the `qa` environment, a manual approval in the CodePipeline will promote the change to the Staging (`stage`) environment.
8. The `stage` environment will allow for one last review before things go to the production (`prod`) environment.

### Production Deployment

1. Now that things are ready for a production deployment, a time and date should be set for the production deployment and all deployment documentation processes and notifications should be done.
2. At the desired time, a manual approval in the `stage` CodePipeline will then promote the changes to the `prod` environment in the production AWS account.
3. The changes will then be deployed to the production account.
4. Once things have been successfully deployed to production, the `dev` branch will be overwritten with the release that was just deployed to production.

## Optional Flow Steps

In the above flow, there is one additional environment that changes can be pushed to if they are high-risk.  Here are the details:

1. Once an engineer is ready to get their changes merged into the `main` branch, they can optionally first choose to create a pull request into the `dev` branch.
2. Once their branch is merged into the `dev` branch, the changes will be deployed to an "off-to-the-side" environment where the changes can be tested and verified without blocking the main flow.
3. Once the changes look good on the `dev` environment, the engineer can create a separate pull request for their branch to the `main` branch.
4. The `dev` branch is wiped out and replaced whenever there is a production deployment.  It is replaced with the release SHA that was just deployed to the production environment.  This prevents the `dev` environment from becoming a "junk drawer" of failed test branches.
5. The `dev` branch and environment should never really be considered a "stable" environment.
6. **NOTE:** The `dev` branch should never be merged into the `main` branch, only `feature`/`bugfix` branches should ever be merged into the `main` branch.

# AWS CodeBuild/CodePipeline Infrastructure

This project uses AWS CodeBuild and AWS CodePipeline to get your Lambda deployed.  Here we will outline the different components of the deployment flow.

## CodeBuild Orchestrator

- The orchestrator is a CodeBuild project which is triggered by a GitHub Webhook.
- This CodeBuild project can be found in the primary region where you set up the infrastructure and have a name that follows this pattern: `(your project name)-orchestrator`
- The orchestrator will examine the changes that were just committed and determine the type of change which was just made.
- The changes will be packaged into different ZIP archives and then deploy them to archive S3 bucket.
- The appropriate CodePipelines will then be triggered based on the type of change that was committed.
- The orchestrator creates different ZIP archives, the contents of those ZIP archives are managed by `*.list` files which are located here: `v1/env/cfn/codebuild/orchestrator/`

## Project Infrastructure CodePipelines

- There are two project infrastructure CodePipelines, the setup CodePipeline and the Infrastructure CodePipeline.

## Setup CodePipeline

- When the initial setup CloudFormation template runs, it creates a setup CodePipeline.
- This CodePipeline will get triggered within a minute of the first successful CodeBuild orchestrator run.
- This CodePipeline is very simple, it's only purpose is to create and maintain the infrastructure CodePipeline.
- This CodePipeline may feel like an extra step, but it is there so that project infrastructure changes can be made easily.
- Updates to the CodePipeline should be rare.
- **NOTE:** If changes need to be made to the setup CodePipeline, then the the [main setup template](iac/cfn/setup/main.yaml) will need to be edited and the changes manually run from the AWS CloudFormation console.

## Infrastructure CodePipeline

- This CodePipeline is initially created and maintained by the setup CodePipeline.
- The template that manages this CodePipeline is located here: `iac/cfn/codepipeline/infrastructure.yaml`
- Any environment parameters overrides can be set in the JSON files located in this folder: `env/cfn/codepipeline/infrastructure`
- This CodePipeline will create/maintain all of the base infrastructure for this project.  Some examples are:
    * General IAM Roles, such as roles for the testing CodeBuild projects, deployment CodePipelines, etc.
    * General CodeBuild projects.  For example, CodeBuild projects that the deployment CodePipelines would use for testing stages, etc.
    * Deployment CodePipelines for all of the different environments: `dev`, `int`, `qa`, `stage`, and `prod`.
- You can review the CloudFormation parameters in the [infrastructure CodePipeline](iac/cfn/codepipeline/infrastructure.yaml) template to see what options are all available.
    * For example, there is a parameter to turn on a manual approval step for the infrastructure CodePipeline; this is useful for approving changes to the production infrastructure after being verified in non-prod.
- You would use this CodePipeline to set up things that are shared by all deployment CodePipelines, or things that will rarely change.
- There is an optional CodeBuild stage that can be activated which would allow you to run custom commands, such as triggered a different IAC solution or triggering an AWS CLI command after things are all updated.
- This CodePipeline is configured to work with up to three different regions and deploy to two regions as standard functionality.
    * It is good to have your infrastructure and functions deployed in two regions for spreading load, redundancy, and disaster recovery.
    * The CodePipeline could do deployments in up to three different regions.  However, with each region you add, you also add complexity.
    * Though there is a good case to be made for running in two regions for things like disaster recovery, the case for going into more regions gets weaker (as costs will rise for maintenance, data synchronization, etc.)
    * The main reason for third region is that you could switch to a different region if one of your normal regions is going to be down for a prolonged period of time.
    * Running in two regions is fairly complex and one should make sure they have that perfected before trying to add the complexity of any additional regions.

## Deployment CodePipelines

- There is an individual CodePipeline for each environment.
- Because each environment gets a CodePipeline, this means that environments can be added or removed in a modular fashion.
- In the default flow, there are five environments:
    * `dev`: This is an optional, "off-to-the-side" non-prod environment which can be used for testing risky changes (without blocking the main flow).  It is trigger by changes to the `dev` branch.
    * `int`: This is the first non-prod environment in the main deployment flow; it is triggered when changes are made to the `main` branch (or updates to any branch with `hotfix` somewhere in the middle of the name).
    * `qa`: This is the second non-prod environment in the main deployment flow; it is triggered when there is a manual approval in the `int` CodePipeline.
    * `stage`: This is the third and final non-prod environment in the main deployment flow; it is triggered when there is a manual approval in the `qa` CodePipeline.
    * `prod`: This is the only environment in the production account and the final step in the main deployment flow; it is triggered when there is a manual approval in the `stage` CodePipeline.
- Each environment has the option to enable an application and infrastructure testing stage.  These are controlled by parameters in the CloudFormation template.
- Just like the infrastructure CodePipelines, each deployment CodePipeline is configured to work with up to three different regions and deploy to two regions as standard functionality.
    * Please see the details for the infrastructure CodePipeline to understand the reasoning for this.

# Version Management

The version of the application can be managed manually or automatically (via the deployment flow).

---
**NOTE**

Semantic versioning should be used for maintaining the version of this application.  If the same versioning method is used between applications, it is helpful for consistency.

[Semantic Versioning](https://semver.org)

Given a version number `MAJOR.MINOR.PATCH`, increment the:

- `MAJOR` version when you make incompatible API changes,
- `MINOR` version when you add functionality in a backwards compatible manner, and
- `PATCH` version when you make backwards compatible bug fixes.
- Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.

---

## Manual Version Update

Since this is a Node.js application, we will leverage the `package.json` file for managing the version number.

1. Once you have your changes ready to commit to your `bugfix`/`feature` branch, review the changes and see which semantic versioning level they match up with.
2. Open the `package.json` file.
3. Update the the `version` property in the JSON object to the needed value.
4. Commit this change with the rest of your changes.
5. Merge your pull request into either the `main` or `dev` branch.
6. The build process will notice that you manually updated the version number and will use your version number.
7. When your changes are promoted to the Quality Assurance (QA) environment, this version number will be used as the pre-release tag in GitHub: `https://github.com/<organization>/<repository>/releases`
8. Once the application is promoted to production, the GitHub pre-release will get promoted to a full release.

**NOTE:** It is recommended to manually update the package JSON file to ensure that thought is given to which semantic versioning level should be updated.

## Automatic Version Update

If you merge a `bugfix`/`feature` branch into the `main` or `dev` branch, but did not manually update the `package.json` file, the following will happen:

1. The build process will notice that you did not update the version number and will automatically increment the `PATCH` level of the semantic version (e.g. `1.1.2` to `1.1.3`).
2. When your changes are promoted to the Quality Assurance (QA) environment, this version number will be used as the pre-release tag in GitHub: `https://github.com/<organization>/<repository>/releases`
3. Once the application is promoted to production, the GitHub pre-release will get promoted to a full release.

# Frequently Asked Questions

1. I don't like all these environments, are they all required?
    - No, not all of the environments are required.  That is why each environment has its own deployment CodePipeline, the idea is that you could remove environments.
    - If you look through the CloudFormation templates, you will notice that there are configuration options for environment setup, such as environment names, which one is the initial environment, etc.
    - At minimum, you would need to have three environments, one primary non-prod environment, one prod environment, and then one off-to-the-side environment for testing risky changes.
    - **NOTE:** In-depth full testing of removing environments has not been done, if you try to remove/refactor environments and run into major bugs, please report them so that the boilerplate can be improved.
2. Why are there all these testing stages, and why are they failing for my project?
    - Each CodePipeline has the option to test your application and AWS infrastructure using a CodeBuild testing phase (which, by default, uses [Cucumber.js](https://cucumber.io/docs/installation/javascript/)).
    - If your test phases are failing (most-likely due to the boilerplate code being replaced with your actual code), then you have the following options:
        * Update the Cucumber.js test to tests that will work with your application and AWS infrastructure (recommended).
        * Replace the Cucumber.js tests with your own test suite/testing solution (this is also fine/recommended).
        * Shut off the testing phases in all environments (which is supported by CloudFormation parameters) and figure out the whole testing thing later (not recommended).
    - Though you can use another product other than Cucumber/Cucumber.js, please note the reporting will break, unless your testing suite outputs to one of the [supported formats](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html#build-spec.reports.file-format).
3. Can I turn on testing stages for only certain environments?
    - Yes, the application and infrastructure testing stages can be turned on or off in any environment via parameters in the CloudFormation templates.  You can use the [environment JSON files](env/cfn/codepipeline/deploy) for this purpose.
4. When making changes to the infrastructure CodePipeline, I don't like the fact that the changes are deployed out to the production account immediately, I would like to approve changes; is there a way to do that?
    - Yes, you can activate a manual approval step for the infrastructure CodePipeline using the [environment JSON files](env/cfn/codepipeline/infrastructure) to override the default parameter value for the `EnableManualApproval` CloudFormation parameter.
    - When things are first set up, we want all the infrastructure to get established in both the non-prod and prod accounts.  But as your product matures, you may not want that.  You may want to approve infrastructure changes in production (or even non-prod), so this feature was added.
5. I have added or removed files, but the CodePipelines cannot find them.  How do I fix this?
    - The orchestrator CodeBuild uses some [`*.list`](env/cfn/codebuild/orchestrator) files to know which things it should include and exclude from the various artifact ZIP archives (allowing you to control when, say, the infrastructure CodePipeline is triggered).
    - Make sure that your [`*.list`](env/cfn/codebuild/orchestrator) files are up-to-date with your latest changes, and then get these changes merged in via a pull request, this will trigger a new orchestrator run and the ZIP archive files will be updated appropriately.
    - If you look at the build logs from the orchestrator CodeBuild, you will see which files are being included into the different ZIP archives.
6. When the automatic GitHub patch occurs, it is triggering another build, which starts a build loop; how do I fix this?
    - When you first set up the project, the orchestrator is given the ID of a GitHub user that it adds to the GitHub WebHook, any changes coming from this user are ignored.
    - If the wrong GitHub ID is provided for the GitHub WebHook, then it will get triggered when you do not want it to be triggered.
    - The GitHub ID needs to match the ID of the GitHub service account that is being used by the deployment flow.  This username should be in the following SSM parameter: `/account/main/github/service/username`
    - You can find the GitHub ID by using this HTTPS URL: `https://api.github.com/users/<your github user name>`
    - **NOTE:** If you notice a build loop happening, you want to make sure to disable the CodePipeline that is in the loop; otherwise, if left to run, you could run up a significant CodeBuild/CodePipeline bill.
7. How can I determine if a feature of the deployment flow is configurable?
    - There are many configurable aspects of the CloudFormation templates, the best thing to do is open the template you are interested in and see what parameters the template already has available.
8. Can I make manual updates to the function(s)?
    - Though manual updates can be made to the function(s) via the AWS Console, this is discouraged as these changes could be forgotten (since they would not be in git/source control).
    - Manual changes will usually be overwritten when the next changes are deployed from the CodePipeline/CloudFormation stack.
    - The only time manual changes might make sense is to confirm something like a permissions fix before adding it to git/source control.

# TODO List

1. Add a feature to have the orchestrator optionally do the `npm install`, instead of having to add the modules to the repository.
    * This will have the downside of slower orchestrator builds.
2. Add diagrams to documentation.
3. Fix bug where the `package.json` file is incremented when only documentation files are updated.