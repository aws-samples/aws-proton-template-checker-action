# AWS Proton Template checker action ⚛️

This action helps you to validate changes to an AWS Proton template (CloudFormation & Jinja template) are valid. Whenever a pull request is created that includes a file in your template bundle, this action will:

1. Render the template using a sample spec
2. Run the rendered templates through `cfn-lint`
3. Publish a summary of any findings 
4. Fail if it finds any errors

## Usage 

Add the following GitHub workflow to your template repository to validate template changes during pull requests (it's recomended you turn on branch protection so invalid are blocked from being merged).

```yaml
name: Validate Updated Templates
on:
  pull_request:
    branches: [ "main" ]
jobs:
  TemplateChecker:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0
    - name: Get changed files
      id: changed-files
      uses: tj-actions/changed-files@v34
      with:
        separator: ","
    - name: Validate changed templates
      uses: aws-samples/aws-proton-template-checker-action@v1.0
      with: 
        changed_files: "${{steps.changed-files.outputs.all_changed_files}}"
```

## Requirements

To use this action, you have to provide a _few_ additional files in your template bundle. 

```
spec/spec.yaml
spec/sample-outputs.yaml [only for service templates]
```

The `spec.yaml` file needs to contain a valid spec for the template it's a part of. These values will be used to render the template.

The `sample-outputs.yaml` is a simple yaml file which contains a list of key/values to emulate environment and service outputs. This is only needed for service templates (service instances and pipelines).

### Spec files and layout

In order to render your template, this action expects a `spec/` directory in the template bundle with a sample `spec.yaml` and an optional `sample-outputs.yaml` file for Service Templates. Here's an example layout:


```
/my-template/instance_infrastructure/...
/my-template/pipeline_infrastructure/...
/my-template/schema/                        # The schema is validated and used to inject default values
/my-template/schema/schema.yaml
/my-template/spec/
/my-template/spec/spec.yaml                 # This is a real life spec, filled out. This is used to emulate a developer spec
/my-template/spec/sample-outputs.yaml       # This file contains a key value yaml of sample environment and service outputs [service templates only]
```

#### `sample-outputs.yaml` file

The `sample-outputs.yaml` file emulates outputs that'll be used to render your template. Service templates (including Pipelines) can use outputs from environments and components when rendering (as well as service instance outputs when rendering a pipeline template). Since the `aws-proton-template-checker-action` renders your template without registering it with AWS Proton, these outputs aren't available. In order to emulate the presence of these outputs, you can include fake outputs in the `sample-outputs.yaml` file:

```yaml
environment:
  SNSTopicArn: arn:aws:sns:my-cool-topic
  SNSTopicName: my-cool-topic
  VPCSecurityGroup: daves-cool-security-group
  PrivateSubnet1: subnet-1
  PrivateSubnet2: subnet-2
  PublicSubnet1: public-subnet-1
  PublicSubnet2: public-subnet-2
service:
  LambdaRuntime: ruby2.7
```

These values will be piped into your template and will fill in the paramaterized values in your template. The above sample outputs will be applied to the template below:

```yaml
Environment:
  Variables:
    SNSTopicArn: '{{environment.outputs.SNSTopicArn}}'
Policies:
  - AWSLambdaVPCAccessExecutionRole
  - SNSPublishMessagePolicy:
      TopicName: '{{environment.outputs.SNSTopicName}}'
VpcConfig:
  SecurityGroupIds:
    - '{{environment.outputs.VPCSecurityGroup}}'
  SubnetIds:
  {% if service_instance.inputs.subnet_type == 'private' %}
      - '{{environment.outputs.PrivateSubnet1}}'
      - '{{environment.outputs.PrivateSubnet2}}'
  {% else %}
      - '{{environment.outputs.PublicSubnet1}}'
      - '{{environment.outputs.PublicSubnet2}}'
  {% endif %}
```

Pipeline templates also allow you to include outputs from service instances. the `service` block of the `sample-outputs.yaml` file will provide the values  for your template. As an example, see the below snippet from a pipeline template. 

```yaml
Environment:
  Variables:
    Runtime: '{{service_instances[0].outputs.LambdaRuntime}}'
```

For simplicity, you provide sample outputs for your service, environment and components only once (not once per service instance or environment).  

## Limitations

This is currently a work-in-progress, here are some things we don't currently support:

1. Components outputs
2. Schemas defining `default` values must use a lower-cased `default`. 
3. Error handling when a sample output is not available to template has somewhat cryptic error messaging. 

## CFN Lint Errors

By default, this action will run all CFN Lint rules for all regions. If this is not what you want, you can add [CFN Overrides](https://github.com/aws-cloudformation/cfn-lint#template-based-metadata) to your template. 

As an example:

```
Metadata:
  cfn-lint:
    config:
      regions:
        - us-east-1
        - us-east-2
      ignore_checks:
        - E2530
```        
