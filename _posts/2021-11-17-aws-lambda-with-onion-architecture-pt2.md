---
layout: post
title: C# AWS Lambdas with Onion Architecture (By an AWS Beginner) pt2
date: 2021-11-17 19:00 +0000
draft: false
publish: true
tags: C# AWS Lambda Domain-Driven-Design
---
In my previous post I described how I put together some C# AWS Lambda functions with a shared Onion architecture behind them. This was my first attempt at using AWS Lambda and I feel like my solution worked well for me but with the potential downside that each new AWS Lambda function required a new  C# project. 

In this follow on post I am going to address this issue by using a single project which contains multiple Lambda functions. 

You can find the code used in this blog post on my [GitHub repository](https://github.com/the-dext/blog_dotNet_aws_lambda_with_onion_architecture_pt2).

---

# Updating Our Solution

## Migrating the Function Handlers

Let’s start by adding a new C# project to contain the AWS Lambda functions that were previously in their own dedicated projects. 
For the sake of trying this out I named mine *MultipleLambdas.csproj*

All I then had to do was copy the function handlers from the original projects into the new one, and rename them so that they do not conflict (remembering to fix-up the namespace too).

![](/images/lambda_with_onion_architecture_pt2/solution_structure.png)


## Sharing Common Code

A benefit of this new project structure is that it’s easier to share code that is common between function handlers. For example we can define a base class for our functions to hold our IoC container setup.
You can see an example of this in *FunctionBase.cs* which is now the only place that the configures the IoC container. 

```c#
    public class FunctionBase
    {
        private readonly IServiceCollection _serviceCollection;
        private readonly ServiceProvider _serviceProvider;
        private readonly Lazy<IMediator> _mediatr;
        private readonly Lazy<ILogger> _logger;

        public FunctionBase()
        {
            this._serviceCollection = new ServiceCollection()
                .AddApplicationServices()
                .AddLoggingService();
            this._serviceProvider = this._serviceCollection.BuildServiceProvider();

            this._mediatr = new Lazy<IMediator>(() => this._serviceProvider.GetRequiredService<IMediator>());
            this._logger = new Lazy<ILogger>(() => this._serviceProvider.GetRequiredService<ILogger>());
        }

        protected ILogger Logger => this._logger.Value;
        protected IMediator Mediator => this._mediatr.Value;
    }
```

The other Lambda function handlers just need to inherit from this base class which is as simple as changing the class definitions, for example `public class GetProductsFunction : FunctionBase`

## Deployments
This structure is easier to work with now that all our Lambda function handlers are co-located. But we still want our Lambdas to deploy in the same way as the previous version did, so how can we do that?

It turns out that this is quite simple. 
The Cloud Formation template files are just JSON files that we can edit.

We can copy one of the cloud formation templates (*serverless.template*) from our old solution, and then add as many function handlers as we need to the *resources* object, one entry for each function handler we want to deploy. 

In this screen shot you can see that I've combined multiple lambdas function handlers into the single serverless.template file (I've highlighted the sections of the file that are for each of my Lambda function handlers).

![](/images/lambda_with_onion_architecture_pt2/serverless_template_contents.png)


## That’s All We Need To Do!
Now when we deploy our solution using the *serverless.template* file each of the function handlers listed as resources will be deployed as AWS Lambdas. 

# Thanks For Reading
That's this end of this blog post (I did promise it would be shorter than the last one). 
Thanks for reading especially if you made it through the first part of this blog, hopefully you have found this article useful and there is something that you can apply in your solutions.

You can find the code used in this blog post on my [GitHub repository](https://github.com/the-dext/blog_dotNet_aws_lambda_with_onion_architecture_pt2).