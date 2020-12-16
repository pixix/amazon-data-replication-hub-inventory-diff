
[中文](./README_CN.md)

# AWS Data Replication Hub - Inventory Diff Plugin

_This AWS Date Replication Hub - Inventory Diff Plugin is based on 
[amazon-s3-resumable-upload](https://github.com/aws-samples/amazon-s3-resumable-upload) contributed by
[huangzbaws@](https://github.com/huangzbaws)._

[AWS Data Replication Hub](https://github.com/awslabs/aws-data-replication-hub) is a solution for replicating data from different sources into AWS. This project is for 
S3 replication plugin. Each of the replication plugin can run independently. 

The following are the planned features of this plugin.

- [x] Amazon S3 object replication between AWS Standard partition and AWS CN partition
- [x] Replication from Aliyun OSS to Amazon S3
- [x] Replication from Tencent COS to Amazon S3
- [x] Replication from Qiniu Kodo to Amazon S3
- [ ] Replication from Huawei Cloud OBS to Amazon S3
- [x] Support replication with Metadata
- [x] Support One-time replication
- [x] Support Incremental replication

## Architect

![S3 Plugin Architect](s3-plugin-architect.png)

An ECS Task running in AWS Fargate lists all the objects in source and destination buckets and determines what objects should be
replicated, a message for each object to be replicated will be created in SQS. A *time-based CloudWatch rule* will trigger the ECS task to run every hour.

The *JobWorker* Lambda function consumes the message in SQS and transfer the object from source bucket to destination 
bucket.

If an object or a part of an object failed to transfer, the lambda will try a few times. If it still failed after
a few retries, the message will be put in `SQS Dead-Letter-Queue`. A CloudWatch alarm will be triggered and a subsequent email notification will be sent via SNS. Note that the ECS task in the next run will identify the failed objects or parts and the replication process will start again for them.

This plugin supports transfer large size file. It will divide it into small parts and leverage the 
[multipart upload](https://docs.aws.amazon.com/AmazonS3/latest/dev/mpuoverview.html) feature of Amazon S3.


## Deployment

Things to know about the deployment of this plugin:

- The deployment will automatically provision resources like lambda, dynamoDB table, ECS Task Definition in your AWS account, etc.
- The deployment will take approximately 3-5 minutes.
- Once the deployment is completed, the data replication task will start right away.

###  Before Deployment

- Configure **credentials**

You will need to provide `AccessKeyID` and `SecretAccessKey` (namely `AK/SK`) to read or write bucket in S3 from another partition or in other cloud storage service. And a Parameter Store is used to store the credentials in a secure manner.

Please create a parameter in **Parameter Store** from **AWS Systems Manager**, you can use default name `drh-credentials` (optional), select **SecureString** as its type, and put a **Value** following below format.

```
{
  "access_key_id": "<Your Access Key ID>",
  "secret_access_key": "<Your Access Key Secret>",
  "region_name": "<Your Region>"
}
```

- Set up **ECS Cluster** and **VPC**

The deployment of this plugin will launch an ECS Task running in Fargate in your AWS Account, hence you will need to set up an ECS Cluster and the VPC before the deployment if you haven't got any. 

> Note: For ECS Cluster, you can choose **Networking only** type. For VPC, please make sure the VPC should have at least two subnets across two available zones.


### Available Parameters

The following are the all allowed parameters for deployment:

| Parameter                 | Default          | Description                                                                                                               |
|---------------------------|------------------|---------------------------------------------------------------------------------------------------------------------------|
| sourceType                | Amazon_S3        | Choose type of source storage, for example Amazon_S3, Aliyun_OSS, Qiniu_Kodo, Tencent_COS                                 |
| jobType                   | GET              | Choose GET if source bucket is not in current account. Otherwise, choose PUT.                                             |
| srcBucketName             | <requires input> | Source bucket name.                                                                                                       |
| srcBucketPrefix           | ''               | Source bucket object prefix. The plugin will only copy keys with the certain prefix.                                      |
| destBucketName            | <requires input> | Destination bucket name.                                                                                                  |
| destBucketPrefix          | ''               | Destination bucket prefix. The plugin will upload to certain prefix.                                                      |
| destStorageClass          | STANDARD         | Destination Object Storage Class.  Allowed options: 'STANDARD', 'STANDARD_IA', 'ONEZONE_IA', 'INTELLIGENT_TIERING'        |
| ecsClusterName            | <requires input> | ECS Cluster Name to run ECS task                                                                                          |
| ecsVpcId                  | <requires input> | VPC ID to run ECS task, e.g. vpc-bef13dc7                                                                                 |
| ecsSubnets                | <requires input> | Subnet IDs to run ECS task. Please provide two subnets at least delimited by comma, e.g. subnet-97bfc4cd,subnet-7ad7de32  |
| credentialsParameterStore | drh-credentials  | The Parameter Name used to keep credentials in Parameter Store.                                                           |
| alarmEmail                | <requires input> | Alarm email. Errors will be sent to this email.                                                                           |
| lambdaMemory              | 256              | Lambda Memory, default to 256 MB.                                                                                         |
| multipartThreshold        | 10               | Threshold Size for multipart upload in MB, default to 10 (MB)                                                             |
| chunkSize                 | 5                | Chunk Size for multipart upload in MB, default to 5 (MB)                                                                  |
| maxThreads                |10                | Max Theads to run multipart upload in lambda, default to 10                                                               |

### Deploy via AWS Cloudformation

Please follow below steps to deploy this plugin via AWS Cloudformation.

1. Sign in to AWS Management Console, switch to the region to deploy the CloudFormation Stack to.

1. Click the following button to launch the CloudFormation Stack in that region.

    - For Standard Partition

    [![Launch Stack](launch-stack.svg)](https://console.aws.amazon.com/cloudformation/home#/stacks/create/template?stackName=DataReplicationS3Stack&templateURL=https://aws-gcr-solutions.s3.amazonaws.com/Aws-data-replication-component-s3/latest/Aws-data-replication-component-s3.template)

    - For China Partition

    [![Launch Stack](launch-stack.svg)](https://console.amazonaws.cn/cloudformation/home#/stacks/create/template?stackName=DataReplicationS3Stack&templateURL=https://aws-gcr-solutions.s3.cn-north-1.amazonaws.com.cn/Aws-data-replication-component-s3/latest/Aws-data-replication-component-s3.template)
    
1. Click **Next**. Specify values to parameters accordingly. Change the stack name if required.

1. Click **Next**. Configure additional stack options such as tags (Optional). 

1. Click **Next**. Review and confirm acknowledgement,  then click **Create Stack** to start the deployment.

If you want to make custom changes to this plugin, you can follow [custom build](CUSTOM_BUILD.md) guide.

> Note: You can simply delete the stack from CloudFormation console if the replication task is no longer required.

### Deploy via AWS CDK

If you want to use AWS CDK to deploy this plugin, please make sure you have met below prerequisites:

* [AWS Command Line Interface](https://aws.amazon.com/cli/)
* Node.js 12.x or later
* Docker

Under the project **source** folder, run below to compile TypeScript into JavaScript. 

```
cd source
npm install -g aws-cdk
npm install && npm run build
```

Then you can run `cdk deploy` command to deploy the plugin. Please specify the parameter values accordingly, for example:

```
cdk deploy --parameters srcBucketName=<source-bucket-name> \
--parameters destBucketName=<dest-bucket-name> \
--parameters alarmEmail=xxxxx@example.com \
--parameters jobType=GET \
--parameters sourceType=Amazon_S3 \
--parameters ecsClusterName=test \
--parameters ecsVpcId=vpc-bef13dc7 \
--parameters ecsSubnets=subnet-97bfc4cd,subnet-7ad7de32
```

> Note: You can simply run `cdk destroy` if the replication task is no longer required. This command will remove the stack created by this plugin from your AWS account.
#========================================


# amazon-inventory-diff
amazon-inventory-diff

The cdk-solution-init-pkg provides a reference for building solutions using the AWS Cloud Development Kit (CDK). This package contains basic build scripts, sample source code implementations, and other essentials for creating a solution from scratch.

***

## Initializing the Repository

After successfully cloning the repository into your local development environment, a source code package must be built based on your language of choice. This will define which language the CDK code will be written in.

Run the `initialize-repo.sh` script at the root level of the project file. This script will prompt a series of questions before initializing a git repo using the current directory name as the solution name. It will also stage the `deployment` and `source` directories with fundamental assets for your solution.

- The language selected when running this script will determine whether to provision a TypeScript, Python, Java, or C# CDK project into your `deployment` and `source` directories. This is the language you will be working with when defining your infrastructure using the CDK. Your source code packages for Lambda functions and custom resources may be in different languages.

***

## File Structure

Upon successfully cloning the repository into your local development environment but **prior** to running the initialization script, you will see the following file structure in your editor:

```
|- .github/ ...               - resources for open-source contributions.
|- deployment/                - contains build scripts, deployment templates, and dist folders for staging assets.
  |- .typescript/                - typescript-specific deployment assets.
  |- .python/                   - python-specific deployment assets.
  |- .java/                     - java-specific deployment assets.
  |- .csharp/                   - csharp-specific deployment assets.
|- source/                    - all source code, scripts, tests, etc.
  |- .typescript/                - typescript-specific source assets.
  |- .python/                   - python-specific source assets.
  |- .java/                     - java-specific source assets.
  |- .csharp/                   - csharp-specific source assets.
|- .gitignore
|- .viperlightignore          - Viperlight scan ignore configuration  (accepts file, path, or line item).
|- .viperlightrc              - Viperlight scan configuration.
|- buildspec.yml              - main build specification for CodeBuild to perform builds and execute unit tests.
|- CHANGELOG.md               - required for every solution to include changes based on version to auto-build release notes.
|- CODE_OF_CONDUCT.md         - standardized open source file for all solutions.
|- CONTRIBUTING.md            - standardized open source file for all solutions.
|- copy-repo.sh               - copies the baseline repo to another directory and optionally initializes it there.
|- initialize-repo.sh         - initializes the repo.
|- LICENSE.txt                - required open source file for all solutions - should contain the Apache 2.0 license.
|- NOTICE.txt                 - required open source file for all solutions - should contain references to all 3rd party libraries.
|- README.md                  - required file for all solutions.

* Note: Not all languages are supported at this time. Actual appearance may vary depending on release.
```

**After** running the initialization script, you will see a language-specific directory in both the `/source` and `/deployment` folders expanded based on your CDK language choice. Example below after `./initialize-repo.sh` is run with `typescript` selected as the language of choice. Notice the removal of the language-specific directories after running the command. The repo is now ready for solution development.

```
|- .github/ ...               - resources for open-source contributions.
|- deployment/                - contains build scripts, deployment templates, and dist folders for staging assets.
  |- cdk-solution-helper/     - helper function for converting CDK output to a format compatible with the AWS Solutions pipelines.
  |- build-open-source-dist.sh  - builds the open source package with cleaned assets and builds a .zip file in the /open-source folder for distribution to GitHub
  |- build-s3-dist.sh         - builds the solution and copies artifacts to the appropriate /global-s3-assets or /regional-s3-assets folders.
  |- clean-dists.sh           - utility script for clearing distributables.
|- source/                    - all source code, scripts, tests, etc.
  |- bin/
    |- cdk-solution.ts        - the CDK app that wraps your solution.
  |- lambda/                  - example Lambda function with source code and test cases.
    |- test/                   
    |- index.js                 
    |- package.json             
  |- lib/
    |- cdk-solution-stack.ts  - the main CDK stack for your solution.
  |- test/
    |- __snapshots__/
    |- cdk-solution-test.ts   - example unit and snapshot tests for CDK project.
  |- cdk.json                 - config file for CDK.
  |- jest.config.js           - config file for unit tests.
  |- package.json             - package file for the CDK project.
  |- README.md                - doc file for the CDK project.
  |- run-all-tests.sh         - runs all tests within the /source folder. Referenced in the buildspec and build scripts.
|- .gitignore
|- .viperlightignore          - Viperlight scan ignore configuration  (accepts file, path, or line item).
|- .viperlightrc              - Viperlight scan configuration.
|- buildspec.yml              - main build specification for CodeBuild to perform builds and execute unit tests.
|- CHANGELOG.md               - required for every solution to include changes based on version to auto-build release notes.
|- CODE_OF_CONDUCT.md         - standardized open source file for all solutions.
|- CONTRIBUTING.md            - standardized open source file for all solutions.
|- LICENSE.txt                - required open source file for all solutions - should contain the Apache 2.0 license.
|- NOTICE.txt                 - required open source file for all solutions - should contain references to all 3rd party libraries.
|- README.md                  - required file for all solutions.
```

***

## Building your CDK Project

After initializing the repository, make any desired code changes. As you work through the development process, the following commands might be useful for 
periodic testing and/or formal testing once development is completed. These commands are CDK-related and should be run at the /source level of your project.

CDK commands:
- `cdk init` - creates a new, empty CDK project that can be used with your AWS account.
- `cdk synth` - synthesizes and prints the CloudFormation template generated from your CDK project to the CLI.
- `cdk deploy` - deploys your CDK project into your AWS account. Useful for validating a full build run as well as performing functional/integration testing
of the solution architecture.

Additional scripts related to building, testing, and cleaning-up assets may be found in the package.json file or in similar locations for your selected CDK language. You can also run `cdk -h` in the terminal for details on additional commands.

***

## Running Unit Tests

The `/source/run-all-tests.sh` script is the centralized script for running all unit, integration, and snapshot tests for both the CDK project as well as any associated Lambda functions or other source code packages. 

- Note: It is the developer's responsibility to ensure that all test commands are called in this script, and that it is kept up to date.

This script is called from the solution build scripts to ensure that specified tests are passing while performing build, validation and publishing tasks via the pipeline.

***

## Building Project Distributable
* Configure the bucket name of your target Amazon S3 distribution bucket
```
export DIST_OUTPUT_BUCKET=my-bucket-name # bucket where customized code will reside
export SOLUTION_NAME=my-solution-name
export VERSION=my-version # version number for the customized code
```
_Note:_ You would have to create an S3 bucket with the prefix 'my-bucket-name-<aws_region>'; aws_region is where you are testing the customized solution. Also, the assets in bucket should be publicly accessible.

* Now build the distributable:
```
chmod +x ./build-s3-dist.sh \n
./build-s3-dist.sh $DIST_OUTPUT_BUCKET $SOLUTION_NAME $VERSION \n
```

* Deploy the distributable to an Amazon S3 bucket in your account. _Note:_ you must have the AWS Command Line Interface installed.
```
aws s3 cp ./global-s3-assets/ s3://my-bucket-name-/$SOLUTION_NAME/$VERSION/ --recursive --acl bucket-owner-full-control --profile aws-cred-profile-name \n
```

* Get the link of the solution template uploaded to your Amazon S3 bucket.
* Deploy the solution to your account by launching a new AWS CloudFormation stack using the link of the solution template in Amazon S3.

***

## Building Open-Source Distributable

* Run the following command to build the open-source project:
```
chmod +x ./build-open-source-dist.sh \n
./build-open-source-dist.sh $SOLUTION_NAME \n
```

* Validate that the assets within the output folder are accurate and that there are no missing files.

***

Copyright 2019 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at

    http://www.apache.org/licenses/

or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the License for the specific language governing permissions and limitations under the License.
