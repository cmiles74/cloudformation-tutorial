# Provision Pragmatically and Programmatically with CloudFormation

![CloudFormation Diagram](https://raw.githubusercontent.com/cmiles74/cloudformation-tutorial/master/template-diagram.png
"Diagram of our CloudFormation Tutorial Stack")

How many times have you gazed upon the list of services that Amazon is providing through their Amazon Web Services (AWS) family of products and felt an unpleasant mixture of anxiety and confusion? If you're anything like me, it would be literally _every time_ you visited their website. While I was one of the first in my circle of software developer friends to get on board the Elastic Compute Cloud (EC2) train, I haven't been great at keeping up with new developments. This year, I decided, would be the year that I get back on track and sort out this alphabet soup of cloud virtualization technology.

After clicking around at random and reading an unwholesome amount of bland introductory pages, I finally started to get an idea of how I should try to put all of these pieces together. The first thing I figured out:

> It is way, way too hard to get everything up and running by clicking a bunch
> of buttons.

What are your goals? For me, it's to be able to deploy a web-based application for one of my clients. As a corollary, I want to be able to entirely scrap a web-based application for one of my clients (maybe it's been replaced, maybe they've hired another consultant). In effect, that's two goals. Plus I would like to keep all of my clients separate, I don't want changes to a firewall rule to effect all of my clients, just the one client who's willing to take the risk. If you think about it, keeping things isolated will also make it easier to figure out who we can charge for what services at the end of the month; we should also tag these resources in a way that makes sense. That's four goals and that's only what we've thought of so far. And we should snapshot on a daily basis and maybe not keep all of the snapshots. Oh, backing up the database is important, too!

* Provision resources for a project
* Retire resources for a project
* Isolate resources for a project
* Report on resources by project
* Snapshot instance volumes and delete old backups on a schedule
* Backup the databases on a schedule and delete old backups on a schedule

Looking at these goals, the number of services, the way many of them build on each other... It all becomes clear that the answer is automation. Of course, Amazon provides an automation tool: [CloudFormation][0].

This is a first in a series of articles that will cover all of these goals. We'll start out with an empty file and we'll gradually build that file up, adding more and more resources to that file as we go along. I am planning three articles all together, this on covers a brief introduction to the tools and gets as far as setting up a virtual private cloud for the project, splitting it into two networks and putting security groups in place. So exciting!

I encourage you to follow along as we build up the template but you can always checkout the finished product [over at GitHub][35].

## Install the AWS Command Line Tools

Amazon provides a goofy drag-and-drop interface for CloudFormation but that runs kind of counter to the whole point of this article. You can also write your CloudFormation template and then upload it via their web interface, I think that's also a little clunky. My recommendation is that you install the [AWS Command Line Tools][1] and use those tools to deploy. Amazon also provides packages for Windows, but if you're working from within the Windows environment you might be more interested in the [AWS Tools for PowerShell][2] which do much the same thing but, you know, with [PowerShell][3].

This is kind of a power tip, but if you're managing more than one Amazon account or dealing with more than one region, you can [configure profiles][4] to make that process easier.

With one of these tools installed and properly configured, you can invoke CloudFormation directly from your console window! `:-)`

## Start Our New CloudFormation Template

In the world of CloudFormation, the document you are writing is called a "template". I believe it's referred to as a template because one template can be used to create many different sets of resources. For instance in this tutorial we're going to be writing a template that will provision a web server and a database server; we can use the template to provision as many of these web-and-database-server pairs as we might like, all independent of one another.

You can write your template in [YAML][5] or [JSON][6], if you're writing it by hand (as we are right now) then I _strongly_ recommend that you write it in YAML. I know, I know, YAML isn't everyone's favorite but getting all of the ending braces and brackets lined up in a JSON document is it's own kind of mental torture. You deserve better, trust me: don't do that to yourself.

Anyway, we're going to start off with a description and a parameter. A parameter, as you may have suspected, is a value that we can supply when we are actually using our template to provision something.

