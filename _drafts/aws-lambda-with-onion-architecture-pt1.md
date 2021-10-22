---
layout: post
title: AWS Lambdas with Onion Architecture (.net), by an AWS Noob (pt 1)
date: 2021-10-15 12:09 +0100
draft: true
publish: false
tags: C# dotNet AWS Domain-Driven-Design
---
In this post I'm going to try and document my experience of learning AWS Lambda and applying Onion Architecture to my .net solution.

I'm new to AWS, having recently joined a team that are heavily using it and my previous employers not using AWS, so if you're a seasoned AWS practitioner you might find some room for improvement in some of the things I've done.

_Update: This post has become quite large so I've decided to split it into a series of posts_

---
# My Background
Although I have used Onion Architecture many times on my types of project I have never written any AWS Lambda before and I had never seen any examples where this architecture is used. At first I was unsure if this was just because the examples are all kept simple or or if this architecture style is just incompatible with Lambda, as I had heard many times that Lambdas have to be small. 

Spoiler Alert - I'm happy to say that so far it seems that Onion Architecture works great with AWS Lambdas, and I'm really pleased how simple it seems to be!

---
# What is Onion Architecture ?
So before we dive into examples of how to setup an Onion Architecture for your solution, let me take a bit of time to explain what it actually is.
Onion Architecture is a layered architecture style first documented (to my knowledge) by Jeffrey Palermo in 2008.
Unlike an n-tier architecture which promotes arranging your dependencies in a vertical manner, Onion Architecture (and other similar architectures such as Hexagonal and Clean) promote arranging your dependencies in an inward facing manner.
Usually when drawing out these types of architecture they are represented by concentric circles similar to the layers of an onion, hence the name.

I've been using Onion Architecture for a number of years now, and it's never failed to help me produce a solution that is well laid out and maintainable. It's an excellent architecture pattern to promote loose coupling between layers and makes separation of concerns easier.
Whether the solution has been a monolith or micro-service has made no difference.

## My Onion Architecture Implementation
I don't want this blog post to become too long so I'll probably write a separate post about how I use Onion Architecture in more detail. For the purpose of this article you just need to know a these key things.

Usually my solution is arranged like this.
![Basic Onion Architecture](./../_site/lambda_with_onion_architecture/onion_architecture_basic.png)

1) The Domain layer it contains all the business rules for the app. 
__It is only referenced by the application and infrastructure layer and it does not reference any other layer__.

2) The infrastructure layer deals with talking to external things, like databases, and logging APIs.

3) The Application layer takes commands, coordinates the repositories and domain layers to invoke business rules, and returns responses back to the caller. 
The Application, Infrastructure and Domain layers working together provide you with a working application all be it without any sort of user interface.
That's where the hosted layer comes in.

1) The Hosted layer/System Boundary contains hosted processes that take input from somewhere - a user clicking a button, an event being delivered from a message queue, or a HTTP message coming in to a REST/GraphQL endpoint. 
I've named it the hosted layer because these processes require some kind of hosting whether that be IIS/Kestrel, WPF or something else. 
You can also think of this as the system boundary, it's the interface between your solution and the user/some other process.

1) The DTO Boundary is not an actual layer. My art skills aren't very good but what I'm trying to show here is that communication between the Hosted Layer and the Application Layer is __only__ done through some form of DTO. I tend to use Commands for inputs and DTO Responses, which is made easy by using the Mediatr and AutoMapper nuget packages (both by Jimmy Bogard). 

It's worth noting that this architectural style relies on Inversion of Control, and if for some reason you couldn't use IoC then you may struggle to implement this architecture without losing loose coupling and separation of concerns.

His blog posts about Onion Architecture are an interesting read and I recommend you check them out (here)[https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/]

# What _**I Think**_ an AWS Lambda is
After reading about AWS Lambda I understood that they offered serverless on-demand processing and ran at a function level (eg a web api endpoint), as opposed to a container hosting a process which provides many functions (eg. a web api with multiple endpoints). 
So it makes sense that best practice seems to be that a lambda does one thing and quickly, and best practice is something I would like to follow.

But I was unsure if this meant that I would be limited in how I structure a domain model behind the lambda(s), most examples I had were simplistic and if followed in an enterprise application they would would result in the logic for a single domain actually being distributed across many lambdas projects, making it almost impossible to have a consolidated domain in my microservice.

I should point at this point that I don't subscribe to the belief that a micro-service *has* to be made up of a single component (e.g. a lambda). 
In my opinion a micro-service represents a bounded context and is made up of however many runtime components are needed to support that context (it could be a single one, or it could be a few). Having a hard a rule that a micro-service has to be one single runtime component can lead to micro-services that are not truly independant of each other.

