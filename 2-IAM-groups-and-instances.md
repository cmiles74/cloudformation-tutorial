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

We use the [IAM Group resource][35] to create our new group. The only property that we provide is the `Policies` property that contains a list of `PolicyName` and `PolicyDocument` pairs, in this case we define only one that we have named "TutorialAdminPolicy".

A `PolicyDocument`contains a `Statement` that holds a list of `Effect` instances; each of those in turn contains an `Action` property with a list of permissions. We use the `Resource` property to tie in a reference to our bucket, in this case the backup bucket's ARN. If we take a look at the first `Effect`, you'll see that we've assigned four "ListBucket..." permissions for our backup bucket. The second effect applied three more actions to every file in the backup bucket, that's indicated by the `/*` at the end of the bucket's ARN.

Any account that we add to this group will be able to download or delete any of the files in the bucket (as well as upload). In this case when we say "files" we really mean database dumps for this project. They will also be able to pull up the S3 console for the bucket but they will need a link that is _directly to the bucket_. They will not be able to log into S3 and browse the list of all buckets.

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

### Setup a Role for Our Instances

Next we need to setup the role that our instances can assume to get access to resources (for now, just the backup bucket).

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

* the first lets the instance run the [CloudWatch agent][52] and submit events to the [CloudWatch service][37]
* the next lets the instance interact with [Systems Manager][38], letting us manage the instances as a group
* The last provides read-only access to the Systems Manager parameter store.

Amazon's CloudWatch service will aggregate data submitted by your instances and let you setup reports, dashboards and alerts based on this data (i.e. when CPU usage gets too high or available disk space is too low). Systems Manager provides some tools for managing your instances, like installing patches or a software package. It lets you perform these actions on several instances as a group which can be handy. I'm not going to go over these services in-depth here but I encourage you to spend some time checking them out if you haven't done so already.

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

We are creating a new [Policy resource][39] that we will assign to our instances. We link the role we just created and then add a new policy that provides access to the backup bucket. As you can see, it is exactly the same as the role that we created for our administrators group.

The last piece of this puzzle is an "instance profile" that we can assign to our instances.

```yaml
  TutorialInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref TutorialInstanceRole
```

With all of the other work done, there's not much to see here. We create a new [InstanceProfile resource][40] and link it to our role. When we provision instances we can assign them this profile, they will then have the ability to write to our backup bucket, submit performance and event data to CloudWatch and accept remote commands from System Manager.

### Provision CloudWatch Log Groups

The very last thing we need before provisioning are a couple CloudWatch [log groups][50]. We can instruct our instances to ship some of their log files out to these groups and then view the logs through the CloudWatch console.

```yaml
  TutorialBootLog:
    Type: AWS::Logs::LogGroup

  TutorialKernelLog:
    Type: AWS::Logs::LogGroup
```

We use the [LogGroup][51] resource to create two new log groups. These groups will be names after the name we provide in the template with some random characters tacked on at the end. We're creating the groups now because we're going to reference them directly when we provision the instances.

### Provision the Database Server

I know what you are thinking: all that work and we're only now provisioning our instances! It's true, there was a lot of preparation and things to think about but we are well on our way to having a template that will provision our servers in a repeatable and reasonably safe manner. Just think of all the templates you will be writing!

We're going to provision the database server first. This one is a little bit involved so I'm going to break it up into pieces. First we provision a new instance with the [Instance resource][41]...

```yaml
  TutorialDatabaseServer:
    Type: AWS::EC2::Instance
    Metadata:
      AWS::CloudFormation::Init:
        config:
          files:
            /etc/profile.d/cloudformation-init.sh:
              content: !Sub |
                export BACKUP_S3_BUCKET="${TutorialBackupS3Bucket}"
```

Note that in the code stanza above I stop before the properties, we'll go over the properties next. 

The first thing we do is setup the "metadata" for the CloudFormation initialization script (we will call that at the very end) with the [CouldFormation Init type][42]. This is were we tell the initialization script what we would like it to do when it runs, in this case we add a new environment variable to a new file in `/etc/profile.d` called `cloudformation-init.sh`. Whenever someone logs into the machine this script (along with everything else in the directory) will be evaluated, the end result will be that the name of our backup bucket will be available through the `BACKUP_S3_BUCKET` environment variable. Keep in mind that this metadata doesn't do anything on it's own: the initialization script will use it when it runs and we'll do that at the end of our instance definition.

We also need to provide the configuration file for the CloudWatch agent. This agent will collect performance data and log files and ship them back to the CloudWatch service.