```yaml
Description: Provisions a simple cluster with a web and a database server

Parameters:
  KeyPairName:
    Type: String
    Description: Name of keypair used when provisioning instances
```

Throughout this article we'll continue to build up this template but if you need to jump ahead or need to refer to the entire thing, you can [browse through it in the GitHub project][14].

Some decisions are best postponed until the absolute last minute and parameters are here when you have decisions to postpone. When we decide to provision our resources, we can provide the name of the EC2 key pair to use at that time.

I'm not going to go into a big discussion about how to manage your EC2 key pairs. Maybe you have one per person who has access to provision instances. Maybe you have one per client. However you manage it, you can pass it in when you provision resources.

## Provision a New Virtual Private Cloud

We need at least one resource in order to provision our template. For this project we're going to create our own [Virtual Private Cloud][8] (VPC) to hold our resources. This will keep things isolated from everything else you have might have deployed to AWS.

```yaml
Resources:

  TutorialVPC:                   # name of the resource
    Type: AWS::EC2::VPC          # type of resource
    Properties:                  # properties of the resource
      CidrBlock: "10.6.0.0/24"
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: "Name"
          Value: "Tutorial VPC"
```

The `Resources` section of the template will contain a stanza for each of the resources we provision, this will be the bulk of the template. Each stanza in this section will start with the name of the resource (`TutorialVPC`), we can use this name when we need to reference the resource while creating other resources (for instance, when we add subnets to this VPC). After the name we need to declare the `Type` of resource: every CloudFormation resource has a type, they are all clearly documented in the [CloudFormation API Documentation][7]. In the example above we are creating a new [VPC][9] instance. Next we set the `Properties` for our resource, the values provided here will be particular to each resource. Most resources accept a `Tags` property but not all, if you set `Tags` on a resource that doesn't support that property CloudFormation will throw an error.

Every VPC needs a block of IP addresses, we set this with the `CidrBlock` property and provide [a non-routable network with 254 available addresses][10]. I chose a network that won't conflict with the address range I use at home or at work, this is important if you ever want to use [a Virtual Private Network (VPN) to connect your Amazon VPC to your own network][11], perhaps to make access to these resources easier.

We'd like Amazon to continue to assign names to our instances, so we set the `EnableDnsSupport` and `EnableDnsHostnames` properties to true. Lastly we set some tags on our resources to remind ourselves why we provisioned them in the first place.

## Provision Resources With Our Template

With our template set, we're ready to use it to provision some resources! When we provide the template to CloudFormation, we also have to provide our parameter values and we have the opportunity to tag _all_ of the provisioned resources (maybe be client or project or both).

Go ahead and run the command below to being provisioning. When CloudFormation sets up all of the resources it links them together into a "stack". Once complete, you can deal with the stack as one unit. The stack name provides a human-friendly handle for all of the related resources.

```shell
aws cloudformation create-stack \
  --stack-name tutorial \
  --template-body file://template.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=cloudfront-tutorial \
  --tags Key=Project,Value=cf-tutorial
```

Remember to provide your own key pair name where we have used `cloudfront-tutorial`.

As soon as CloudFormation receives the template it will return a new `StackId` with the [Amazon Resource Name][15] (ARN) for the stack, you will have to wait a bit for the stack to finish being provisioned. You can check on your stack through the [CloudFormation web console][12] and see how things are going. Some resources take longer than others, CloudFormation will ensure they are created in the right order and work out the dependencies. This template should move along pretty quickly as we aren't provisioning much.

I broke the command over several lines to try and make it easier to follow. We...

* Ask the AWS tools to call out to CloudFormation with the "create-stack" command
* Set the name of the stack we are creating to "tutorial"
* Provide our template to CloudFormation with the `file://` URL
* We set the one template parameter, `KeyPairName` (note the weird way we have to set parameters)
* We provided a one tag key and value that CloudFormation will use to tag all of the provisioned resources
  
Multiple parameters you can separate them with a space, multiple tags are separated with a comma. It's weird, I don't know why they aren't uniform.

