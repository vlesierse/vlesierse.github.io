---
layout: post
title:  "Infrastructure as Code at Exact"
author: Vincent Lesierse
date:   2017-05-19
tags: [exact,devops]
comments: true
cover: devops-process.png
---
Recently at Exact we have done a project to proof wether Exact Online is able to run on a public cloud environment (Microsoft Azure or Amazon AWS). With this project we have set the principal to automate everything. Doing this ten years ago it looked hard to do maybe even impossible. Meanwhile the world has change and everything has an API. Now is the time do bring the practices from software development into the world of infrastructure and operations. 

## Code
Ok, now the question is what do I need to do this myself? I'll show you what we used to accomplish this and why. Hopefully it will help you in the right direction.

### Infrastructure Templates
With our project we looked at the two major cloud providers (Microsoft Azure and Amazon AWS). Both they offer a rich set of APIs to provision infrastructure on their platforms.

#### Azure Resource Manager Templates
Microsoft Azure allows you to describe your infrastructure using JSON files. These files allows you to specify resources, variables, in and output parameters. With some basic functionality you can do all kind of operations to make them more dynamic.

```json
...
{
  "apiVersion": "2016-04-30-preview",
  "type": "Microsoft.Compute/virtualMachines",
  "name": "[variables('vmName')]",
  "location": "[resourceGroup().location]",
  "dependsOn": [
    "[concat('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))]",
    "[concat('Microsoft.Network/networkInterfaces/', variables('nicName'))]"
  ],
  "properties": {
    "hardwareProfile": {
      "vmSize": "[parameters('vmSize')]"
    },
    "osProfile": {
      "computerName": "[variables('vmName')]",
      "adminUsername": "[parameters('adminUsername')]",
      "adminPassword": "[parameters('adminPassword')]",
      "customData": "[base64(parameters('customData'))]"
    },
    "storageProfile": {
      "imageReference": {
        "publisher": "[variables('imagePublisher')]",
        "offer": "[variables('imageOffer')]",
        "sku": "[parameters('ubuntuOSVersion')]",
        "version": "latest"
      },
      "osDisk": {
        "createOption": "FromImage"
      }
    },
    "networkProfile": {
      "networkInterfaces": [
        {
          "id": "[resourceId('Microsoft.Network/networkInterfaces',variables('nicName'))]"
        }
      ]
    },
    "diagnosticsProfile": {
      "bootDiagnostics": {
        "enabled": "true",
        "storageUri": "[concat(reference(concat('Microsoft.Storage/storageAccounts/', variables('storageAccountName')), variables('apiVersion')).primaryEndpoints.blob)]"
      }
    }
  }
}
...
```

