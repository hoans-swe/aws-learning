---
title: CI/CD on AWS
markmap:
  colorFreezeLevel: 2
  initialExpandLevel: 2
---

# CI/CD on AWS

## Introduction

### Why CI/CD

- So far: create resources **by hand** (console), call AWS **programmatically** (CLI, SDK), deploy with **Elastic Beanstalk**.
- Every manual step is a chance to **make a mistake** → automate the path from code to AWS.
- Goal: code lives **in a repository** and is deployed to AWS:
    - **Automatically**, the same way every time.
    - **Tested** before it is deployed.
    - Through several **stages / environments** (dev → test → staging → prod).
    - With a **manual approval** where needed (e.g. before prod).

### Continuous Integration (CI)

- ![Continuous integration loop: the developer pushes code to the orders-api repository several times a day, the build server (CodeBuild, Jenkins) fetches each new commit, builds and runs the tests, and sends the results back within minutes: passed means safe to merge, failed means fix it while the change is fresh](../images/CICDContinuousIntegration.png)
- Developers **push code often** to a shared repository (CodeCommit, GitHub, Bitbucket…).
- A **build / test server** checks every push automatically (CodeBuild, Jenkins…).
- The developer gets **fast feedback**: which tests and checks passed or failed.
- Benefits:
    - **Find bugs early** → small changes are easy to fix.
    - **Deliver faster**, because the code is always tested.
    - Deploy often; developers are not blocked by big merges.
- Output: a **tested build artifact** (zip, container image…) ready to deploy.

### Continuous Delivery (CD)

- ![Continuous delivery: developer push → repository → build and test produce artifact v2 → every passing build is deployed, replacing v1 with v2 on each application server; continuous delivery puts a manual approval before prod, continuous deployment goes to prod automatically](../images/CICDContinuousDelivery.png)
- Software can be **released reliably whenever needed**.
- Deployments are **frequent and quick**: from "one release every 3 months" to "many releases a day".
- In practice this means **automated deployment** (CodeDeploy, Jenkins, Spinnaker…).
- **Continuous delivery** vs **continuous deployment** (not on the slides):
    - Delivery: every build is *ready* for prod; a **person approves** the prod release.
    - Deployment: every build that passes the tests goes to prod **automatically**.

### CI/CD Services on AWS

- ![CI/CD on AWS: developers push to CodeCommit (or GitHub / Bitbucket), CodeBuild (or Jenkins) builds and tests, then CodeDeploy deploys to EC2 instances, on-premises servers, Lambda or ECS, or Elastic Beanstalk deploys and provisions; CodePipeline orchestrates all the stages](../images/CICDOnAWS.png)
- **Code** → **AWS CodeCommit**: managed Git repositories (details below).
    - Or any 3rd-party repository: **GitHub**, **Bitbucket**…
    - CodeCommit was closed to new customers in July 2024 and reopened in November 2025.
- **Build + test** → **AWS CodeBuild**: managed build containers, pay per build minute.
    - Or any 3rd-party CI server, e.g. **Jenkins**.
- **Deploy + provision** → **AWS Elastic Beanstalk**: deploys the code **and** creates the environment.
- **Deploy only** → **AWS CodeDeploy**: to **EC2** instances, **on-premises** servers, **Lambda** functions and **ECS** services you already run.
- **Orchestrate** → **AWS CodePipeline**: chains source → build → test → (approval) → deploy, triggered by each push.

## AWS CodeCommit

### Version Control & Git

- **Version control**: see every change made to the code over time, and **roll back** if needed.
- **Git** is the standard version control system.
    - Each developer has a **local copy** of the repository, synced with a **central online repository**.
- Benefits of a central repository:
    - **Collaborate** with other developers.
    - The code is **backed up** somewhere.
    - Every change is **visible and auditable** (who changed what, when).
- Hosted Git options: **GitHub**, **GitLab**, **Bitbucket**… and **AWS CodeCommit**.

### Features

- **Private Git repositories**, fully managed and highly available.
- **No limit on repository size**: it scales seamlessly.
    - Per-file limits: 6 MB in the console / API / CLI, 2 GB per Git blob.