```yaml
            /etc/amazon/amazon-cloudwatch-agent.json:
              content: !Sub |
                {
                  "metrics": {
                    "append_dimensions": {
                      "AutoScalingGroupName": "${!aws:AutoScalingGroupName}",
                      "ImageId": "${!aws:ImageId}",
                      "InstanceId": "${!aws:InstanceId}",
                      "InstanceType": "${!aws:InstanceType}"
                    },
                    "metrics_collected": {
                      "cpu": {
                        "measurement": [
                          "cpu_usage_idle",
                          "cpu_usage_iowait",
                          "cpu_usage_user",
                          "cpu_usage_system"
                        ],
                        "metrics_collection_interval": 30,
                        "totalcpu": false
                      },
                      "disk": {
                        "measurement": [
                          "used_percent",
                          "inodes_free"
                        ],
                        "metrics_collection_interval": 30,
                        "resources": [
                          "*"
                        ]
                      },
                      "diskio": {
                        "measurement": [
                          "io_time"
                        ],
                        "metrics_collection_interval": 30,
                        "resources": [
                          "*"
                        ]
                      },
                      "mem": {
                        "measurement": [
                          "mem_used_percent"
                        ],
                        "metrics_collection_interval": 30
                      },
                      "statsd": {
                        "metrics_aggregation_interval": 30,
                        "metrics_collection_interval": 10,
                        "service_address": ":8125"
                      },
                      "swap": {
                        "measurement": [
                          "swap_used_percent"
                        ],
                        "metrics_collection_interval": 30
                      }
                    }
                  },
                  "logs": {
                    "logs_collected": {
                      "files": {
                        "collect_list": [
                          {
                            "log_group_name": "${TutorialBootLog}",
                            "file_path": "/var/log/boot.log"
                          },
                          {
                            "log_group_name": "${TutorialKernelLog}",
                            "file_path": "/var/log/messages"
                          }
                        ]
                      }
                    }
                  }
                }
```

This configuration file tells the CloudWatch agent to ship back data on CPU, disk, memory and swap file usage back to CloudWatch as well as two log files: the boot and kernel logs.

Next we'll set the properties for our instance...

```yaml
    Properties:
      ImageId: !Ref ImageId
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyPairName
      NetworkInterfaces:
        - SubnetId: !Ref TutorialPrivateSubnet
          DeviceIndex: 0
          GroupSet:
            - !Ref TutorialPrivateSecurityGroup
      BlockDeviceMappings:
        - DeviceName: !Sub "/dev/${RootDevice}"
          Ebs:
            VolumeSize: !Ref VolumeSize
            VolumeType: gp2
      IamInstanceProfile: !Ref TutorialInstanceProfile
      Tags:
        - Key: "Name"
          Value: "Tutorial Database Server"
```

Way back in part 1 we create these parameters and provided default values for them, now they are finally coming into play. If you look back at the top of the template, you'll see that we set parameters for the image, instance type, key pair name and root volume. While our template requires that a key pair name be provided the other values are populated with default values. My expectation is that the default values will normally be used, most of the reason they are at the top in the parameter section is to make them easy to get at.

The first thing we do is choose the image we'd like for this instance, in the code above we have a reference to the parameter, the default value is for the [Amazon Linux 2][43] image. There are other Linux images out there, I picked this one mostly because it already has the AWS tools and CloudFormation stuff installed, making things that much easier. Also kind of interesting, if you develop locally you can work with an [Amazon Linux 2 Docker Container][44] on your workstation.

Next we use our instance type parameter, the default values is for `t2.micro` because this is a tutorial and I don't want to cost you money; this type is eligible for use under [Amazon's free tier][45]. If you haven't used up your 750 hours of free tier usage for the month you shouldn't be charged for provisioning these instances. Setting cost aside, a micro instance is likely enough to host a low traffic website, like your personal blog.

We set the name of the key pair we want to use to provision our instance, note that the value we reference in the `KeyName` property is the one parameter we set for this script back in part 1. You will need to have this key pair handy in order to SSH to the instance.

We'd like the database server to be on our private subnet and we take care of that when we specify the `NetworkInterfaces` property. Here we pass in a reference to our private subnet, we then set the security group to our "private" security group with the `GroupSet` property of the network interface.

Every instance needs disk space and we can customize the [Elastic Block Store][46] (EBS) volumes for our instance with the `BlockDeviceMappings` property. We map in one volume of 250GB (referencing our parameter) to the root device by using the substitution function with our parameter to set the value to  `/dev/xvda`, we chose the "general purpose" (gp2) volume type.

The default volume type is "gp2" and it's a reasonable choice, more information about the various types may be found in the [EBS volume type documentation][47]. Also note that the root device that the instance boots from may vary from one Linux distribution to the other. For instance, under Ubuntu the root volume needs to be mapped to `/dev/sda1`; if the instance can't find the volume you will see it start up in the EC2 console but it will stop in just a couple of minutes.

Instead of managing credentials for our EC2 instances we thoughtfully created a profile! We set the profile for our instance with the `IamInstanceProfile` property and pass in a reference to that profile.

