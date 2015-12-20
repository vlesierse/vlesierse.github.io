---
layout: post
title:  "Cache busting using ASP.NET 5"
author: Vincent Lesierse
date:   2015-12-20
tags: [aspnet5]
comments: true
---
You are building this awesome new feature in your application's javascript code and designed a kick-ass responsive UI by using media queries in your css.
The marketing team has create a cool banner on the front page announcing this new feature. You've create automated tests and everything and happily you push your changes to production.
Suddenly you get calls from people asking what has happened to this new feature and why the application misbehaves. You're trying to figure out the problem but it works on your machine?!
On you colleague's computer you see the problem. The old javascript and css files are still used and you have to tell him to clear his browser cache. How are you going to tell this to all your customers?

This blogpost shows how you can load your javascript and css resources in your application using ASP.NET 5.

## Browsers
The web browser has some smart tricks to optimize your user's experience using your application. Connecting to a remote server and download files takes time.
Loading a common web application first calls for the HTML which will probably be generated dynamically using MVC for example. Each page easily could contain 20+ javascript and css files to render your client side application.
The web server sends some cache control headers with the response instructs the browser to store those resources in the browser's cache.
The next time the browser have to reach out to the server for those resources, he will probably decide to load the file from cache.

[Google's Web Fundamentals: HTTP Caching](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)

## ASP.NET 5
When you create an ASP.NET 5 MVC application from either to yeoman generator or Visual Studio project template it already includes the Microsoft.AspNet.StaticFiles NuGet package.
This package has middleware for your application to serve files from disk. If you don't have the package in your project.json you can added with ease.

```json
"dependencies": {
    "Microsoft.AspNet.StaticFiles": "1.0.0-rc1-final"
  },
```

When you inject the `StaticFileMiddleware` in your request pipeline it will look for the files in the `wwwroot` folder and serves them to the clients.

```csharp
app.UseStaticFiles();
```

While serving the files it will add a `ETag` header to the response. This way ASP.NET 5 will optimize your application.

```
Accept-Ranges:bytes
Content-Length:103
Content-Type:application/javascript
Date:Sun, 20 Dec 2015 15:54:23 GMT
ETag:"1d13b3e955769e7"
Last-Modified:Sun, 20 Dec 2015 15:53:35 GMT
Server:Kestrel
```

### E-Tag HTTP Header
The problem with using ETags is that the browser will always ask the server if the resource is changed. If it is still the same it will response with a 304 status code.
When the resource changes, the server with send an updated version with status code 200 and a new ETag header which will be store in the browser's cache.
Using ETags saves you the bytes going over the network to your users but it will not cut the amount of calls. ASP.NET 5 still have to handle all those request and checks if the files are changed.

### Add Cache-Control HTTP Header
A good practice is to add a Cache-Control header with your response wich will contain a max-age value. This will instruct the browser to use the file from cache for a certain period instead of asking the server if the file has been changed.

This is a way how to do this easily with ASP.NET 5. You can change the code in your `Startup.cs` to:

```csharp
app.UseStaticFiles(new StaticFileOptions() {
    OnPrepareResponse = (context) => {
        var headers = context.Context.Response.GetTypedHeaders();
        headers.CacheControl = new CacheControlHeaderValue() {
            MaxAge = TimeSpan.FromDays(1)
        };
    }
});
```

This will solve flooding you server with HTTP requests but the problem of invalidating the cache at your users still exists. So we need to change the strategy.

```
Accept-Ranges:bytes
Cache-Control:max-age=86400
Content-Length:103
Content-Type:application/javascript
Date:Sun, 20 Dec 2015 15:54:23 GMT
ETag:"1d13b3e955769e7"
Last-Modified:Sun, 20 Dec 2015 15:53:35 GMT
Server:Kestrel
```

### Versioning
If you are changing your resource files you could choose to version them. When the browser see the resource it is actually a different one for him because the url has been changed. Resulting that it will not try to load the resource from cache.
It is possible to adopt all kinds of versioning strategies like symantic versioning.

```html
<script src="~/js/app-1.0.js"/></script>
```

Versioning of your files will make maintenance of your files a hell. In short, DON'T! 
A better approach is to just change your url but keep the physical location of the resource the same.

```html
<script src="~/js/app.js?v=1"/></script>
```
This also will change the url and solves the caching problem, but changing the url for each change of the resource is also not really maintanable.

#### asp-append-version
There is no need for you to change the version manually. You can use the `asp-append-version` attribute on the script or link tag.

```html
<script src="~/js/app.js" asp-append-version="true" /></script>
```

MVC will generate a hash of your file at runtime and renders this to your script or link tag.
This means when your file changes it will calculate another hash which is basically another version.

```html
<script src="/js/app.js?v=J4F2S0DV72Q7OV0fi5JNlEa-uRAClFhpCCiOYMOHqMg"></script>
```

Unfortunately there is no file watcher on this file, like ASP.NET Bundling & Minification does. ASP.NET 5 doesn't recalculate the hash when the file changes at runtime. You have to restart your application to see effect.

### Webpack, Gulp, Grunt, etc
What about when you are adopting modern client side development techniques like building you client code using Webpack, Gulp or Grunt?
They can great in minifing and calculate hashes after compilation/transpilation. It will save your server some time in calculate hashes because the files are already there.
How can ASP.NET 5 discover the right files with the correct hash code?

#### asp-src-include
You are able to discover the files on the web server and inject the url using the `asp-src-include` attribute on the script and link tags.

 ```html
<script asp-src-include="~/js/app_*.js" /></script>
```

## Summary
Working with resource files like javascript and css doesn't need to hard anymore. It's easy to optimize your user experience and be able to change you application often.
ASP.NET 5 hands you some nice tag helpers which can help you with your projects.
 