- Code stays **in your AWS account** → easier security and compliance.
- **Security** built in: encryption, IAM access control (see below).
- **Integrates** with CodeBuild, CodePipeline, Jenkins and other CI tools.
- Pull requests with **approval rules** for code review.

### Security

- ![CodeCommit security: a developer (IAM user) runs git push over SSH or HTTPS, an IAM policy check allows GitPush, and the orders-api repository in account 111122223333 is encrypted at rest with a KMS key; a partner developer in account 444455556666 calls STS AssumeRole, gets temporary credentials for an IAM role in the repo account and pushes as that role](../images/CodeCommitSecurity.png)
- Interactions use **standard Git** commands.
- **Authentication** (who you are):
    - **SSH keys**: IAM users upload their SSH public key in the **IAM console**.
    - **HTTPS**: **Git credentials** (user name + password) generated in IAM for an IAM user, or the **AWS CLI credential helper**.
    - **git-remote-codecommit** (`codecommit://` URLs): for **roles, federated / SSO** users and other **temporary credentials** (not on the slides).
- **Authorization** (what you can do): **IAM policies** on users / roles, e.g. allow `codecommit:GitPull` and `codecommit:GitPush` on one repository ARN.
- **Encryption**:
    - **At rest**: automatic, with **AWS KMS** (AWS managed key or your own customer managed key).
    - **In transit**: only **HTTPS** or **SSH**, both encrypted.
- **Cross-account access**:
    - **Never share** SSH keys or AWS credentials.
    - Create an **IAM role** in the repository's account and let the other account assume it with **AWS STS `AssumeRole`**.

### CodeCommit vs GitHub

- ![CodeCommit vs GitHub: both support pull requests, CodeBuild / CodePipeline integration and Git over SSH and HTTPS; CodeCommit users are IAM users and roles, it is hosted by AWS in your account with a minimal UI and KMS encryption; GitHub users are GitHub accounts, hosted on github.com or self-hosted with Enterprise Server, with a full-featured UI](../images/CodeCommitVsGitHub.png)
- **Both**: pull requests / code review, CodeBuild integration, SSH + HTTPS.
- **Users**: CodeCommit → **IAM users & roles**; GitHub → **GitHub accounts**.
- **Hosting**: CodeCommit → **managed by AWS**; GitHub → github.com, or **GitHub Enterprise Server** on your own servers.
- **UI**: CodeCommit → **minimal**; GitHub → **full featured**.

## AWS CodePipeline

### Overview

- **Visual workflow** that **orchestrates** the whole CI/CD process (where it fits: see "CI/CD Services on AWS" above).
- ![CodePipeline stages: pipeline orders-api runs Source (CodeCommit main branch) → Build (CodeBuild) → Test with two parallel actions (CodeBuild unit tests and Device Farm mobile UI tests) → manual Approval → Deploy (CodeDeploy to prod EC2); below, the example providers for each action category](../images/CodePipelineStages.png)
- A pipeline is made of **stages**, run **in order**, e.g. Build → Test → Deploy → Load testing.
    - Each stage has one or more **actions**, run **in sequence and / or in parallel**.
    - A **manual approval** action can be added to any stage.
- Action categories and example providers:
    - **Source**: CodeCommit, ECR, S3, GitHub, GitLab, Bitbucket (GitHub / GitLab / Bitbucket through a connection).
    - **Build**: CodeBuild, Jenkins, CloudBees, TeamCity.
    - **Test**: CodeBuild, AWS Device Farm, 3rd-party tools.
    - **Deploy**: CodeDeploy, Elastic Beanstalk, CloudFormation, ECS, S3…
    - **Invoke**: Lambda, Step Functions.
    - **Approval**: manual.

### Artifacts

- ![CodePipeline artifacts: the developer pushes to CodeCommit, which writes SourceArtifact to the S3 artifact bucket; CodeBuild reads it as input and writes BuildArtifact back to S3; CodeDeploy reads BuildArtifact and deploys to EC2 instances](../images/CodePipelineArtifacts.png)
- Each stage can create **artifacts**: its output (source code zip, build output…).
- Artifacts are stored in an **S3 bucket** (the *artifact store*) and passed as **input** to the next stage.
    - Stages never talk to each other directly: they **upload to and download from S3**.
- Artifacts are **encrypted with KMS**; the pipeline's **service role** needs access to the bucket.
- Cross-Region actions use an artifact bucket **in each Region** (not on the slides).