# My Target Architecture
I decided my target architecture for this prototype would be largely the same as what I had done previously, but with AWS Lambdas in the Hosted / System Boundary layer. If this is possible then adding AWS Lambdas into my future projects would be pretty simple and I would not need to change who I design my software.
A bonus would also be that introducing AWS Lambdas into my existing projects would be also very easy.
So my new target architecture diagram will look pretty similar to the previous version.

![](./../_site/lambda_with_onion_architecture/target_architecture.png)


Time to start coding...

---
# The First Attempt
I started off by creating a new .net core solution with a made up domain, app layer and repository layer. It's nothing complicated, the repository retrieves a hard coded set of products, or a single one by SKU (stock keeping unit).

![](../_site/lambda_with_onion_architecture/project_layout_before_lambdas.png)

There are quite a few files in this example, but it's all fairly simple.
The application layer has a couple of classes in the queries namespace `GetProductBySkuQuery` and a `GetProductsQuery`. These are MediatR commands that can be issued by any class that needs to query data from the application layer.
Each of these query classes has a corresponding handler, this are the classes that MediatR will instantiate and call when one of the queries are sent to the application layer.
In our example the query handlers are `GetProductBySkyQueryHandler` and `GetProductsQueryHandler`.

Finally in the application layer we have a Dtos namespace, with a single dto representing what a product will look like when it is passed out of our application layer.

The implementations of the MediatR commands and handlers are very simple.
Here is what the GetProductBySku query and handler look like, along with the dto that is returned.

---

GetProductBySkuQuery.cs
```c#
public class GetProductBySkuQuery : IRequest<ProductDto>
{
    public GetProductBySkuQuery()
    {

    }

    public GetProductBySkuQuery(Guid tenantId, string sku)
    {
        this.TenantId = Guard.Against.Default(tenantId, nameof(tenantId));
        this.Sku = Guard.Against.NullOrWhiteSpace(sku, nameof(sku));
    }

    public string Sku { get; set; }

    public Guid TenantId { get; set; }
}
```

GetProductBySkuQueryHandler.cs
```c#
public class GetProductBySkuQueryHandler : IRequestHandler<GetProductBySkuQuery, ProductDto>
{
    private readonly IProductsRepository _productsRepository;
    private readonly IMapper _mapper;

    public GetProductBySkuQueryHandler(IProductsRepository productsRepository, IMapper mapper)
    {
        this._productsRepository = Guard.Against.Null(productsRepository, nameof(productsRepository));
        this._mapper = Guard.Against.Null(mapper, nameof(mapper));
    }
    public async Task<ProductDto> Handle(GetProductBySkuQuery request, CancellationToken cancellationToken)
    {
        Guard.Against.Null(request, nameof(request));

        var product = await this._productsRepository.GetBySkuAsync(request.TenantId, request.Sku, cancellationToken);
        return this._mapper.Map<ProductDto>(product);
    }
}
```
ProductDto.cs
```c#
public class ProductDto
{
    public Guid Id { get; set; }
    public Guid TenantId { get; set; }

    public decimal Price { get; set; }

    public string Description { get; set; }

    public string Name { get; set; }
}
```
---
## Adding some AWS Lambda Projects
At this point we essentially have a solution that works, which could be proven by using unit tests. But users and other processes have no way of interacting with our application, so the next step is to create our Hosted / Service Boundary layer. 

We are doing this with AWS Lambda, and in keeping with the best practices we need a separate lambda for each of the commands that our application layer will accept.
This is quite simple by installing the extension 'AWS-Toolkit for Visual Studio' and once installed using the new 'C# AWS Lambda Project' project type.

![](../_site/lambda_with_onion_architecture/create_lambda_project_dialog.png)

I used this dialog to create the two lambda projects shown below
![](../_site/lambda_with_onion_architecture/project_with_lambdas_added.png)

(If you are following along with this article you may notice that you don't have any serverless.template files. These are files I added later to deploy the projects using AWS Cloud Formation. Ignore these for now, I'll explain them later.)

Each of our lambda projects has a Function.cs class, which the toolkit created for us. This is the entry point of the lambda, lets take a closer look at the Funciton.cs file in the `GetProducts` lambda.