When provisioning is complete, you can head over to the [VPC web console][13] to inspect your new virtual cloud. There's not a lot to see, but if you choose "Yourq VPCs" from the left-hand navigation bar and then select your "Tutorial VPC" from the list, you can click the "Tags" tab and look at the tags associated with the resource. You'll see the tag we specified in the template (`Name`) and the tag we passed into the `create-stack` command (`Project`). CloudFormation also set several tags of it's own, linking the resource to the stack.

As we work through the template we might update the stack or delete it entirely. The command to update the stack is almost exactly the same.

```shell
aws cloudformation update-stack \
  --stack-name tutorial \
  --template-body file://template.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=cloudfront-tutorial \
  --tags Key=Project,Value=cf-tutorial
```

If CloudFormation runs into any problems while creating or updating your stack, it will roll back to the last valid state of the stack. If this happens during the creation of the stack you'll end up with no resources at all, if it happens during an update then it will be as if your update attempt never occurred.

Deleting the stack is quite a bit shorter.

```shell
aws cloudformation delete-stack --stack-name tutorial
```

Go ahead and run that command now to delete the stack. The AWS tool command won't return any results but if you switch to the CloudFormation console you might catch a glimpse of the stack before it's entirely deleted. If you switch over to the VPC console you'll see the new VPC is gone, it was deleted along with the stack.

In my opinion, this is a really nice way to work. You can edit your template and work on getting all of the bits and pieces of your stack created and linked together in a way that makes sense (provisioned in the VPC, in the correct subnet, with the correct security group and policies, etc.) When you feel pretty good about your work you can go ahead and provision. If there's a problem or something isn't lining up the way it should you can simply delete the stack and continue to work on your template. For me, this is far more convenient then clicking through the various web consoles and trying to link everything up by filling out fields or applying actions to all of the various pieces.

### Retaining Resources through Stack Updates and Even Deletion

As much as CloudFormation can, it will try to preserve resources between updates. On the other hand there are some changes that can't be made after provisioning: in those cases CloudFormation will delete the resource and create a replacement. For many things this doesn't really matter, but for things like instances or Elastic IP addresses you may really _need_ to retain those resources. The [CloudFormation Documentation][7] is pretty clear on what resource updates can be performed without "service interruption", keep your eyes open for those warnings.

You can also set a `DeletionPolicy` for the resources in your template. This lets you signal to CloudFormation that, for certain resources, you would like to keep the resource (even if a replacement is required for provisioning) or (if the resource supports it) perform a "snapshot" backup before deleting the resource (i.e., EC2 volumes or RDS instances). That said, even with the deletion policy set for your critical resources, you should think hard about updating them after provisioning the stack.

My recommendation is that you leave the deletion policies out of your template while you are working on it. Wait until you feel pretty good about the template, maybe until the point where you're ready to deploy your application or project. When you reach that point, go back and set the deletion policy to `Retain` on those things that are truly critical (volumes with data on them, instances you've customized heavily, etc.) and then update your stack.

Now that we have the basic ideas and the machinery down, we can move a little more quickly.

## Provision an Internet Gateway

For this project We're going to divide our virtual private cloud into two subnets: one will be able to communicate with the public internet and will house our web server. The other subnet will be able to talk to the first but will not face the internet, it will house our database server. In this way we can ensure that machines on the public internet cannot communicate directly with our database server.

In order to get our external subnet facing the internet, we need to provision an [Internet Gateway][16] and then setup routes so that instances in our virtual cloud can send and receive traffic over the public internet.

```yaml
  TutorialInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: "Name"
          Value: "Internet Gateway"
```

There isn't a lot to configure for the [InternetGateway resource][17]. We give it a name and add a tag. Next we need to attach it to our virtual cloud.

```yaml
  TutorialVPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref TutorialVPC
      InternetGatewayId: !Ref TutorialInternetGateway
```

There's not a lot to configuring the [VPCGatewayAttachment][18] either. The interesting bit is where we tell it which VPC to which we'd like it attached. We use the [`!Ref` function][30] to indicate that we are providing a _reference_ to another resource in the template and then we provide the name of that resource.

## Setup Our Subnets