[Link](https://docs.microsoft.com/en-us/azure/azure-resource-manager/)

#### AWS CloudFormation Templates
Amazon AWS has a similar approach to describe your infrastructure called CloudFormation. With CloudFormation you also define resources, in and output parameters but you have more options in the choosing the format. Like ARM you can use JSON but to make it even more readable you can use YAML (Yet another markup language) as well.

```yaml
Ec2Instance: 
  Type: "AWS::EC2::Instance"
  Properties: 
    ImageId: 
      Fn::FindInMap: 
        - "RegionMap"
        - Ref: "AWS::Region"
        - "AMI"
    KeyName: 
      Ref: "KeyName"
    NetworkInterfaces: 
      - AssociatePublicIpAddress: "true"
        DeviceIndex: "0"
        GroupSet: 
          - Ref: "myVPCEC2SecurityGroup"
        SubnetId: 
          Ref: "PublicSubnet"
```

AWS also provide some tooling on top of CloudFormation which allow you to view your template in a graphical diagram and it even comes with a designer.

[Link](https://aws.amazon.com/cloudformation/)

#### Terraform
Terraform is a great tool provided by Hashicorp which allow you to describe your infrastructure in the same way agnostic of which cloud provider your are using. It support providers like AWS and Azure (Limited by enough for IaaS), but also OpenStack, Google Cloud, Digital Ocean, etc. It contains the same characteristics as the cloud native solutions like being idempotent and allows you to store the state remotely to make it possible to work together on one environment.

The reason why using Terraform made the experience great was that is far less verbose as ARM or CloudFormation. As the complexity of your infrastructure increases your will see a steep increase in your template's file size. There is support for separating files but it makes it complex to work and deploy it.
Terraform's syntax it far more compact as it using calculations for referencing resources and you can make as many files as you want, but during deployment it will be grabbed all together.

```
resource "aws_instance" "web" {
  ami           = "${data.aws_ami.ubuntu.id}"
  instance_type = "t2.micro"
  network_interface {
     network_interface_id = "${aws_network_interface.foo.id}"
     device_index = 0
  }
}
```

[Download](https://www.terraform.io/)

### Configuration Automation
Having your infrastructure provisioned as code is great but it doesn't mean you are already there. In oder to run your application you also have to configure the base image is such a way that it include the required middleware for your application to work. In Exact Online's case it will be IIS with ASP.NET on a Web Server for instance. Also this part you could and should code.

#### Chef
Chef is an Open Source tool which allows you to describe your configuration in a desired state, which make it idempotent as well. It uses a Ruby dialect as syntax and is very clean and easy to learn. Just look at the sample below.

```ruby
iis_site 'Foo Site' do
  protocol :http
  port 80
  path "#{node['iis']['docroot']}/foo"
  action [:add,:start]
end
```

Chef contains out a client and a server component. The client is installed on the machines and contacts the server on a specified interval. It will as the server, "Hey, I'm this node, my role is ... and runs in environment .... How do I have to configure myself?" As soon as the server responds it will send down the code to the client which use this to executed the configuration. When finished, or not, it will report back about it's current state.

Configuration of machines is one thing, but the compliance around this is another. With that Inspec could help. With Inspec it's possible to create rules as code describing what the machine should have configured. It sends a report to the Chef Server with the results.

[Download](https://www.chef.io/)

### Immutable infrastructure
Chef and the configuration using the Chef server works great for more static environment. As you might know, today's  cloud platforms are more dynamic in nature. What does this mean? Mostly likely your application like Exact Online doesn't have the same load profile through the entire day. With Exact Online we see more load during business hours then in the evenings or the weekends. In the public cloud you could scale machines easily on demand which results in cost savings.

Imagine the during the morning new machines have to be added to your environment. If you use just Chef for the configuration of the machine and deploy the application, it will take 20 to 30 mins until the machine is available. To speed this up you should use images which are pre-baked with the desired state of configuration and even with the application installed so that it will take 1 to 3 mins to create the machine and have the application running.

#### Packer

For our project we used Packer to do exactly that. It's also an open source tool provided Hashicorp which allow you to back images for a specific cloud. It supports the major clouds like Azure and AWS but also provisioners like Chef. This allowed us to use the same technology for configuring the more stated machines within our infrastructate and baking the images for the dynamic ones.

[Download](https://www.packer.io/)

## Deployment Pipeline
Currently we do a good job deploying the application to our production environment. Everything is automated from the beginning to the end. As soon a developer check-in his code to the source repository it will trigger some basic checks wether the application compiles. As soon as it does and the Unit Tests doesn't show any problems the code will be merge to the release branch. Every 2 hours the latest version of the release branch is taken out of the repository and a suite of automated integration and UI tests will verify if the application is ready for deployment. If it does the release will be flagged as release candidate. During the night the release candidate will be pulled in to the production environment and a set of scripts will make sure the application is updated. During this process there is no human interaction or what ever, which makes it process robust and fast. It allows us to deploy a new version of Exact Online every day which comply do our quality standards.

The question is, why don't we do the same for infrastructure setup and configuration? Why do we still allow our operations engineers to RDP/SSH into a machine and change the configuration. Sometimes we already to have documents explaining step by step what to do, why don't we automate this as we already do with our applications?

A deployment pipeline for your infrastructure looks very similar and I believe it's even better to deploy you infrastructure together with your application. Let's face it, only together they bring value to your customers.

## Summary
Practicing Infrastructure as Code is your only way if you would like to deliver quality within a fast moving world. There are currently a lot of tools around which are even open sources to help you out to do the best possible job. Hopefully this post inspired you and help you to take the first steps in this new world.
