# AWS CICD

CICD: Continuous Integration Continuous Deployment

Expect a lot of questions – need to do some practical on this.

- CodeCommit (CI): code repo like GitHub
- CodePipeline: orchestrator of CICD in AWS
- CodeBuild (CI): builds and tests code from repos, like Jenkins
- CodeDeploy: deploy build onto application servers, CD involves regular delivery, e.g. multiple times per day. Alternative to ElasticBeanstalk deployment.
- CodeStar
- CodeArtifact
- CodeGuru

## CodeCommit

- Might be deprecated, integrate with GitHub instead.
- Version control - uses Git.
- CodeCommit gives a private repo
- No repo size limit
- Managed, fully available
- Secure, encrypted (using KMS, etc), IAM Access Control
- Can integrate with other tools, e.g. Jenkins
- Authentication: SSH, or HTTPS
- Authorisation: IAM Policies
- Cross account access via IAM and STS – check more about STS
- Supports code reviews and Pull requests
- Minimal UI compared to GitHub.

## CodePipeline

- Visual workflow tool to orchestrate CICD
- Source, build, tests, deploy, invoke (e.g. Lambdas).
- Stages – can have sequential or parallel actions, can define manual approval stages, e.g. for Prod deployment
- Each pipelines created artefacts stored in S3 to be passed onto the next pipeline
- Example: AWS CodeCommit source code -> pulled from CodeBuild and output stored in S3, CodeDeploy picks up output from S3 and deploys to servers.
- Troubleshooting: CloudWatch events (via EventBridge), can trigger actions, e.g. send an email on build failures.
- Pipeline needs enough IAM permissions, e.g. to read/write from/to S3 bucket.
- AWS CloudTrail can be used to Audit API Calls made from CodePipeline

### Demo

- Prerequisite: create an Elastic BeanStalk environment, 2 environments; Prod and dev.
- Build a custom pipeline – name, with QUEUED execution mode.
- Create a new Service Role for permissions
- Source, e.g. GitHub V2, S3, create the GitHub connection
- Repository name and branch name
- Output artefact format
- Trigger – e.g., push to main branch
- Build stage (skipped)
- Deploy – provider, region, environment, e.g. dev or Prod
- IAM Policy: Add AdministratorAccessElasticBeanStalk for Pipeline
- Commit a code change to the repo – the pipeline deploys to Elastic Beanstalk dev environment

### Demo – adding a stage

- Edit the Pipeline, add a stage called Deploy to Prod
- Add Action Group – provider – Elastic Beanstalk, choose Prod environment.
- Add Action – manual approval

## CodeBuild

- Build instructions in buildspec.yml in the root of source code directory (can configure a different name or sub-directory).
- Monitoring: CloudWatch metrics, EventBridge, CloudWatch, etc, to monitor the build.
- Build runs on a container running a docker image, AWS provide a series of docker images for various programming language/frameworks, the config in the buildspec.yml is used by the running container. Can extend an AWS base image for unsupported languages.
- Feature to cache build files in S3 (optional) for faster build performance.
- Output build artefacts to an S3 bucket.

### Buildspec.yml

- At root, can override.
- Env vars: plaintext, parameter store SSM or AWS Secrets manager
- Phases – install (dependencies), pre_build, build, post_build (e.g. Zip up output)
- Artifacts – what to upload to S3
- Cache block

#### Example

```
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo Installing dependencies...
      - npm install
  pre_build:
    commands:
      - echo Running lint checks...
      - npm run lint
  build:
    commands:
      - echo Build started on `date`
      - npm run build
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Running tests...
      - npm test

artifacts:
  files:
    - '**/*'
  discard-paths: yes
  base-directory: dist
```

### Demo

- Code Build – create new, GitHub source, repo
- Check: rebuild every time code is pushed option, on PUSH
- Environment – managed image – use relevant AWS image
- Role name (for permissions)
- Timeout 1hour default
- VPC – to run code build in your VPC
- Compute size (for building container to use)
- Env vars
- Build spec: use a buildspec file vs insert build commands (choose buildspec file option), it is here that you can override the buildspec file name or directory if not at root.
- Artefacts: options to send items to S3
- Start a build – if there is a found buildspec.yml file in the source code.
- Add a test stage to CodePipeline
- Once you have tested on build, remove the option from CodeBuild for ‘rebuild every time a change is pushed’. This stops code changes triggering the build at the code build level – it will be moved to the Code pipeline instead.
- Go to CodePipeline – edit pipeline
- Add a stage: TestCode
- Action Provider: CodeBuild
- Output artefact: OutputOfTest
- This stage is inserted before deploying to Prod.
- The code in the repo has HTML, this is a grep command in the build phase (in buildspec.yml) that checks for some text in the HTML file. To make the check/test fail, the text is changed in the HTML so the grep cmd doesn’t find the text.

  Build phase will fail, and stop the Pipeline, no prod deployment happens.

## CodeDeploy

- Automates application version deployment,e.g. v1 to v2
- Targets: EC2, ECS, Lambda, On-prem servers.
- Allows update and rollback
- Gradual deployment, can configure deployment speed.
- appspec.yml – file to configure deployment