With that internet stuff out of the way, we can create our two subnets. First, our public subnet.

```yaml
  TutorialPublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref TutorialVPC
      CidrBlock: "10.6.0.0/25"
      MapPublicIpOnLaunch: true
      Tags:
        - Key: "Name"
          Value: "Public Subnet"
```

We use the [Subnet resource][19] to split off [a chunk of our VPC][20] with half of our address space, we use the `VpcId` property to link this subnet declaration to our VPC. We set the `MapPublicIpOnLaunch` property to `true` so that each instance gets a public IP when launched.

```yaml
  TutorialPrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref TutorialVPC
      CidrBlock: "10.6.0.128/25"
      MapPublicIpOnLaunch: false
      Tags:
        - Key: "Name"
          Value: "Private Subnet"
```

In the stanza above, we [take the rest of our address space][21] and allocate that to our private subnet, again we use the `VpcId` property to link this subnet declaration to our VPC. We don't want instances in this network to have public IP addresses and we indicate that by setting the `MapPublicIpOnLaunch` property to `false`.

## Setup Our Routing

We always have a default route that lets the addresses in our VPC communicate with each other but we don't start out with any routes out to our internet gateway and the public internet. In order to setup our routes we first need to setup our routing tables.

```yaml
  TutorialPublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref TutorialVPC
      Tags:
        - Key: "Name"
          Value: "Public Internet Route Table"

  TutorialPrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref TutorialVPC
      Tags:
        - Key: "Name"
          Value: "Private Route Table"
```

There's not much to the [RouteTable resource][22], we indicate that we're going to be managing routes for our VPC with the `VpcId` property and set a tag.

Next we're going setup the actual routes.

```yaml
  TutorialInternetRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref TutorialInternetGateway
      RouteTableId: !Ref TutorialPublicRouteTable
```

The [Route resource][23] adds a new route to a routing table. Here we add a route to our public table to route traffic for addresses on the public internet through our internet gateway. We let CloudFormation know that this resource shouldn't be created unless the `VPCGatewayAttachment` is already setup with the [`DependsOn` attribute][24]. Like the `Retain` attribute, you can place this on any resource.

```yaml
  TutorialPublicSubnetPublicRoute:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref TutorialPublicRouteTable
      SubnetId: !Ref TutorialPublicSubnet
```

With our route to the public internet setup, we can use the [SubnetRouteTableAssociation resource][25] to link our routing table to our public subnet.

Next we need to do something similar for our private subnet. The big difference here is that our private subnet doesn't connect directly to the internet. Instead we will use a [Network Address Translation (NAT) gateway][26] to let instances on our private subnet connect to the internet without allowing machines on the public internet to initiate connections with instances in our private subnet.

```yaml
  TutorialNatGatewaySubnetRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref TutorialVPC
      Tags:
        - Key: "Name"
          Value: "NAT Gateway Subnet Route Table"
```

We create a new routing table and associate it to our VPC.

```yaml
  TutorialNatGatewayInternetRoute:
    Type: AWS::EC2::Route
    DependsOn: TutorialVPCGatewayAttachment
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref TutorialInternetGateway
      RouteTableId: !Ref TutorialNatGatewaySubnetRouteTable
```

And then we create a route to the public internet through our internet gateway.

The next step is to create our NAT gateway, but first we need to get a publicly accessible IP address that will be assigned to our NAT gateway. We can use the [EIP (Elastic IP) Resource][27] to get a handle on a new Elastic IP address.

```yaml
  TutorialNatGatewayElasticIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
```

We set the `domain` value to `vpc`, indicating that we want to use this address from our VPC. Kind of funny, that's the only valid value for that property.

Now we can setup our NAT gateway...

```yaml
  TutorialNatGateway:
    Type: AWS::EC2::NatGateway
    DependsOn: VPCGatewayAttachment
    Properties:
      AllocationId: !Sub ${TutorialNatGatewayElasticIP.AllocationId}
      SubnetId: !Ref TutorialPublicSubnet
      Tags:
        - Key: "Name"
          Value: "NAT Gateway"
```

