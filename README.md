# Provision Pragmatically and Programmatically with CloudFormation

How many times have you gazed upon the list of services that Amazon is providing
through their Amazon Web Services (AWS) family of products and felt an
unpleasant mixture of anxiety and confusion? If you're anything like me, it
would be literally _every time_ you visited their website. While I was one of
the first in my circle of software developer friends to get on board the Elastic
Compute Cloud (EC2) train, I haven't been great at keeping up with new
developments. This year, I decided, would be the year that I get back on track
and sort out this alphabet soup of cloud virtualization technology.

After clicking around at random and reading an unwholesome amount of bland
introductory pages, I finally started to get an idea of how I should try to put
all of these pieces together. The first thing I figured out: 

> It is way, way too hard to get everything up and running by clicking a bunch
> of buttons.

What are your goals? For me, it's to be able to deploy a web-based application
for one of my clients. As a corollary, I want to be able to entirely scrap a
web-based application for one of my clients (maybe it's been replaced, maybe
they've hired another consultant). In effect, that's two goals. Plus I would
like to keep all of my clients separate, I don't want changes to a firewall rule
to effect all of my clients, just the one client who's willing to take the risk.
If you think about it, keeping things isolated will also make it easier to
figure out who we can charge for what services at the end of the month; we
should also tag these resources in a way that makes sense. That's four goals and
that's only what we've thought of so far.

* Provision resources for a project
* Retire resources for a project
* Isolate resources for a project
* Report on resources by project

Looking at these goals, the number of services, the way many of them build on
each other... It all becomes clear that the answer is automation. Of course,
Amazon provides an automation tool: [CloudFormation][0].

## Install the AWS Command Line Tools

Amazon provides a goofy drag-and-drop interface for CloudFormation, but that
runs kind of counter to the whole point of this article. You can also write your
CloudFormation template and then upload it via their web interface, I think
that's also a little clunky. My recommendation is that you install the [AWS
Command Line Tools][1]. Amazon also provides packages for Windows, but if you're
working from within the Windows environment you might be more interested in the
[AWS Tools for PowerShell][2] which do much the same thing but, you know, with
[]PowerShell][3].

This is kind of a power tip, but if you're managing more than one Amazon account
or dealing with more than one region, you can [configure profiles][4] to make
that process easier.

With one of these tools installed and properly configured, you can invoke
CloudFormation directly from your console window! `:-)`

## Start Your New CloudFormation Template

In the world of CloudFormation, the document you are writing is called a
"template". I believe it's referred to as a template because one template can be
used to create many different sets of resources. For instance, in this tutorial
we're going to be writing a template that will provision a web server and a
database server; we can use the template to provision as many of these
web-and-database-server pairs as we might lock, all independent of one another.

You can write your template in [YAML][5] or [JSON][6], if you're writing it by
hand (as we are right now) then I _strongly_ recommend that you write it in
YAML. I know, I know, YAML isn't everyone's favorite but getting all of the
ending braces and brackets lined up in a JSON document is it's own kind of
mental torture. You deserve better, trust me: don't do that to yourself.

Anyway, we're going to start off with a description and a parameter. A
parameter, as you may have suspected, is a value that we can supply when we are
actually using our template to provision something.

```yaml
Description: Provisions a simple cluster with a web and a database server

Parameters:
  KeyPairName:
    Type: String
    Description: Name of keypair used when provisioning instances
```

Some decisions are best postponed until the absolute last minute and parameters
are here when you have decisions to postpone. When we decide to provision our
resources, we can provide the name of the EC2 key pair to use at that time.

I'm not going to go into a big discussion about how to manage your EC2 key
pairs. Maybe you have one per person who has access to provision instances.
Maybe you have one per client. However you manage it, you can pass it in when
you provision resources.

## Provision Resource With Your Template

With our template set, we're ready to use it to provision some resources! Go
ahead and run the command below to being provisioning.

```shell
aws cloudformation create-stack \
  --stack-name tutorial \
  --template-body file://template.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=cloudfront-tutorial
```

You should promptly receive the error below:

> An error occurred (ValidationError) when calling the CreateStack operation:
> Template format error: At least one Resources member must be defined.

Not a surprise, we didn't actually ask CloudFormation to provision anything for us.

------
[0]: https://aws.amazon.com/cloudformation/
[1]: https://aws.amazon.com/cli/
[2]: https://aws.amazon.com/powershell/
[3]: https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-6
[4]: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html
[5]: https://yaml.org/
[6]: https://www.json.org/