### Example appspec.yml

```
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html

hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root

  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 300
      runas: root

  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root

  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 300
      runas: root

```

### CodeDeploy - EC2 or On-Prem Deployments

- In-place deployment or blue/green deployment
- CodeDeploy Agent must be installed on target instances
- Deployment strategies/speeds:
  - AllAtOnce
  - HalfAtATime – half stopped, half started with new version, half stopped, half started with new version, 100% deployed.
  - OneAtATime
- Blue/Green: creates a new auto-scaling group with new instances, traffic routed to new ASG instances, take down old instances, no downtime.

### CodeDeploy Agent

- Can be installed automatically using Systems Manager, or manually.
- EC2 instances must have permission to access S3 to get deployment bundles

### CodeDeploy – Lambda

- Automates traffic shift for Aliases
- Integrated within the SAM framework
- CodeDeploy gradually increases % of traffic from old version to new version of Lambda.
- Strategies:
  - Linear – grow % of traffic every N minutes until 100%
  - Canary – try X %, to test, e.g. 10% for 5 mins, then 100%.
  - AllAtOnce – immediately send 100% to new version

### CodeDeploy - ECS Platform Deployment

- Automate deployment of a new ECS Task Definition
- Only blue/green deployment:
- New target group with new version, same capacity.
- Redirects traffic at ALB for old to new target group
- Strategies: Linear, Canary, AllAtOnce

### Demo - CodeDeploy EC2

- Create an IAM Role for AWS Service: CodeDeploy
- Role must have S3 read access (get example)
- CodeDeploy – create application type: EC2
- Create a deployment group – launch a new EC2 instance first (SSH into instance (to install CodeDeploy Agent) – get the commands)
- Development instances (name)
- Use tags on Instances, set tag Environment=Development
- Choose deployment type: In-Place
- Select: Use EC2 instances
- Set tag group Environment=Development (matches tag on EC2 instances)
- Deployment settings: AllAtOnce
- Load Balancer integration
- Application built output must be in an S3 bucket in the same region as the CodeDeploy service.
- Upload the app .zip to the S3 bucket.
- appspec.yml at the root of the application code will be run (get example):
- files – source/deployment paths
- hooks – BeforeInstall: this point to scripts that are part of the .zip file to setup the application on the EC2 instance, e.g. install and a start a server.
- ApplicationStop: this is run to stop the application running when doing a new deployment.
- Attach instance Role to new IAM Role created earlier
- In CodeDeploy, set the S3 ARN – create the deployment
- Can use hooks to verify the deployment
  ASG

### In-place deployment – updates existing EC2 instances

Blue/green deployment – new ASG created (settings the same as existing ASG). EC2s created in new ASG. Configure the time for the old instances to live before being destroyed. Must be using ELB. Traffic routed to new instances.

### Rollback

- Automatic or manual
- Rolls back to last good deployment by redeploying new instances of that version, not a rollback, but a redeploy of old versions.

## CodeArtifact

- This is for dependencies.
- Secure, scalable artefact management
- Integration: NuGet, Maven , NPM, etc.
- All your artefacts live within your AWS VPC
- Define domains (repositories)
- For local development environments, dependency requests are proxied through CodeArtifact, to the public package location, e.g. NPM.
- The dependencies downloaded from public sources get cached in CodeArtifact for future requests.
- IT leaders can also publish dependencies to CodeArtifact.
- Example: triggering a new CodeBuild if a package is updated, so deployed code always on latest versions of dependencies.

### CodeArtifact – Resource Policy

- Can be used to authorise another account to access CodeArtifact.
- A principal can access All or None of the artefacts in a repository.

### Demo

- Optional – chose upstream dependencies, e.g. NPM
- Domain – account to store artefacts, company-name, key: AWS Managed Key, or Customer managed key.
- Repositories – these are all listed separately, e.g. name given for your repo (company-name), plus all external sources setup (NPM, etc).
- Can configure your local development environments using instructions given in CodeArtefact, e.g. setup up NPM locally to use CodeArtifact repo, instead of public source.
- Can apply permission policies at Repository (lower level) or Domain (higher level).

## CodeGuru

- Supports Java and Python.
- Uses Machine learning to analyse your application.
- CodeGuru Reviewer: code reviews on static code.
- CodeGuru Profiler: performance, detects and optimises your application in the development environment, and monitors performance in Prod and makes recommendations.
- Supports AWS Service app and On-prem apps.
- Has a performance overhead on the application, but minimal.
- Agent runs for CodeGuru on the application instances. You can configure it:
- MaxStackDepth – the depth of the call stack to analyse, e.g. no. Of method calls down the stack.
- MemoryUsageLimit – maximum % of memory the profiler agent can use on the application instance.
- MinimumTimeFor ReportingInMilliseconds – minimum time between sending reports back.
- ReportingIntervalInMilliseconds – frequency to send reports back.
- SamplingIntervalInMilliseconds – short time > no. Of samples. Longer time < no. Of samples.