We use the [NatGateway resource][28] to create our new NAT gateway and indicate that we need our VPCGatewayAttachment resource provisioned before this is created. We set the `SubnetId` property to put it in our public subnet so that it can communicate with the internet.

While it would make a lot of sense to provide the Elastic IP we provisioned directly to the gateway, we instead need to provide the `AllocationId` associated with that Elastic IP instead. Here we use the [`!Sub`][29] function to get the `AllocationId` of our `NatGatewayElasticIP` and provide that value to the `AllocationId` property.

Now we can setup our route for the private subnet to get to the internet via our NAT gateway.

```yaml
  TutorialPrivateSubnetNatRoute:
    Type: AWS::EC2::Route
    DependsOn: TutorialNatGateway
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref TutorialNatGateway
      RouteTableId: !Ref TutorialPrivateRouteTable
```

We create our route, using the `DependsOn` attribute to indicate that we need the `NatGateway` provisioned before this route is created. Then we use the `NatGateway` property to link in our gateway and the `RouteTableId` to tie the route to our routing table.

```yaml
  TutorialPrivateSubnetNatRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref TutorialPrivateRouteTable
      SubnetId: !Ref TutorialPrivateSubnet
```

The final step is to associate our routing table with our private subnet. Whew!

It was a long walk but we have accomplished a lot. Go ahead and provision your template and wait for CloudFormation to finish getting everything setup. From the CloudFormation web console, select your stack and click on the "Resources" tab to see all of the items that you have provisioned. This establishes your VPC and lays all of the groundwork you need, the next step is to get your instances provisioned.

## Setup Our Security Groups

Before we can provision our instances we need to get our [security groups][31] set. We will have one for each subnet and these groups will specify what traffic we will allow in and out of each of our subnets. We will allow web traffic in and out of our public subnets, in our private subnet we're only going to allow web traffic out and nothing inbound.

```yaml
  TutorialPublicSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Description: "Web Server Security Group"
    Properties:
      VpcId: !Ref TutorialVPC
      GroupDescription: Web Server Security Group
      SecurityGroupIngress:
        - CidrIp: 0.0.0.0/0         # Public Internet
          IpProtocol: tcp
          FromPort: 80              # HTTP 
          ToPort: 80
        - CidrIp: 0.0.0.0/0         # Public Internet
          IpProtocol: tcp
          FromPort: 443             # HTTPS
          ToPort: 443
        - CidrIp: 0.0.0.0/0         # Replace with your IP Range
          IpProtocol: tcp
          FromPort: 3389            # MS Remote Desktop
          ToPort: 3389
        - CidrIp: 0.0.0.0/0         # Replace with your IP Range
          IpProtocol: tcp
          FromPort: 22              # SSH
          ToPort: 22
        - CidrIp: 10.3.0.0/24       # Inside the VPC
          IpProtocol: -1            # All traffic
```

We use the [SecurityGroup resource][32] to create our new group for the public subnet. The `Description` we provide is visible in the web console and then we move on to the properties. We use the `VpcId` property to indicate that this group is for our VPC and then provide another, very similar description, for the `GroupDescription` (this property is required).

With that out of the way, we move on to the in-bound (ingress) rules. Each rule is provided by a [Security Group Rule property][33] that is made up of an IP address range, the protocol and then a range of ports. We have created rules for HTTP, HTTPS, Remote Desktop, SSH and traffic inside of our VPC.

In the example above we have opened up the remote desktop port and the SSH port to the entire internet: this is not a great idea. Since these ports are used almost exclusively for management of the instances, a better choice would be to open these ports up to the address range that is used by your office. If you don't have a fixed range of addresses you can use then I encourage you to look into other solutions, perhaps using a VPN.

