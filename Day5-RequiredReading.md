# .NET Essentials 2 - Session 5
## Learning Outcomes
* [Deployed API with Swagger Tooling](#1)
    * [Deploy an API to Azure](#2)
* [Swagger and OpenAPI Specification](#3) 
* [Customize Swagger](#4)
* [XML Comments](#5)
* [HTTP Status Codes](#6)
  
---

<a name="1"></a>
# Deployed API with Swagger Tooling
Today we are going to deploy a .NET Core API to Azure and integrate and explore tools provided by Swagger.

**Resources:**

* [Swagger](https://swagger.io/)
* [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle)
* [Microsoft.AspNetCore.Http StatusCodes](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.statuscodes?view=aspnetcore-3.1)

<a name="2"></a>
## 2) Deploy an API to Azure
We will start with our API from last class with the ToDoController having (Get, Get/{id}, Post, Put, and Delete methods).

For stronger data persistence and to allow for multiple connections a production app we will use a Relational Database. I had significant challenges executing the sqlserver migrations on production while the sqlite migrations were present...so we will swap fully to sqlserver.

```bash
Install-Package Microsoft.EntityFrameworkCore.SqlServer -Version 3.1.10
```

```csharp
services.AddDbContext<ToDoContext>(options => options.UseSqlServer(Configuration.GetConnectionString("ToDoConnection")));
```

Delete your Migrations folder and then run the following:
```bash
Add-Migration ProdCreate -Context ToDoContext -OutputDir "Data/Migrations"
```

Publish your API to Azure with a new App Service, new Resource Group and new Hosting Plan (be sure to select Free). Configure the database to a new region using your Azure for Students Starter subscription.

![](https://i.imgur.com/tfSzvCl.png)

![](https://i.imgur.com/3nWkrn1.png)

![](https://i.imgur.com/wnXYqY1.png)

![](https://i.imgur.com/NHCbd7K.png)

![](https://i.imgur.com/33EL6YL.png)

![](https://i.imgur.com/U0jLv4e.png)

![](https://i.imgur.com/6Y5m0mU.png)

![](https://i.imgur.com/t6hLNkD.png)

![](https://i.imgur.com/cKTAfJ6.png)

---

<a name="3"></a>
## 3) Swagger and OpenAPI Specification
[swagger.io/docs](https://swagger.io/docs/specification/about/)
OpenAPI is a specification for design and development of REST API’s that allows a streamlined approach to describing your entire API. The spec defines a uniform way to define your API resources and methods, along with parameters and authentication.

Swagger is a set of open-source tools built around the OpenAPI Specification that make designing, building, documenting, and testing API’s more efficient.

### Installing Swagger Tools
We will use the Swashbuckle package in order to expose the necessary Swagger tools for our API.

Use the NuGet Package Manager to add the following to your application:
`Swashbuckle.AspNetCore`

### Add SwaggerGen Service
In order for the SwaggerUI to render our API doc we must first use SwaggerGen generate the JSON to define our api:

`Startup.cs`
```csharp
services.AddSwaggerGen(c => {
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "ToDo API", Version = "v1" });
});
```
```csharp
app.UseSwagger();
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "ToDo API V1");
    c.RoutePrefix = string.Empty;
});
```

Run your app and navigate to https://localhost:[YourPort] 

### Try out the Post (/api/ToDo) and then the Get(/api/ToDo):

![](https://i.imgur.com/75tTnbN.png)

![](https://i.imgur.com/f0Iy9Qz.png)

### Add a new Resource and Test Swagger
On your own add a new resource called Project with endpoints to (Get, Get/{id}, Post, Put, and Delete methods).

When you reload your application and navigate to the root path you should see something similar to the following:

![](https://i.imgur.com/VbZ8QtJ.png)

---

<a name="4"></a>
## 4) Customize Swagger
There are various ways that you can customize the swagger tools, here are some of the essentials as outlined in the .NET Core documentation.

### Add Optional Info to API Description
You can add some additional details to your Swagger by updating your AddSwaggerGen() method call with the following:
```csharp
c.SwaggerDoc("v1", new OpenApiInfo
{
    Version = "v1",
    Title = "ToDo API",
    Description = "My API with ToDos and Projects",
    TermsOfService = new Uri("https://www.bcit.ca/study/programs/699ccertt"),
    Contact = new OpenApiContact
    {
        Name = "Phil Weier",
        Email = string.Empty,
        Url = new Uri("https://www.bcit.ca/study/programs/699ccertt#staff"),
    },
    License = new OpenApiLicense
    {
        Name = "Use under LICX",
        Url = new Uri("https://example.com/license"),
    }
});
```

![](https://i.imgur.com/3ypntcI.png)

---

<a name="5"></a>
## 5) XML Comments
You will remember from our introduction to C# that there was a third type of comment syntax that we have not given much attention to…until now.

The use of /// before a line of code allows it to be referenced when generating documentation.

Add the following to your `.csproj` file:
```xml
<PropertyGroup>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>
```

### Add Summary Definition to a Method
The code above will enable the generation of your XML documentation in your bin folder.

Add something like the following just above the HttpDelete Attributes of both of your controllers:
```csharp
/// <summary>
/// Deletes a specific Project.
/// </summary> 
/// <param name="id"></param>
```

#### View the Generated XML
![](https://i.imgur.com/GMOb1at.png)

Update `Startup.cs` by adding the following at the bottom of the services.AddSwaggerGen() method.
```csharp
// Set the comments path for the Swagger JSON and UI.
var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
c.IncludeXmlComments(xmlPath);
```
See the Swagger Doc JSON…

![](https://i.imgur.com/qs1GEK0.png)

See the updated SwaggerUI…

![](https://i.imgur.com/v6dGT1F.png)

### Supress XML Warnings
When adding your comments in each controller, you may have noticed green squiggly underlines throughout your code:

![](https://i.imgur.com/C8iAXIO.png)

Update the previously added PropertyGroup in your .csproj to suppress those warnings:
```xml
<PropertyGroup>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <NoWarn>$(NoWarn);1591</NoWarn>
  </PropertyGroup>
```

### Add Remarks Definition to a Method
Remarks are a more robust way of adding detail to your method documentation. Add something similar to the following to your Create methods:
```csharp
/// <summary>
/// Creates a TodoItem.
/// </summary>
/// <remarks>
/// Sample request:
///
///     POST /Todo
///     {
///        "description": "Item1",
///        "isComplete": true,
///        "priority": 1,
///        "createdOn": "2020-01-01T00:00:00.0000001"
///     }
/// </remarks> 
/// <param name="todo"></param>
[HttpPost]
public IActionResult Create([FromBody] ToDo todo)
{
    if (todo.Description != null || todo.Description != "")
    {
        try {
            _db.ToDos.Add(todo);
            _db.SaveChanges();
        } catch (Exception e) {
            //Bad Request status code 400
            return BadRequest(e);
        }
        return new ObjectResult(todo);
    }
    return BadRequest();
}
```

![](https://i.imgur.com/lsr4XQ3.png)

### Add StatusCode Definitions to a Method
Often times we will want to customize and clearly define the possible HTTP status codes that may come from our API. This helps developers to anticipate the potential outcomes of a call to your API and better handle their integrations with your service.

Add the following to the bottom of your Create xml comments:
```csharp
/// <returns>A newly created Todo Item</returns>
/// <response code="201">Returns the newly created item</response>
/// <response code="400">If the item is not saved</response>
```

![](https://i.imgur.com/ac6qcPM.png)

Now decorate the Create methods with the following attributes:
```csharp
[ProducesResponseType(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
```

Change the return statement to the following:
```csharp
return CreatedAtRoute("GetTodo", new { id = todo.Id}, todo);
```

![](https://i.imgur.com/SWPga5J.png)

FYI, there is a shorthand version of adding these ProducesResponseTypes individually if you are willing to follow (or create your own) "conventions". Feel free to explore here: [Microsoft Docs: Web API Conventions](https://docs.microsoft.com/en-us/aspnet/core/web-api/advanced/conventions?view=aspnetcore-3.1)

### Add Media Type Definitions to Responses
You can define your response type at the class level and your status codes at the method level. While adding Attributes such as [ProducesResponseType()] at the method level…you can clearly define all response types to be “application/json” by decorating the controller with the following attribute:
```csharp
[Produces("application/json")]
[Route("api/[controller]")]
[ApiController]
public class ToDoController : ControllerBase
```

Now replace your ToDoController GetAll() method with the following:
```csharp
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
[HttpGet]
public IActionResult GetAll()
{
    try
    {
        var items = _db.ToDos.ToList();
        if (!items.Any())
        {
            return NoContent();
        }
        return Ok(items);
    }
    catch (Exception e) {
        return BadRequest(e);
    }
}
```
---

<a name="6"></a>
## 6) HTTP Status Codes
Status codes are a list of standardized codes that are provided as part of a web response to a client request. The codes fall within one of the 5 categories that are determined by the first digit of the code:
* 1xx informational response – the request was received, continuing process
* 2xx successful – the request was successfully received, understood and accepted
* 3xx redirection – further action needs to be taken in order to complete the request
* 4xx client error – the request contains bad syntax or cannot be fulfilled
* 5xx server error – the server failed to fulfill an apparently valid request

[Microsoft.AspNetCore.Http StatusCodes](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.statuscodes?view=aspnetcore-3.1)