Lastly we set tags on our instance, in this case we set the "Name" tag.

With that out of the way the only thing left to do is to invoke the CloudFormation initialization script when the instance starts up. To get this done we write a little script that calls `cfn-init` at start time.

```yaml
      UserData: 
        Fn::Base64: 
          !Sub |
            #!/bin/bash -xe

            # cloudformation initialize
            /opt/aws/bin/cfn-init -v -s ${AWS::StackName} --region ${AWS::Region} -r TutorialDatabaseServer

            # download and install cloudwatch agent
            wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
            yum -y install amazon-cloudwatch-agent.rpm
            mv /etc/amazon/amazon-cloudwatch-agent.json /etc/amazon/amazon-cloudwatch-agent/amazon-cloudwatch-agent.json
            sudo systemctl enable amazon-cloudwatch-agent
            sudo systemctl start amazon-cloudwatch-agent
```

Documentation for `cfn-init` is [available on the CloudFormation site][48]. You can see in the example above that we call the script with the name of our stack, the stack's deployment region and the name of our server. When the script runs it will inspect the `Metadata` property that we set at the the beginning of our instance stanza and will carry out those tasks. For this instance, the initialization script will add that new file to `/etc/profile.d` with our custom environment variables.

The `cfn-init` script handles getting our configuration files written to disk, the other thing we need to do is get the CloudWatch agent installed and running. The next bit of script downloads the current version, installs it, moves our configuration file into place and then enables and starts the service so that the agent is started at boot time.

And that's it... For our database server. We need to do pretty much the same thing for our web server.

```yaml
  TutorialWebServer:
    Type: AWS::EC2::Instance
    Metadata:
      AWS::CloudFormation::Init:
        config:
          files:
            /etc/profile.d/cloudformation-init.sh:
              content: !Sub |
                export BACKUP_S3_BUCKET="${TutorialBackupS3Bucket}"
                export DATABASE_SERVER="${TutorialDatabaseServer.PrivateIp}"
```

Mostly everything is exactly the same but there are a couple small differences. When we setup the `Metadata` for the initialization script we added another environment variable that contains the private IP address of the database server, in this way we can deploy this template more than once and the web server in each stack will know where to find it's matching database server.

I don't include it in this article, but we provide the exact same configuration file for the CloudWatch agent that we used for the web server. If you are writing the template along with this article, take a minute to copy and paste that stanza into the file now.

```yaml
    Properties:
      ImageId: !Ref ImageId
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyPairName
      NetworkInterfaces:
        - SubnetId: !Ref TutorialPublicSubnet
          DeviceIndex: 0
          GroupSet:
            - !Ref TutorialPublicSecurityGroup
      BlockDeviceMappings:
        - DeviceName: !Sub "/dev/${RootDevice}"
          Ebs:
            VolumeSize: !Ref VolumeSize
            VolumeType: gp2
      IamInstanceProfile: !Ref TutorialInstanceProfile
      Tags:
        - Key: "Name"
          Value: "Tutorial Web Server"
      UserData: 
        Fn::Base64: 
          !Sub |
            #!/bin/bash -xe

            # cloudformation initialize
            /opt/aws/bin/cfn-init -v -s ${AWS::StackName} --region ${AWS::Region} -r TutorialDatabaseServer

            # download and install cloudwatch agent
            wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
            yum -y install amazon-cloudwatch-agent.rpm
            mv /etc/amazon/amazon-cloudwatch-agent.json /etc/amazon/amazon-cloudwatch-agent/amazon-cloudwatch-agent.json
            sudo systemctl enable amazon-cloudwatch-agent
            sudo systemctl start amazon-cloudwatch-agent
```

In terms of the properties, the only difference is that we have placed our web server in the public subnet so that it can communicate with the public internet (as web servers so often need to do).

At this point we have covered the first four goals that we laid out for ourselves in part 1. With the template as it stands right now, we can...

* Provision resources for a project
* Retire resources for a project
* Isolate resources for a project
* Report on resources by project

I think this is a pretty significant milestone! From here we can tweak the template to match a particular project and with just one command setup the environment. If we have a client that wants a "test" and a "production" environment, all we need to do is provision the template twice.

## Reporting on Resources

This is a little outside the realm of CloudFormation, but I thought we'd take a short break from the template and take a look at some ways we can use the tags we have so diligently been placing on our resources. Take a look at the [billing console][49] for your account. You will be dropped at the dashboard and some summary data for your account will be visible.

If you haven't done so already, click on the "Cost Explorer" link from the left-hand navigation bar and enable it for your account. It's a handy tool that lets you do some simple querying and reporting on your resources and their costs.