### Troubleshooting

- ![CodePipeline troubleshooting: a failed CodePipeline sends a state change event to an EventBridge rule, which notifies an SNS topic that emails the team and invokes a Lambda function for a chat alert; AccessDenied points to the pipeline's IAM service role, and CloudTrail logs the API calls](../images/CodePipelineTroubleshooting.png)
- **Pipeline, stage and action state changes** → events in **Amazon EventBridge** (formerly CloudWatch Events).
    - E.g. rules for **failed** pipelines or **canceled** stages → notify with SNS, run a Lambda.
- A **failed stage stops the pipeline**; the console shows which action failed and why.
- Pipeline **can't perform an action** → check the pipeline's **IAM service role** has the permissions (IAM policy).
- **AWS CloudTrail** audits the CodePipeline **API calls** (who changed or started the pipeline).

## AWS CodeBuild

### Overview

- **Fully managed CI service**: compiles code, runs tests, produces packages (zip, jar, Docker image…).
- **Serverless builds**: no build servers to manage, scales on its own → **no build queue**, builds run in parallel.
- **Pay per build minute** (compute time of each build); alternative to self-hosted **Jenkins**.
- Every build runs in a fresh **Docker container** → reproducible builds.
    - **Managed images** with ready runtimes: Java, Python, Node.js, Go, Ruby, PHP, .NET, Android…
    - **Custom image** (e.g. from ECR) for any other toolchain.
- **Sources**: CodeCommit, S3, GitHub, GitHub Enterprise, Bitbucket, GitLab.
- **Build project**: the build's settings (source, image, role, buildspec); define it in CodeBuild, or from a CodePipeline build action.

### How It Works

- ![How CodeBuild runs a build: the developer pushes to the CodeCommit repo; CodeBuild starts a container from a managed or ECR build image, takes the source and buildspec.yml, uploads artifacts to an S3 bucket, saves and restores an optional cache in S3, and sends build logs to CloudWatch Logs or S3](../images/CodeBuildHowItWorks.png)
- Source + **buildspec.yml** → container from the **build image** → runs the commands → **artifacts** to S3.
- **Cache** in S3 (optional): reuse dependencies between builds → faster builds.
- **Logs** → **CloudWatch Logs** and / or **S3**.
- Build instructions: a `buildspec.yml` in the repo, or typed into the project in the console.

### buildspec.yml

- ![buildspec.yml for an orders-api Node.js project: env sets variables, a Parameter Store value and a Secrets Manager secret; the install, pre_build, build and post_build phases run in order; afterwards artifacts (app.zip) go to S3 encrypted with KMS and the cache (node_modules) is saved to S3 for the next build](../images/CodeBuildBuildspec.png)
- Must be at the **root** of the source code (another name or path can be set in the project).
- `env`: environment variables.
    - `variables`: plain text values.
    - `parameter-store`: values from **SSM Parameter Store**.
    - `secrets-manager`: values from **Secrets Manager** → no secrets in the repo.
- `phases`: commands, run in this order.
    - `install`: install tools and dependencies.
    - `pre_build`: last setup before building (e.g. log in to ECR).
    - `build`: the real build and tests.
    - `post_build`: finishing touches (e.g. zip the output, push an image).
    - A failed `install` / `pre_build` ends the build; after a failed `build`, `post_build` still runs (AWS docs, not on the slides).
- `artifacts`: files to upload to **S3**, encrypted with **KMS**.
- `cache`: paths saved to **S3** (usually dependencies) → next builds are faster.

### Security & Monitoring

- **Security**
    - **IAM service role**: what the build can access (S3, ECR, Parameter Store…).
    - **KMS**: encrypts the build artifacts.
    - **VPC**: run the build inside your VPC to reach private resources (RDS, internal APIs).
    - **CloudTrail**: logs the CodeBuild API calls.
- **Monitoring**
    - **CloudWatch Logs / S3**: build output logs.
    - **CloudWatch Metrics**: build statistics (builds, failures, duration).
    - **EventBridge**: rule on a **failed build** → notify (SNS, Lambda).
    - **CloudWatch Alarms**: alert when failures cross a **threshold**.

## AWS CodeDeploy

### Overview

- **Automates deployments** of new app versions, gradually, with **automatic rollback**.
- **appspec.yml**: describes how to deploy (what to copy where, which scripts / hooks to run).
- Compute platforms and the deployment types each supports:
    - **EC2 / on-premises**: **in-place** or **blue/green** (blue/green on EC2 only, not on-premises servers).
    - **Lambda**: blue/green only, by **shifting traffic** between versions of an alias.
    - **ECS**: blue/green only, a new **task definition** behind an ALB.
- **Rollback** when the deployment fails or a **CloudWatch alarm** fires.

### EC2 / On-premises

- ![CodeDeploy on EC2 or on-premises: the CodeDeploy agent on the instances of a deployment group polls CodeDeploy for work, receives the deployment and its configuration, and downloads the revision (app.zip + appspec.yml) from S3 using the instance profile's s3:Get* and s3:List* permissions; Systems Manager can install and update the agent](../images/CodeDeployAgent.png)
- **CodeDeploy agent** must run on every target instance (prerequisite).
    - **Systems Manager** can install it and keep it updated.
    - The agent downloads the **revision** (bundle) from **S3** → the instance profile needs `s3:Get*` / `s3:List*`.
- **Deployment group**: the target instances, chosen by **tags** or an **Auto Scaling group**.
- **Deployment configurations** (in-place speed vs availability):
    - ![In-place deployment configurations on 4 EC2 instances: AllAtOnce updates all four at once so none is serving; HalfAtATime updates two then two so two are serving; OneAtATime updates one by one so three are serving; Custom sets the minimum healthy hosts, e.g. 75%](../images/CodeDeployInPlaceConfigs.png)
    - `AllAtOnce`: fastest, **downtime**.
    - `HalfAtATime`: **50%** capacity during the deployment.
    - `OneAtATime`: slowest, **least** impact on availability.
    - **Custom**: set your own minimum healthy hosts (%).
- **Hooks**: scripts run at each lifecycle step → verify the deployment after each phase.
- **Deploy to an ASG**
    - ![CodeDeploy blue/green on EC2 with an Auto Scaling group: before, the ALB sends 100% of traffic to the blue ASG with v1; CodeDeploy creates a green copy of the ASG and deploys v2 to it, reroutes the ALB to the green instances, then terminates the blue instances after the chosen wait](../images/CodeDeployBlueGreenASG.png)
    - **In-place**: updates the existing instances; instances the ASG launches later **also get the current version** automatically.
    - **Blue/green**: CodeDeploy creates a **new ASG** (settings copied), deploys v2, switches the **load balancer** to it.
        - You choose how long to **keep the old instances** before they are terminated.
        - **Needs an ELB** to move the traffic.

### Lambda & ECS (traffic shifting)

- ![Traffic shifting to v2 over time: AllAtOnce sends 100% immediately, Canary10Percent5Minutes sends 10% for 5 minutes then 100%, Linear10PercentEvery3Minutes adds 10% every 3 minutes and reaches 100% at 27 minutes; config names start with Lambda or ECS](../images/CodeDeployTrafficShifting.png)
- **Lambda**: shifts the traffic of an **alias** (e.g. `prod`) from the old version to the new one; built into **SAM**.
- **ECS**: deploys a new **task definition** to a **green target group**, then shifts the **ALB** (required) from blue to green.
- Shifting patterns (names start with `Lambda…` or `ECS…`):
    - **Linear**: +X% every N minutes until 100%, e.g. `LambdaLinear10PercentEvery3Minutes`, `ECSLinear10PercentEvery10Minutes`.
    - **Canary**: X% first, then **100%** after N minutes, e.g. `LambdaCanary10Percent5Minutes`, `ECSCanary10Percent30Minutes`.
    - **AllAtOnce**: 100% immediately.
- A CloudWatch alarm during the shift → traffic goes back to the old version.

### Redeploy & Rollbacks

- **Rollback** = deploy an **earlier revision** again.
    - **Automatic**: when the deployment **fails**, or a **CloudWatch alarm** threshold is met.
    - **Manual**: start it yourself.
    - Or **disable rollbacks** for a deployment.
- A rollback is a **new deployment** of the last known good revision (new deployment ID), **not a restore** of the old one.
