# AWS for Fluent Bit Load Testing Framework Guidance

Guidance for AWS for Fluent Bit advanced load testing framework. This framework can run high log throughput tests on [ECS](https://aws.amazon.com/about-aws/whats-new/2019/11/aws-launches-firelens-log-router-for-amazon-ecs-and-aws-fargate/) or [EKS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-logs-FluentBit.html) and validate received logs based on log loss and log duplication to measure Fluent Bit performance. Log destinations will be AWS related output plugins - [CloudWatch](https://docs.fluentbit.io/manual/pipeline/outputs/cloudwatch), [Kinesis](https://docs.fluentbit.io/manual/pipeline/outputs/kinesis), [Firehose](https://docs.fluentbit.io/manual/pipeline/outputs/firehose) and [S3](https://docs.fluentbit.io/manual/pipeline/outputs/s3).

From `AWS for Fluent Bit 2.23.0`, we will publish our ECS load testing results for each latest image in our [release notes](https://github.com/aws/aws-for-fluent-bit/tags) for reference.

## ECS Load Tests

We use [FireLens](https://aws.amazon.com/blogs/containers/under-the-hood-firelens-for-amazon-ecs-tasks/) as the log router to run tasks with different throughput as ECS load tests. Detailed config for each task can be found here [Task definitions](#Task-definitions). An ECS cluster will be launched to run ECS tasks and after all tasks finished, the framework will run validation script to validate logs at the destination and return a result like

```
Total object in S3:  364
Total input record:  12000000
Total record in destination:  12000000
Unique record in destination:  12000000
Duplicate records:  0
Log Loss:  0 %
```

### Task definitions
1. [CloudWatch](https://github.com/aws/aws-for-fluent-bit/blob/mainline/load_tests/task_definitions/cloudwatch.json)
2. [Kinesis](https://github.com/aws/aws-for-fluent-bit/blob/mainline/load_tests/task_definitions/kinesis.json)
3. [Firehose](https://github.com/aws/aws-for-fluent-bit/blob/mainline/load_tests/task_definitions/firehose.json) 
4. [S3](https://github.com/aws/aws-for-fluent-bit/blob/mainline/load_tests/task_definitions/s3.json)

## EKS Load Tests

In EKS, we set up Fluent Bit as a DaemonSet to send logs to AWS destinations. EKS load tests will run each test as a [pod](https://kubernetes.io/docs/concepts/workloads/pods/) on different [nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) with specific throughput and config. An EKS cluster will be launched to hold 4 nodes - each for tests to one specific AWS destination (S3, CloudWatch, Kinesis and Firehose). After all tests finished, the framework will run validation script to validate logs at the destination and return benchmark results. Currently, we only support Cloudwatch load tests and each test will be configured by [ConfigMap](https://github.com/aws/aws-for-fluent-bit/blob/mainline/load_tests/daemonset/cloudwatch.yaml#L32-L121).

### ConfigMap
1. [CloudWatch](https://github.com/aws/aws-for-fluent-bit/blob/mainline/load_tests/daemonset/cloudwatch.yaml)

## Run Load Tests

These tests will use [AWS Cloud Development Kit (CDK)](https://docs.aws.amazon.com/cdk/v2/guide/home.html) to define related AWS resources and provisioning them through [AWS CloudFormation](https://aws.amazon.com/cloudformation/).

### Prerequisites
1. [Configure your AWS account](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html#getting_started_prerequisites)
```
aws configure
```
2. [Install the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html#getting_started_install)
```
npm install -g aws-cdk
```
3. [Bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html#getting_started_bootstrap)
```
cdk bootstrap aws://ACCOUNT-NUMBER/REGION
```
4. EKS specific tools 
- [Install eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- [Install kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [Install aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
5. Set related environment variables
- A list of throughput levels for S3, Kinesis, Firehose plugins
```
export THROUGHPUT_LIST='["20m", "25m", "30m"]'
```
- A list of throughput levels for CloudWatch plugin
```
export CW_THROUGHPUT_LIST='["1m", "2m", "3m"]'
```
- Platform to run the load test. Available values: *ECS*, *EKS*
```
export PLATFORM='ECS'
```
- Desired log destination. Available values: *CloudWatch*, *Kinesis*, *Firehose*, *S3*
```
export OUTPUT_PLUGIN='CloudWatch'
```

Currently, EKS load tests only support *CloudWatch*.
- EKS cluster name
```
export  EKS_CLUSTER_NAME='load-test-fluent-bit-eks-cluster'
```
- AWS creds related environment variables
```
export LOAD_TEST_TASK_ROLE_ARN=<Your task roleARN>
export LOAD_TEST_TASK_EXECUTION_ROLE_ARN=<Your task execution roleARN>
```

More details can be found in [IAM Roles for Tasks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html) and [Amazon ECS task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html).
- Others
```
export TESTING_RESOURCES_STACK_NAME='load-test-fluent-bit-testing-resources'
export LOG_STORAGE_STACK_NAME='load-test-fluent-bit-log-storage'
export PREFIX='load-test-fluent-bit-'
```

Don't forget to change the [variable](https://github.com/aws/aws-for-fluent-bit/blob/mainline/load_tests/setup_test_environment.sh#L18) in the file to your `aws-for-fluent-bit` image like
```
public.ecr.aws/aws-observability/aws-for-fluent-bit:latest
```
6. (EKS only) Launch your EKS cluster 
```
eksctl create cluster -f aws-for-fluent-bit/load_tests/create_testing_resources/eks/cluster.yaml
```
*Note:*
The cluster creation takes around 20 mins and will be launched with 0 nodes. When we run the tests, we will scale up the cluster to 4 nodes (3 mins) and scale it down to 0 node again after tests finished. In this way, we save the creation time every time running the load test.

### Start to run
1. Go to the root path
```
cd aws-for-fluent-bit
```
2. Create a Python virtual environment
```
python3 -m venv .venv
```
3. Activate virtual environment
```
source .venv/bin/activate
```
4. Install the required dependencies
```
pip install -r ./load_tests/requirements.txt
```
5. (ECS only) Deploy S3, Kinesis, Firehose related testing resources
```
cd load_tests/create_testing_resources/kinesis_s3_firehose
cdk deploy
```
*Note:*
Firehose has a hard limit on single delivery stream so you need to use the [Amazon Kinesis Data Firehose Limits form](https://console.aws.amazon.com/support/home#/case/create?issueType=service-limit-increase&limitType=service-code-kinesis-firehose) to request an increase if you want to test high throughput. Therefore, we will not automatically delete this stack in the framework as every newly created Firehose delivery stream needs to ask for an increase and cannot be reused.

6. Set up other testing resources
```
python ./load_tests/load_test.py create_testing_resources
source ./load_tests/setup_test_environment.sh
```
7. Run ECS load tests
```
python ./load_tests/load_test.py ${PLATFORM}
```
8. Delete related testing resources when tests are done
```
python ./load_tests/load_test.py delete_testing_resources
```