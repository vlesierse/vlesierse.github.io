---
layout: post
title:  "Running ASP.NET 5 on Kestrel with Service Fabric"
author: Vincent Lesierse
date:   2015-12-27
tags: [aspnet5, servicefabric]
comments: true
---

By default the ASP.NET 5 project template, which ships with the [Service Fabric SDK](https://azure.microsoft.com/documentation/articles/service-fabric-get-started/#install-the-runtime-sdk-and-tools), configures your ASP.NET 5 application to run with Web Listener.
The ASP.NET team [changed](https://github.com/aspnet/Announcements/issues/69) the hosting model and dropped Helios in IIS. Instead they are forwarding the traffic using a HttpPlatformHandler towards Kestrel.
Because I'm developing my application mostly on my Mac, using Visual Studio Code, I don't see a reason why I would like to support two hosting models (Kestrel and Web Listener) with my application. I also want to profit from the performance improvements Kestrel offers in production.
In this post I will show you how to change the project to use Kestrel instead of Web Listener.

## Let's begin
November 18th, Microsoft has release the first public preview of Service Fabric. This release also contained a new SDK which provided support for ASP.NET 5 RC1.
Up until this release you were bound to use Owin and the Self Host solution to expose your application over HTTP.

The bits will allow you to create a Service Fabric cluster on Azure and deploy your ASP.NET 5 applications with a click of a button.

*Take a look at the Service Fabric party cluster to tryout your application if you don't want to spend money on running your own cluster in Azure.*
 
After playing around with the Visual Studio project template I realized that it is far from complete. If you follow the instructions described by the document it will work just fine.
If you want to do a little more, you have to work arround a couple of things.

## Changing the service manifest
When you select ASP.NET 5 as option in the Service Fabric project template it will generate a `ServiceManifest.xml` file with a `<CodePackage/>` section looking like this:

```xml
<CodePackage Name="C" Version="1.0.0">
    <EntryPoint>
        <ExeHost>
        <Program>approot\runtimes\dnx-clr-win-x64.1.0.0-rc1-update1\bin\dnx.exe</Program>
        <Arguments>--appbase approot\src\WebApplication Microsoft.Dnx.ApplicationHost Microsoft.ServiceFabric.AspNet.Hosting --server Microsoft.AspNet.Server.WebListener</Arguments>
        <WorkingFolder>CodePackage</WorkingFolder>
        <ConsoleRedirection FileRetentionCount="5" FileMaxSizeInKb="2048" />
        </ExeHost>
    </EntryPoint>
</CodePackage>
```

You would think changing `--server Microsoft.AspNet.Server.WebListener` to `--server Microsoft.AspNet.Server.Kestrel` will do the trick, right?
Unfortunately this is not the case... It appears that as soon as you run or publish your application the service manifest will be overwritten and undo your changes.

## Disable updating the service manifest
After looking around for a bit what could be responsible for this behavior, I discovered that it actually very simple to change this.
When you edit your application's `xproj` file you will see this property group:

```xml
<PropertyGroup>
    <SchemaVersion>2.0</SchemaVersion>
    <UpdateServiceFabricManifestEnabled>true</UpdateServiceFabricManifestEnabled>
    <ServiceFabricServiceManifestPath>PackageRoot\ServiceManifest.xml</ServiceFabricServiceManifestPath>
</PropertyGroup>
```

*I can recommend the [Productivity Power Tools add-in for Visual Studio](<UpdateServiceFabricManifestEnabled>true</UpdateServiceFabricManifestEnabled>). It adds a nice menu option to edit your project file in Visual Studio when you right-click on the project.*
 
Changing '<UpdateServiceFabricManifestEnabled>true</UpdateServiceFabricManifestEnabled>' to '<UpdateServiceFabricManifestEnabled>false</UpdateServiceFabricManifestEnabled>' will disable the behavior of changing your service manifest. Now you can change the manifest as you like without this annoying behavior.