# cloudformation_hook_demo
CloudFormation Hook Demo

## Contents
* [Overview](#overview)
* [Basic Terminologies](#terminologies)
  * [Hook](#hook)
  * [Hook Targets](#target)
  * [Target Invocation Point](#point)
  * [Target Action](#action)
  * [Hook Handlers](#handler)
  * [Hook Configuration Schema](#schema)
* [Type configuration](#config)
* [Demo](#demo)
* [S3 KMS Demo](#aws-kms-demo)
* [S3 SSE-S3 Demo](#sse-s3-demo)
* [Clean Up](#cleanup)
* [Summary](#summary)
* [Referrals](#referrals)

<a name="overview"></a>
## Overview

AWS offers two types of guardrails:

1. Preventive guardrails enforce specific policies to help ensure that your accounts operate in alignment to compliance standards, and disallow actions that lead to policy violations. 

2. Detective guardrails detect and alert on unexpected activity and noncompliance of resources within your accounts, such as policy violations. These are helpful in alerting when something requires remediation (either manual or automated). 

This blog is to demonstrate a demo of CloudFormation hook, however to provide some context about CloudFormation Hook, I have taken the following terminologies from [AWS CloudFormation Hook workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/f09fd78b-ef8a-4a9d-9d2b-f31a3e6ca956/en-US/introduction)

AWS CloudFormation Hooks, are a preventive guardrails, a feature that allows users to invoke custom logic to automate actions or inspect resource configurations prior to a create, update or delete CloudFormation stack operation. it allows the users to manage their cloud applications and infrastructure in a safe, predictable, and repeatable way. With AWS CloudFormation Hooks, the users can now validate resource properties and send a warning, or prevent the provisioning operation, for non-compliant resources to reduce security and compliance risk, lower operational overhead, and optimise cost.

With AWS CloudFormation Hooks, the users can publish their policy and controls to the CloudFormation Registry and enforce them against all stack and resource operations in their AWS accounts. For example, the users can inspect their Amazon S3 bucket properties for encryption, public access and logging best practice policies to ensure that developers always create secure S3 buckets in the first place.

A hook includes a hook specification represented by a JSON schema and handlers that are invoked at each hook invocation point. 
In this blog, we will use a pre-existing hook "" to enforce S3 bucket encryption turned on during the bucket creation

---

<a name="terminologies"></a>
## Basic Terminologies:

<a name="hook"></a>
### Hook
A hook contains code that is invoked immediately before CloudFormation creates, updates, or deletes specific resources. Hooks are able to inspect the resources that CloudFormation is about to provision. If hooks find any resources that don’t comply with your organisational guidelines, they are able to prevent CloudFormation from continuing the provisioning process.

<a name="target"></a>
### Hook Targets
Hook targets are the destination where hooks will be invoked against. Hooks support AWS and Non-AWS resources supported by CloudFormation as target. You specify target(s) in the hook schema while authoring your hook. For example, the users can author a hook targeting AWS::S3::Bucket resource. We support ‘n’ targets for a hook. That means, a hook can support multiple targets. There is no limit on number of resource targets a hook can support.

<a name="point"></a>
### Target Invocation Point
Invocation points are the point in provisioning workflow where hooks will be executed. CloudFormation supports PRE (before) invocation point for this release. Meaning, hook authors can write a hook that will be executed before provisioning logic for the target is started. For example, hook with PRE invocation point for target S3 bucket will be executed before CloudFormation starts provisioning S3 bucket in user’s account.

<a name="action"></a>
### Target Action
Target actions are the type of operation hooks will be executed at. These actions are tied with hook targets. Hook targets support CREATE, UPDATE, DELETE actions. For example, when a hook for CREATE action on an S3 target is authored, it will only be executed during a create operation for an S3 bucket.

<a name="handler"></a>
### Hook Handlers
An invocation point and action makes an exact point where the hook will executed. Hook authors write handlers which hosts logic for these specific points. For example, PRE invocation point with CREATE action makes preCreate handler. Hook authors will write code which will get executed any time there is matching target and CloudFormation is performing a matching action.

<a name="schema"></a>
### Hook Configuration Schema
All Hook extension types must have the type configuration schema set to be invoked during stack operations. There are three parts to the Configuration Schema:

- TargetStacks: This property determines if the hook is `on` (TargetStacks = `ALL`) or off (TargetStacks = `NONE`). If the mode is set to `ALL`, the hook applies to all stacks in your account during a `CREATE`, `UPDATE`, or `DELETE` operation. If the mode is set to `NONE`, the hook won't apply to stacks in your account. Valid values: `ALL` | `NONE`
- FailureMode: This field tells the service how to treat hook failures. If the mode is set to `FAIL`, and the hook fails, then the fail configuration stops provisioning resources. If the mode is set to `WARN` and the hook fails, then the warn configuration allows provisioning to continue with a warning message defined in your Hook logic. Valid values: `FAIL` | `WARN`
- Properties: You can include specific hook runtime properties. These should match the shape of the properties supported by hooks schema. For example, you can define your desired encryption setting (`AES256` or `KMS`) as a property. At runtime, the hook will refer to the configuration property to evaluate against your desired encryption setting.

---

<a name="config"></a>
## Type configuration
AWS CloudFormation supports hook type specific configuration which can be used at set by hook users. CloudFormation service will refer this configuration at runtime when it is executing hook in an account. Hook configuration supports enabling or disabling hook at stack level, failure mode and hook runtime property values.

Below is the example of typical hook configuration:
```json
"{
    "CloudFormationConfiguration": {
        "HookConfiguration": {
            "TargetStacks": "ALL",
            "FailureMode": "FAIL",
            "Properties": {
                ...
            }
        }
    }
}"
```

Hook configuration support below properties:

- TargetStacks:
  - This field will enable hooks at stack level. This field gives users ability to further fine grain control over when to execute hook in a stack. Valid values are `ALL` or `NONE`

- FailureMode:
  - This field tells service how to treat hook failures. Valid values are `FAIL` or `WARN`. If mode is `FAIL` and hook fails, stack operations will be failed too. In case of `WARN` mode, stack operation will be impacted and hook failure will be shown as warning message stack events.

- Properties:
  - Specifies hook runtime properties. These should match the shape of the properties supported by hooks schema. Hook authors define runtime properties in the hook schema. You can identify property definitions and if they are required by examining the hook schema under TypeConfiguration.

- Example: For the above sample hook schema, the configuration json would be

  ```json
  "{
      "CloudFormationConfiguration": {
          "HookConfiguration: {
              "TargetStacks": "ALL",
              "FailureMode": "FAIL",
              "Properties": {"limitSize": "1","encryptionAlgorithm": "aws:kms"
              }
          }
      }
  }"
  ```

---
 
<a name="demo"></a>
##  Demo

Let's get started with the demo.

<a name="aws-kms-demo"></a>
### S3 KMS Demo: 

We are going to enforce a preventive guardrail, where in a user is not allowed to create an S3 bucket without **aws-kms** encryption

1. Create an IAM Role using [CloudFormation Hook Role](https://github.com/aws-cloudformation/aws-cloudformation-samples/blob/main/hooks/python-hooks/s3-bucket-encryption/hook-role.yaml) and note down the ARN

    _IAM_Role_Permission_Policy_
    ![IAM_Role_Permission_Policy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qhg7uxur9selbdmw4y48.png)

    _IAM_Role_Trust_Policy_
    ![IAM_Role_Trust_Policy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2tzqd9c11f8elnc95rk2.png)

2. Activate CloudFormation Hook: Navigate to CloudFormation > Public Extensions > Select **Hooks** under **Extension Type** > Select **Third Party** under **Publisher** and then search for **Extension name prefix = AWSSamples::S3BucketEncrypt::Hook**

    _Activate_Hook_
    ![Activate_Hook](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t7ixkahr5l64yb5rzuf2.png)

    _Hook_Execution_Role_
    > Note: **The execution role is _extremely_ important**. An invalid role will not allow to create any buckets.
    ![Hook_Execution_Role](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/shs0uxx0tziqxw4n8unm.png)

    
    _Hook_Config_

    ```json
    {
      "CloudFormationConfiguration": {
        "HookConfiguration": {
          "TargetStacks": "ALL",
          "FailureMode": "FAIL",
          "Properties": {
            "minBuckets": "1",
            "encryptionAlgorithm": "aws-kms"
          }
        }
      }
    }
    ```
    
    ![Hook_Config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9p7l1l1jt8q62g74fbt0.png)
    
    _Hook_Resources_
    ![Hook_Resources](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bn2us0r4q2bkry5di70d.png)
    
    _Hook_Schema_
    ![Hook_Schema](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/epfx6gcihfv5803qwk0m.png)
    
    _Hook_Config_
    ![Hook_Config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4rfki63q705cv9biu27u.png)
    
    _Hook_Activate_
    ![Hook_Activate](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9csruedqa9tsuw2kzkti.png)

3. Use the following CloudFormaton template to create an unenrypted bucket, The expected result is a failure due to CloudFormation Hook preventive guardrail.

    ```json
    AWSTemplateFormatVersion: "2010-09-09"
    Description: This CloudFormation template provisions an unencrypted S3 Bucket
    Resources:
      S3Bucket:
        Type: 'AWS::S3::Bucket'
        DeletionPolicy: Delete
        Properties:
          BucketName: examplebuketcreatedfromcloudformation
    ```

    _unencrypted-s3_
    ![unencrypted-s3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/489qeyraklytls8djway.png)

4. Use the following CloudFormation Template  to create a S3 bucket with **aws:kms** encrytion, which will allow the bucket creation.

    ```json
    AWSTemplateFormatVersion: "2010-09-09"
    Description: This CloudFormation template provisions an encrypted S3 Bucket
    Resources:
      EncryptedS3Bucket:
        Type: 'AWS::S3::Bucket'
        Properties:
          BucketName: !Sub 'encryptedbucket-kms-${AWS::Region}-${AWS::AccountId}'
          BucketEncryption:
            ServerSideEncryptionConfiguration:
              - ServerSideEncryptionByDefault:
                  SSEAlgorithm: 'aws:kms'
                  KMSMasterKeyID: !Ref EncryptionKey
                BucketKeyEnabled: true
          Tags: 
            - Key: "keyname1"
              Value: "value1"

      EncryptionKey:  
        Type: AWS::KMS::Key
        Properties:
        Description: KMS key used to encrypt the resource type artifacts
        EnableKeyRotation: true
        KeyPolicy:
          Version: "2012-10-17"
          Statement:
          - Sid: Enable full access for owning account
            Effect: Allow
            Principal: 
              AWS: !Ref "AWS::AccountId"
            Action: kms:*
            Resource: "*"

    Outputs:
      EncryptedBucketName:
        Value: !Ref EncryptedS3Bucket
    ```

    _encrypted-s3-kms_
    ![encrypted-s3-kms](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w8zf6skuo9pfrslxlkiz.png)

---

<a name="sse-s3-demo"></a>
### S3 SSE-S3(AES256) Demo: 

We are going to enforce a preventive guardrail, where in a user is not allowed to create an S3 bucket without **SSE-S3(AES256)** encryption.

1. Use the following CloudFormation Template to create a S3 bucket with **AES256** encrytion

    ```json
    AWSTemplateFormatVersion: "2010-09-09"
    Description: This CloudFormation template provisions an encrypted S3 Bucket
    Resources:
      EncryptedS3Bucket:
        Type: 'AWS::S3::Bucket'
        Properties:
          BucketName: !Sub 'encryptedbucket-s3-${AWS::Region}-${AWS::AccountId}'
          BucketEncryption:
            ServerSideEncryptionConfiguration:
              - ServerSideEncryptionByDefault:
                  SSEAlgorithm: 'AES256'
                BucketKeyEnabled: true
          Tags: 
            - Key: "keyname1"
              Value: "value1"
    ```

    _encrypted-s3-AES256-failed_
    ![encrypted-s3-AES256-failed](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vew6f8cftt6fluip5717.png)
    

2. In order to fix this, we need to edit the CloudFormation Hook config to update **encryptionAlgorithm** from **aws:kms*** to **AES256** and re-deploy the CloudFormation Template in Step 1 of Demo 2, which will allow the bucket creation.

    _edit-cf-hook_

    ```json
    {
      "CloudFormationConfiguration": {
        "HookConfiguration": {
          "TargetStacks": "ALL",
          "FailureMode": "FAIL",
          "Properties": {
            "minBuckets": "1",
            "encryptionAlgorithm": "AES256"
          }
        }
      }
    }
    ```

    ![edit-cf-hook](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x8ebtotp14iumgpgzt9v.png)
    
    _encrypted-s3-AES256_
    ![encrypted-s3-AES256](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cofo9kmxktg4lum8we70.png)
    
---

<a name="cleanup"></a>
### Clean Up

1. Deactive CloudFormation Hook.
    
    _Deactivate_Hook_
    ![Deactivate_Hook](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hc2xvivmk39u9r8fs1g1.png)
    
    _Final_Hook_
    ![Final_Hook](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/oepbgg04rjrp2nhkdoxa.png)
    
2. Delete all S3 buckets CloudFormation Stacks.

3. Delete IAM Role CloudFormation Stack.

---

<a name="summary"></a>
### Summary
- We have learnt how to enable CloudFormation hooks to use as a preventive guardrails.

- To get started, you can explore sample hooks published to the [CloudFormation Public Registry](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/registry-public.html) or author Hooks using the [CloudFormation CLI](https://github.com/aws-cloudformation/cloudformation-cli) and publish them to your [CloudFormation Private Registry](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/registry.html). 

- The registry provides a central location where you can browse CloudFormation extensions, such as resources, modules, and hooks that are available for use in your account. You can also refer to [sample Hooks](https://github.com/aws-cloudformation/aws-cloudformation-samples/tree/main/hooks).

---

<a name="referrals"></a>
### Referrals:

- [AWS CloudFormation Hook workshop](https://catalog.us-east-1.prod.workshops.aws/v2/workshops/f09fd78b-ef8a-4a9d-9d2b-f31a3e6ca956/en-US)

- [S3 bucket creation with encryption is failing because of AWSSamples::S3BucketEncrypt::Hook](https://stackoverflow.com/questions/71333656/s3-bucket-creation-with-encryption-is-failing-because-of-awssampless3bucketenc)

---