```c#
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace GetProducts
{
    public class Function
    {
        private readonly IServiceCollection _serviceCollection;
        private readonly ServiceProvider _serviceProvider;
        private readonly Guid _tenantId;
        private readonly Lazy<IMediator> _mediatr;

        public Function()
        {
            this._serviceCollection = new ServiceCollection()
                .AddApplicationServices()
                .AddLoggingService();
            this._serviceProvider = this._serviceCollection.BuildServiceProvider();

            this._mediatr = new Lazy<IMediator>(() => this._serviceProvider.GetRequiredService<IMediator>());

            // JUST FOR TESTING, forces the tenant ID to be a known one so the user doesn't have to remember it
            this._tenantId = Guid.Parse("743872ea-7e68-421b-9f98-e09f35d76117");
        }

        public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)
        {
            var logger = this._serviceProvider.GetRequiredService<ILogger>();
            logger.SetLoggerContext(context.Logger);
            logger.LogInfo($"Fetching all products for tenant: {this._tenantId}");

            try
            {
                var query = new GetProductsQuery(this._tenantId);
                var queryResponse = await this._mediatr.Value.Send(query);

                logger.LogInfo($"Returning {queryResponse.Count()} records");

                // return result
                return new APIGatewayProxyResponse
                {
                    StatusCode = (int)HttpStatusCode.OK,
                    Body = JsonConvert.SerializeObject(queryResponse)
                };
            }
            catch (Exception ex)
            {
                logger.LogError($"exception; {ex.Message}");
                return new APIGatewayProxyResponse
                {
                    StatusCode = (int)HttpStatusCode.InternalServerError,
                };
            }
        }
    }
}
```
Both the lambdas are pretty similar so let's concentrate on this one and break what is going on.

### Setting the JSon Serializer
First of all we have the `[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]` assembly attribute.
The AWS-Toolkit added this for us and it sets up a JSon Serializer that is compatible with AWS Lambda. I'm sure it's possible to use a different serializer implementation if you needed to, such as Newtonsoft Json, but I'm happy with the default so I'll move on.

### The Function Constructor
Next we have the class definition, and the class constructor.
In my example I have used the constructor to instantiate a new `ServiceCollection` and configure the IoC container by calling the `AddApplicationServices()` and `AddLoggingService` extension methods and building the IoC container.

Next I setup a lazy instantiation of Mediatr by using the following line
`this._mediatr = new Lazy<IMediator>(() => this._serviceProvider.GetRequiredService<IMediator>());`
This allows the creation of Mediatr to be done within the constructor, but deferred until actually needed by the code that requires it.

### The Function Handler
The next part of the file is the Function Handler method that will be invoked once our AWS Lambda has been created.
The signature for this method is 
`public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)`

Note that the request argument and the method return type are special in that we are using `APIGatewayProxyRequest` and `APIGatewayProxyResponse` respectively.
The argument and return types you will use in your AWS Lambda function handlers are specific to how your AWS Lambda is being triggered. 
In my example I am creating the equivalent of API endpoints, and I want them to be accessible via a public API gateway so that I can call them from my PC when they have been deployed to the cloud. So to do that I have to accept and return the types required by the API gateway. If my Lambda was beign triggered some other way then these types would be different.

The rest of the method is not very complex, and when you strip away the logging lines and the exception handling it will come down to just 3 lines of code.
1) Creating one of the MediatR commands
2) Dispatching the command via MediatR and waiting for the response
3) Serializing the command response into an API Gateway reponse body and returning it (along with a status code)
```c#
var query = new GetProductsQuery(this._tenantId);
var queryResponse = await this._mediatr.Value.Send(query);
// return result
return new APIGatewayProxyResponse
{
    StatusCode = (int)HttpStatusCode.OK,
    Body = JsonConvert.SerializeObject(queryResponse)
};
``` 

That's it. This Lambda is now fully functioning, and you can test this by running the debugger. The AWS-Toolkit for Visual Studio understands how to debug an AWS Lambda locally by setting the AWS Lambda project as the start-up project and clicking the debug icon, which will launch your browser and show a special debugging page that you can use to invoke your AWS Lambda and step through it.

![](../_site/lambda_with_onion_architecture/lambda_test_tool_icon.png)
![](../_site/lambda_with_onion_architecture/lambda_debugger.png)

---
# Deploying To The Cloud

# It's a Wrap...well maybe not
At this point everything deploys and works and it's pretty good. We have followed AWS Lambda best practices (as far as I know them), and used an architecture that promotes single responsibility in our code whilst allowing us to build up a a comprehensive domain model.

But there's a potential problem...

In a real enterprise solution if we are using this implementation pattern then as we add new functionality we will be adding new AWS Lambda projects. Eventually the number of AWs Lambda projects is going to grow and in this is not ideal. 
Having many AWS Lambda projects will make it harder harder for us to keep all the nuget packages used consistent between the projects, code potentially harder to navigate around and sharing code between each AWS Lambda project is clunky and requires extra work (for example the duplicated code to configure IoC in each Lambda). Overall having many AWS Lambda projects just feels unnecessary.

In the next post in this series I will address this issue by showing how to consolidate them into a single project.

--- 
# Thanks For Reading
If you've made it this far, thanks for reading and I'm sorry what I thought was going to be a reasonably sized post turned into such a monster.

In the next part I'll talk more about how I tackled the problem of having separate AWS Lambda projects for each AWS Lambda. This will make our solution more manageable as it grows to support much more functionality - which means many more AWS Lambdas.