ECS Pipeline Documentation
==========================
This pipeline uses AWS CodePipeline, AWS CodeBuild and CloudFormation
to continuously deploy application code from Github to a deployed solution
in an ECS cluster.

### Assumptions
- Source Code is hosted in Github
- One pipeline per environment
- ECS Cluster is already created
- S3 Bucket where application configuration will be stored. Idealy this bucket
  is locked down as much as possible to keep application secrets safe. At a minimum
  any pipelines using this bucket will need access.
- ELBv2(Application Load Balancer) is deployed and a target group is created that the
  ECS service will be registered with.
- Application Repository already has a working Dockerfile

### Definitions
- `CloudFormation Source Bucket` - The S3 bucket that contain the CloudFormation template
  and the corresponding CloudFormation options file.
- `ECS Cluster Name` - The name of the ECS Cluster this application will be deployed on.
- `Target Group Arn` - The ARN for a target group created for use with this application.

### Pipeline Prep

1) Ensure the ECR Repository is [created.](http://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html)
2) Upload the [ECS Service CloudFormation template](service.template) to the
[CloudFormation Source Bucket](#definitions). Note the Parameters starting with
DB are not actually required and are an example of how you might pass an environment
variable to your application when using this method.
3) Create a file named service-options.json with the following content, this will
be referred to as the options file in the rest of this document. Replace
__ECS Cluster Name__ with the [ECS Cluster Name](#definitions) and __Target Group Arn__
with the [target group ARN](#definitions). The __App Name__ should be unique across all of
your applications. Any of the DB\* parameters can be ignored if they do not apply, these
are an example of how environment variables are passed to the containers. Once this file
is finished being edited upload it to the [CloudFormation Source Bucket](#definitions).

```javascript
{
  "Parameters": {
    "AppName": "<App Name>",
    "ContainerImage": "IMAGE_NAME_PLACEHOLDER",
    "DbConnection": "mysql",
    "DbDatabase": "my_database",
    "DbHost": "mysql",
    "DbPassword": "secret",
    "DbPort": "3306",
    "DbUsername": "root",
    "DesiredCount": "1",
    "ECSCluster": "<ECS Cluster Name>",
    "Environment": "Development", "MaxCapacity": "4",
    "MinCapacity": "2",
    "TargetGroupArn": "<Target Group Arn>"
  }
}
```

### Application Repo
To prepare the application repository we need to ensure a buildspec file exists in the
root of the repository.

In the application repository add the [buildspec.yml](buildspec.yml) file to the root of the repository.

#### Pre Build
In the prebuild we are generating a timestamp that will be used as a part of the tag name
for the new container. This is because CloudFormation won't deploy a change if a resource
doesn't actually change so we need to ensure that image name is always updated.
```yaml
commands:
  - echo Setting timestamp for container tag
  - echo `date +%s` > timestamp #Echo the current date time stamp to a file so it can be read later
```

#### Build
This is largely uninteresting, we are simply building the Docker container using the git
repository name and then tagging it with the ECR URL so it can be pushed to ECR.

```yaml
commands:
  - echo Building and tagging container
  - docker build -t $REPOSITORY_NAME .
  - docker tag $REPOSITORY_NAME $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$REPOSITORY_NAME:$BRANCH-`cat ./timestamp`

```

#### Post Build
First we push the docker image to ECR in Post build.
```yaml
commands:
  - echo Pushing docker image
  - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$REPOSITORY_NAME:$BRANCH-`cat ./timestamp`
```

Next we pull down the Cloudformation template and options file from the [CloudFormation Source Bucket](#definitions).
We then replace the text __IMAGE_NAME_PLACEHOLDER__ with the name of the Docker image that was
just built.
```yaml
  - echo Preparing CloudFormation Artifacts
  - aws s3 cp s3://$ECS_SERVICE_BUCKET/$ECS_SERVICE_KEY task-definition.template
  - aws s3 cp s3://$ECS_SERVICE_PARAMS_BUCKET/$ECS_SERVICE_PARAMS_KEY cf-config.json
  # We assume the parameters json file has IMAGE_NAME_PLACEHOLDER in it, if not this will not work
  - sed -i "s/IMAGE_NAME_PLACEHOLDER/$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com\/$(cat ./repo_name):$BRANCH-$(cat ./timestamp)/g" ./cf-config.json
```

#### Artifacts
Our final act is to define the task-definition.template and cf-config.json files which will be
used as input artifacts for the final stage in the pipeline.

```yaml
artifacts:
  files:
    - task-definition.template
    - cf-config.json
```

### Pipeline Deployment
Finally it is time to launch the Pipeline stack! For this new stack you simply need to create
a new stack using the [pipeline](pipeline.yaml) template.

Here is an outline of the parameters needed for the stack and what purpose they serve:
- `Environment` - This should match the environment set in your [options](#pipeline_prep)
  file.
- `OAuthToken` - This OAuth token should have access to the repository you want to deploy
  with the admin:repo_hook and repo permissions. [Github Doc for Token Creation](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)
- `Organization` - This is the Github Organization that your repository is located in
  so for this repo it would be rackerlabs.
- `Repository` - This is the name of the Github repository to deploy
- `Branch` - The name of the branch you want to continuosly deploy with this pipeline.
- `ECSServiceParameterBucket` - This is the bucket you placed your [options file](#pipeline_prep)
  and [template](#pipeline_prep) in.
- `ECSServiceParameterKey` - The S3 Key of the [options file](#pipeline_prep) located in your
  [CloudFormation Source Bucket](#definitions).
- `ECSServiceBucket` - This is the bucket you placed your [options file](#pipeline_prep)
  and [template](#pipeline_prep) in.
- `ECSServiceKey` - The name of the [CloudFormation template](#pipeline_prep), if you followed
  this tutorial then it should be __service.template__.
- `ECSServiceStack` - The name of the CloudFormation stack that will be created by your pipeline.
  This is the stack that deploys your ECS service and should likely have the name of your
  app located in it.