Next click on the "Cost Allocation Tags" link, also on the left-hand navigation bar. At the top you can see that there are some cost allocation tags that AWS can generate on it's own, go ahead and click the "Activate" button to turn those on. Lower down on the page will be a list of all of the tags that we have defined (as well as any tags you had already setup). You can choose which tags you want to make available in the Cost Explorer but my advice is just to click the top-most checkbox and make them all active. You never know when a tag will come in handy! Click the "Activate" button and confirm that, yes, you want the tags to be "active".

Now click back to the "Cost Explorer" link and click on "Launch Cost Explorer". If you just activated the cost explorer now, you will probably have to wait until tomorrow; keep it in mind and try to come back to this section.

If the Cost Explorer has data on hand then you will be presented with another dashboard attempting to summarize your costs on amazon. Click on the magnifying glass icon on the left-hand navigation bar and choose "Cost and Usage", the report builder will appear with your last six months of spending. Click on the "Last 6 Months" pull-down and choose "1M" from the "Historical" section at the bottom, this will show your last month of spending. 

On the right-hand side is a list of filters and this is where the tags will come in handy. Click on the "Tag" link in the filter list, a list of your tags will appear; click on "Project" and a list of all your values for the "Project" tag will be listed. In this tutorial we've been putting "cf-tutorial" in the "Project" tags, check the box for "cf-tutorial" and then press the "Apply filters" button to update the report.

What you are looking at now is likely a very dull report, because we've been using low-cost free-tier instances. But, still, what we have is a report on just the resources that belong to this project. If you were to place a "Client" tag in your templates you could report on the entire cost of a client (maybe you could use that to figure out how to bill) and you can break down a client's costs by particular projects. It is a valuable tool and definitely worth putting some time in exploring your options.

## Set Alarms on Instance Usage

I think of this as related to our reporting goals: we'd like to set some alarms that monitor the data we're sending to CloudWatch to let us know if our servers start to go off the rails. We'll set alarms for when CPU usage gets high or if we start to run out of disk space but I bet you can think of lots of other things you'd like to monitor.

We're going to use the [Simple Notification Service][53] to alert us when an alarm is hit. The way this will work is that when CloudWatch sees an alarm trigger, it will send a message to the SNS "topic". It will be our responsibility to log into the SNS console and add ourselves to the notification list (by email or text message), we aren't going to populate the list of subscribers.

```yaml
  TutorialAlarmTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: TutorialAlarms
```

We use the [Topic][54] resource to create our new alarm target topic. With the SNS topic created we can now create our first alarm.

```yaml
  TutorialDatabaseCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Database Server CPU Usage High
      AlarmActions:
        - !Ref TutorialAlarmTopic
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 60
      EvaluationPeriods: 3
      Threshold: 89
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: InstanceId
          Value: !Ref TutorialDatabaseServer
```

With the [Alarm][55] resource we create an alarm for CPU usage on the database server, if the server is using more than 89% of the CPU for more than 3 minutes then the alarm will trigger and send a message to our SNS topic.

```yaml
  TutorialDatabaseDiskAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Database Server Disk Usage High
      AlarmActions:
        - !Ref TutorialAlarmTopic
      MetricName: disk_used_percent
      Namespace: CWAgent
      Statistic: Average
      Period: 60
      EvaluationPeriods: 3
      Threshold: 89
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: InstanceId
          Value: !Ref TutorialDatabaseServer
        - Name: ImageId
          Value: !Ref ImageId
        - Name: InstanceType
          Value: !Ref InstanceType
        - Name: path
          Value: /
        - Name: device
          Value: !Sub ${RootDevice}1
        - Name: fstype
          Value: !Ref RootFsType
```

This next alarm is for the amount of disk used by the database server, here we set an alarm to trigger if more than 89% of the disk is in use. This one will also wait for the disk to be in this state for 3 minutes before triggering the alarm.

Provisioning this alarm is a bit more work, you can see that we had to add several `Dimensions` to get the alarm setup. This is because when the CloudWatch agent reports this data it references all of these dimensions and we need to match them all in order to have a working alarm. If we had left one of these out then the alarm wouldn't match any data and would never trigger.

The only bit left is the backup jobs but we'll leave that for part 3. `;-)` Congratulations on working your way through all of this material! While it can be pretty dry reading you can really get a lot of mileage building templates for projects that you know you will be deploying over and over.

------
[35]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-iam-group.html
[36]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html
[37]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
[38]: https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html
[39]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-policy.html
[40]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-instanceprofile.html
[41]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html
[42]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-init.html
[43]: https://aws.amazon.com/amazon-linux-2/
[44]: https://hub.docker.com/_/amazonlinux/
[45]: https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-limits.html
[46]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEBS.html
[47]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html
[48]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-init.html
[49]: https://console.aws.amazon.com/billing/home
[50]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html
[51]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-logs-loggroup.html
[52]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html
[53]: https://aws.amazon.com/sns/
[54]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-sns-topic.html
[55]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cw-alarm.html

