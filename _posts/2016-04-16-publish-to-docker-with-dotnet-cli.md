---
layout: post
title:  "Easy publishing to Docker with the .NET CLI"
author: Vincent Lesierse
date:   2016-05-16
tags: [netcore, docker]
comments: true
---
From day one when Microsoft announced ASP.NET Core (at that time ASP.NET 5 or vNext) I was excited running my application cross platform. Especially with the power of containerized applications provided by Docker, this will be a game changer.
With the RTM release around corner I thought of creating a .NET CLI tool which allows you to publish your application easily to a Docker container. Not that it is difficult to do without tooling, but to make it as easy as possible for developers to publish their .NET Core applications to Docker.

## Prerequisites

- [.NET Core](http://dot.net)
- [Docker](https://docker.io) 

## Use the .NET Core Docker Publisher
Setting up your project to use the Docker publisher is very easy. These steps should get you up and running in no time.

The tool will create a `Dockerfile` next to your published application and execute `docker build` to create an Docker image.

### Change project.json
Next, you should change your `project.json` file and add the `dotnet-publish-docker` package to `tools` and add the `dotnet publish-docker` to the post-publish script.

```json
"tools": {
  "dotnet-publish-docker": "1.0.0-alpha1"
},
"scripts": {
  "postpublish": "dotnet publish-docker --publish-folder %publish:OutputPath%"
}
```

> It's possible to create your own base Docker image and use this as `--base-image` parameter.

### Publish your project
Now you're able to publish your application and you should see it creates a Docker image for your.  

```bash
dotnet publish
```

### Run your Docker image
With the Docker CLI you should see your image and will be able to run it.

```bash
docker images
docker run --rm <appname>
```

## .NET CLI Tools
The .NET CLI is a great tool which allows developers to build, run, package, restore, test and publish their applications. However the extensibility model makes it even more awesome.
When you create an application which starts with `dotnet-` that the .NET CLI will use this application as command. For example, `dotnet-mycommand` is translated as `dotnet mycommand`.

> Did your know that all the dotnet commands are standalone applications? There is a `dotnet-build`, `dotnet-run`, etc next to the `dotnet` executable.

### Create a .NET CLI Tool
When you would like to create your own .NET CLI tool I have some tips and considerations for you.

#### Publish to local NuGet feed
Change or create the `NuGet.config` in your solution and add an entry to a local folder. This allows you to create a sample application and test out the development experience using your tool.

~~~ xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    ...
    <add key="Local" value="/Users/vlesierse/.nuget/feed" />
    ...
  </packageSources>
</configuration>
~~~

If you want to publish your tool to your local feed you can simply use this command.

~~~ bash
dotnet pack --version-suffix dev --output /Users/vlesierse/.nuget/feed
~~~

#### Use the Microsoft.Extensions.CommandLineUtils
Using this package makes it very easy to create command line applications (like .NET CLI). It help you parse the options, arguments and commands given to application. It even generates a nice help output to improve the experience.

~~~ csharp
var app = new CommandLineApplication
{
    Name = "dotnet publish-docker",
    FullName = ".NET Core Docker Publisher",
    Description = "Docker Publisher for the .NET Core applications",
};
app.HelpOption("-h|--help");

var baseImageOption = app.Option("--base-image|-b", "Docker base image", CommandOptionType.SingleValue);
var publishFolderOption = app.Option("--publish-folder|-p", "The path to the publish output folder", CommandOptionType.SingleValue);
var projectPath = app.Argument("<PROJECT>", "The path to the project (project folder or project.json) being published. If empty the current directory is used.");
            
app.OnExecute(() =>
{
    ...
});

return app.Execute(args);
~~~

Please take a look a my [GitHub project](https://github.com/vlesierse/dotnet-publish-docker) any feedback is welcome.

- _Updated: May 16 2016 - As .NET Core RC2 has been released some of the statements weren't valid any more._
