---
layout: post
title:  "Getting started with Vamp on Azure Container Service"
author: Vincent Lesierse
date:   2016-03-25
tags: [vamp,acs,dcos]
comments: true
---
*TODO: Introduction*

## Vamp
*TODO: What is Vamp and why should you install it on ACS?*

## Prerequisites
For this samples I've choosen to make use of the command line interfaces which are availabe for the used technologies.

- [Azure trial subscription](https://azure.microsoft.com/free): If you don't have a subscription for Microsoft's Azure, you can create one for free.
- [Azure CLI](https://azure.microsoft.com/documentation/articles/xplat-cli-install): A confinient way to talk with Azure is via the command line interface. Alternatives are the PowerShell SDK, REST API of Web interface.
- [DCOS CLI](https://docs.mesosphere.com/administration/cli/install-cli): 
- [Vamp CLI](http://vamp.io/documentation/cli-reference/installation)

## Create your cluster

```bash
git clone https://github.com/vlesierse/vamp-dcos
cd vamp-dcos/azure
azure group create VampCluster westeurope
```

```bash
azure group deployment create VampCluster -f azuredeploy.json -e azuredeploy.parameters.json
```

*TODO: Provide the values or change the azuredeploy.parameters.json file*

*TODO: Result with information to connect to you cluster*
```
data:    masterFQDN  String  vampclustermgmt.westeurope.cloudapp.azure.com
data:    sshMaster0  String  ssh azureuser@vampclustermgmt.westeurope.cloudapp.azure.com -A -p 2200
data:    agentFQDN   String  vampclusteragents.westeurope.cloudapp.azure.com
```

## Connect to your cluster

```bash
ssh -L 8080:localhost:80 -N azureuser@vampclustermgmt.westeurope.cloudapp.azure.com -p 2200
```

- Mesos: http://localhost:8080/mesos
- Chronos: http://localhost:8080/chronos
- Marathon: http://localhost:8080/marathon


### Configure DCOS
```
dcos config set core.mesos_master_url http://localhost:8080/mesos/
dcos config set marathon.url http://localhost:8080/marathon/
```

## Install Vamp

```
cd ../marathon
dcos marathon app add vamp.json
dcos marathon app add vamp-gateway.json
```

```bash
ssh -L 8080:localhost:80 -L 8081:10.0.0.4:8000 -N azureuser@vampcluster.westeurope.cloudapp.azure.com -p 2200
```

## Run you application

```bash
cd ../vamp
export VAMP_HOST=http://localhost:8080
vamp generate blueprint --file sava.yaml
vamp deploy save:1.0
```

*Note: At the moment of writing the `vamp deploy` command shows and error, but the deployment does succeed.*