```yaml
      SecurityGroupEgress:
        - CidrIp: 0.0.0.0/0         # Public Internet
          IpProtocol: tcp
          FromPort: 80              # HTTP
          ToPort: 80
        - CidrIp: 0.0.0.0/0         # Public Internet
          IpProtocol: tcp
          FromPort: 443             # HTTPS
          ToPort: 443
        - CidrIp: 0.0.0.0/0         # Public Internet
          IpProtocol: tcp
          FromPort: 587             # SSL SMTP
          ToPort: 587
        - CidrIp: 10.3.0.0/24       # Inside the VPC
          IpProtocol: -1            # All traffic
      Tags:
        - Key: "Name"
          Value: "Public Subnet"
```

Here we setup the rules for outbound traffic: HTTP, HTTPS, and SMTP over SSL. We also have a rule that allows all traffic inside our VPC. If you are more security conscious, you can nail down specific types of traffic inside your VPC (i.e., just web, SSH and database traffic).

The security group for our private subnet is much simpler.

```yaml
  TutorialPrivateSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Description: "Private Security Group"
    Properties:
      VpcId: !Ref TutorialVPC
      GroupDescription: Private Subnet Security Group
      SecurityGroupIngress:
        - CidrIp: 10.3.0.0/24       # Inside the VPC
          IpProtocol: -1            # All traffic
      SecurityGroupEgress:
        - CidrIp: 10.3.0.0/24       # Inside the VPC
          IpProtocol: -1            # All traffic
        - CidrIp: 0.0.0.0/0         # Public Internet
          IpProtocol: tcp
          FromPort: '80'            # HTTP
          ToPort: '80'
        - CidrIp: 0.0.0.0/0         # Public Internet
          IpProtocol: tcp
          FromPort: '443'           # HTTPS
          ToPort: '443'
      Tags:
        - Key: "Name"
          Value: "Private Subnet"
```

We are only allowing incoming traffic from inside our VPC into the private subnet. Outbound, we allow all traffic destined for other addresses in our VPC and we're allowing only HTTP and HTTPS to the public internet. Traffic from addresses in our private subnet is already being routed through our NAT gateway, we're likely only going to use the internet to download software updates.

## Provision Other Resources

There are a lot of other resource that we might provision, aside from instances. You might need queues, notification topics, etc. For this project we're going to setup an S3 bucket to hold dump files from our database server.

The nice thing about setting these up before the instances is that we can tell the instances about these resources and set policies so that our instances can access these resources without providing them with a specific set of credentials.

```yaml
  TutorialBackupS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: Private
      Tags:
        - Key: "Name"
          Value: "Backup Bucket"
```

Simple enough! We use the [Bucket resource][34] to create our bucket and we set that bucket to be private with the `AccessControl` property.

## Until Next Time

That is all we are going to cover in this article, we went over a lot of material. In the next piece in the series we'll setup some groups and roles and actually provision some instances.

Thank you for reading!

----
[0]: https://aws.amazon.com/cloudformation/
[1]: https://aws.amazon.com/cli/
[2]: https://aws.amazon.com/powershell/
[3]: https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-6
[4]: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html
[5]: https://yaml.org/
[6]: https://www.json.org/
[7]: https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/Welcome.html
[8]: https://aws.amazon.com/vpc/faqs/
[9]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html
[10]: http://jodies.de/ipcalc?host=10.6.0.0&mask1=24&mask2=
[11]: https://aws.amazon.com/premiumsupport/knowledge-center/create-connection-vpc/
[12]: https://console.aws.amazon.com/cloudformation
[13]: https://console.aws.amazon.com/vpc
[14]: https://github.com/cmiles74/cloudformation-tutorial/blob/master/template.yaml
[15]: https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html
[16]: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html
[17]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-internetgateway.html
[18]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc-gateway-attachment.html
[19]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html
[20]: http://jodies.de/ipcalc?host=10.6.0.0&mask1=25&mask2=
[21]: http://jodies.de/ipcalc?host=10.6.0.128&mask1=25&mask2=
[22]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route-table.html
[23]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html
[24]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-dependson.html
[25]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html
[26]: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html
[27]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-eip.html
[28]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-natgateway.html
[29]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html
[30]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html
[31]: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html
[32]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html
[33]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group-rule.html
[34]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html
[35]: https://github.com/cmiles74/cloudformation-tutorial
