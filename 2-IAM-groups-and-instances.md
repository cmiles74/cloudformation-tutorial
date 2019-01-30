---
layout: post
title: Provision Pragmatically and Programmatically with CloudFormation, 2
---

![CloudFormation Diagram](https://raw.githubusercontent.com/cmiles74/cloudformation-tutorial/master/template-diagram-2.png "Diagram of our CloudFormation Tutorial Stack")

This is the second part in a three part series where we build up a CloudFormation template for what I think of as a pretty typical environment: one virtual private cloud broken into two subnets and including two instances. If you haven't read the first part in the series, I encourage you to check it out now!

## Setup an "Admininstrators" Group

For this project we're creating a general "administrators" group that has access to the backup bucket with our database backups. I generally don't allow anyone direct access to the EC2 console unless they really need it, so I don't address that in this template (in addition, not every EC2 web console action can be managed this way). My thinking here is that we typically give "administrators" or "developers" accounts on the individual instances, they don't really need the EC2 web console in order to get their work done.

```yaml
  TutorialAdminGroup:
    Type: AWS::IAM::Group
    Properties:
      Policies:
        - PolicyName: TutorialAdminPolicy
          PolicyDocument:
            Statement:
            - Effect: Allow
              Action:
                - s3:ListBucket
                - s3:ListBucketByTags
                - s3:ListBucketMultipartUploads
                - s3:ListBucketVersions
              Resource: !Sub ${TutorialBackupS3Bucket.Arn}
            - Effect: Allow
              Action:
                - s3:PutObject
                - s3:DeleteObject
                - s3:GetObject
              Resource: !Sub ${TutorialBackupS3Bucket.Arn}/*
```

We use the [IAM Group resource][35] to create our new group. The only property that we provide is the `Policies` property that contains a list of `PolicyName` and PolicyDocument` pairs, in this case we define only one that we have named "TutorialAdminPolicy".

A `PolicyDocument`contains a `Statement` that holds a list of `Effect` instances; each of those in turn contains an `Action` property with a list of permissions. We use the `Resource` property to tie in a references to our bucket, in this case the backup bucket's ARN. If we take a look at the first `Effect`, you'll see that we've assigned four "ListBucket..." permissions for our backup bucket.

Any account that we add to this group will be able to inspect and download the files in the bucket. By files, we mean database dumps for this project.

One last note: since we are creating an IAM role with our template, we need to let CloudFormation know that this is okay. From here on out we need to add the `--capabilities` flag to our `aws` commands.

```shell
aws cloudformation create-stack \
  --stack-name tutorial \
  --template-body file://template.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=cloudfront-tutorial \
  --tags Key=Project,Value=cf-tutorial \
  --capabilities CAPABILITY_IAM
```

If you don't include the tag, CloudFormation will exit with an error. The same flag can be used with "update" commands.

## Provision Our Instances

I know what you are thinking: all that work and we're only now provisioning our instances! It's true, there was a lot of preparation and things to think about but we are well on our way to having a template that will provision our servers in a repeatable and reasonably safe manner. Just think of all the templates you will be writing!

Oh, wait, I forgot about the roles and policies for the instances. Sorry!

### Setup a Role for Our Instances

First we need to setup the role that our instances can assume to get access to resources (for now, just the backup bucket).

```yaml
  TutorialInstanceRole:
    Type: AWS::IAM::Role
    Properties:
      ManagedPolicyArns:
        - arn:aws-us-gov:iam::aws:policy/CloudWatchAgentServerPolicy
        - arn:aws-us-gov:iam::aws:policy/service-role/AmazonEC2RoleforSSM
        - arn:aws-us-gov:iam::aws:policy/AmazonSSMReadOnlyAccess
      AssumeRolePolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
```

We use the [Role resource][36] to create our new instance role and we assign three canned policies...

* the first lets the instance run the CloudWatch agent and submit events to the [CloudWatch service][37]
* the next lets the instance interact with [Systems Manager][38], letting us manage the instances as a group
* The last provides read-only access to the Systems Manager parameter store.

Amazon's CloudWatch service will aggregate data submitted by your instances and let you setup reports, dashboards and alerts based on this data (i.e. when CPU usage gets too high or available disk space is too low). Systems Manager provides some tools for managing for instances, like installing patches or a software package. It lets you perform these actions on several instances as a group, which can be handy. I'm not going to go over these services in-depth here but I encourage you to spend some time checking them out if you haven't done so already.

```yaml
  TutorialInstanceRolePolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: TutorialInstanceRolePolicy
      Roles:
        - Ref: TutorialInstanceRole
      PolicyDocument:
        Statement:
          - Effect: Allow
            Action:
              - s3:ListBucket
              - s3:ListBucketByTags
              - s3:ListBucketMultipartUploads
              - s3:ListBucketVersions
            Resource: !Sub ${TutorialBackupS3Bucket.Arn}
          - Effect: Allow
            Action:
              - s3:PutObject
              - s3:DeleteObject
              - s3:GetObject
            Resource: !Sub ${TutorialBackupS3Bucket.Arn}/*
```

We are creating a new [Policy resource][39] that we will assign to our instances. We link the role we just creates and then add a new policy that provides access to the backup bucket. As you can see, it is exactly the same as the role that we created for our administrators group.

The last piece of this puzzle is an "instance profile" that we can assign to our instances.

```yaml
  TutorialInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref TutorialInstanceRole
```

With all of the other work done, there's not much to see here. We create a new [InstanceProfile resource][40] and link it to our role. When we provision instances we can assign them this profile, they will then have the ability to write to our backup bucket, submit performance and event data to CloudWatch and accept remote commands from System Manager.

### Provision the Database Server

We're going to provision the database server first so that we can tell the web server a little bit about it.

```yaml

```

------
[35]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-iam-group.html
[36]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html
[37]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
[38]: https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html
[39]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-policy.html
[40]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-instanceprofile.html
