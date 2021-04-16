# .NET Essentials 2 - Session 4
## Learning Outcomes
* [Introduction to API Development](#1)
* [.NET Core Web API](#3)
* [Download and Install Db Browser for SQLite](#4)
* [List and Detail API Methods (GET)](#5)
* [Create API Method (POST)](#6)
* [Update API Method (PUT)](#7)
* [Delete API Method (DELETE)](#8)
* [Testing With Postman](#9)
* [Enabling Cross Origin Resource Sharing](#10)
* [Create a React Client to Consume the API](#11)
    * [Todos.js (GET)](#12)
    * [Create ToDo (POST)](#13)
    * [Delete a ToDo (DELETE)](#14)
    * [Update a ToDo (PUT)](#15)
    
---

<a name="1"></a>
## 1) Introduction to API Development
API stands for application programmer interface. An API over the web allows other developers to access services or resources from the API host. In other words, developers can write code in their preferred platform to call remote API methods. These methods usually return JSON but sometimes they return XML or other resources. 

### HTTP Requests
When making requests for web services four common types of HTTP requests are made:

| Method | Action |
| -------- | -------- |
| GET     | Retrieves all or one items |
| POST    | Creates an item |
| PUT     | Updates an item |
| DELETE  | Removes an item |

PUT is similar to POST but PUT is for changes to an existing resource.  If a PUT occurs twice the second request has no effect on an identical object.   A second POST will try to create another object.

---

<a name="3"></a>
## 2) .NET Core Web API
Web API was introduced by Microsoft in ASP.NET 4 to provide a simple and automatic delivery of RESTful content in either XML or JSON format. 

Web API: 
* automatically generates JSON, XML from your database content.
* can reside in its own project or in the same web MVC project or web forms project.
* is more lightweight than an MVC project.

### Create a .NET Core Web API application
 
#### Install the following packages
```bash
Install-Package Microsoft.EntityFrameworkCore  -Version 3.1.10
Install-Package Microsoft.EntityFrameworkCore.Tools -Version 3.1.10
Install-Package Microsoft.EntityFrameworkCore.Sqlite -Version 3.1.10
```

#### Define your model(s) and DbContext
```csharp
public class ToDo 
{
    [Key, DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    public string Description { get; set; }
    public bool IsComplete { get; set; }
}
```

```csharp
public class ToDoContext: DbContext
{
    public ToDoContext(DbContextOptions<ToDoContext> options) : base(options) { }
    public DbSet<ToDo> ToDos { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<ToDo>().HasData(
            new { Id = 1, Description = "Clean house", IsComplete = false },
            new { Id = 2, Description = "Bake cake", IsComplete = false }
        );
    }
}
```
#### Add connection string and register in startup
We are going to use a Sqlite database which is a document within your project folder.

```json
"ConnectionStrings": {
    "ToDoConnection": "Data Source=.\\Data\\ToDo.db"
},
```

Don't forget to register the context in your Startup.cs file:
```csharp
services.AddDbContext<ToDoContext>(options => options.UseSqlite(Configuration.GetConnectionString("ToDoConnection")));
            
```

#### Add an InitialCreate migration and run the migration.
```bash
Add-Migration InitialCreate -OutputDir "Data/Migrations"
Update-Database
```

### Exercises
1. Add an int property called Priority to the ToDo model class and add Priority = 1 and Priority = 3 respectively to the two items in the OnModelCreating method. Add a migration called AddedPriority and run the migrations.

2. Add a DateTime property called CreatedOn to the ToDo model class and assign CreatedOn = DateTime.Now to both items in the OnModelCreating method. Add a migration called AddedCreateOn and run the migrations.

<a name="4"></a>
### Download and Install Db Browser for SQLite
In order to easily review and manage your sqlite db I (and Microsoft) recommend: [DB Browser for SQLite](https://sqlitebrowser.org/dl/)

Use the Open Database and Browse Data tabs...when done, remember to always Close Database.

![](https://i.imgur.com/Kjysx9l.png)


<a name="5"></a>
## 3) List and Detail API Methods (GET)
In WebAPI, controllers are usually set up to handle web requests for a single entity or view model type. To enable WebAPI for ToDos, right click the controllers folder and choose **Add | Controller**, select **API controller – Empty** and then click **Add**.

![](https://i.imgur.com/OaAyE1V.png)


When prompted, name the controller as ToDoController and click Add:

![](https://i.imgur.com/BRGjAOa.png)


Use dependancy injection to add the context in the controller:
```csharp
private readonly ToDoContext _db;

public ToDoController(ToDoContext db)
{
    _db = db;
}
```

```csharp
// GetAll() is automatically recognized as
// http://localhost:<port #>/api/todo
[HttpGet]
public IActionResult GetAll()
{
   try
   {
      var items = _db.ToDos.ToList();
      if (!items.Any()
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

```csharp
// GetById() is automatically recognized as
// http://localhost:<port #>/api/todo/{id}
// For example:
// http://localhost:<port #>/api/todo/1
[HttpGet("{id}", Name = "GetTodo")]
public IActionResult GetById(long id)
{
   var item = _db.ToDos.FirstOrDefault(t => t.Id == id);
   if (item == null)
      return NotFound(id);
   return Ok(item);
}
```

### Test the API in the Browser
To save time, update your launch url within your launchSettings.json to the following:
```json
"launchUrl": "api/todo",
```

If you run your api, a browser will open to the `https://localhost:<port #>/api/todo` url and you will see a full listing of all ToDo’s in the database.
 
You will see only one specific record when you go to this address:
`https://localhost:<port #>/api/todo/1`

***Notice that the attribute values returned by web api are camel-cased and not Pascal cased like in the C# code. You will need to adjust your client code accordingly when working with the attribute values that are sent in the response.***

___

<a name="6"></a>
## 4) Create API Method (POST)
Page requests which send complex data sets to the server usually use POST requests. The POST request data is usually contained in the body of the request. .NET allows you to group attribute values of a pre-defined complex custom type when you use the `[FromBody]` attribute in the parameter list.

In this case, all attributes in the JavaScript which match the name and case of attributes in the ToDo class are bundled together in one object. Being able to receive an object as a parameter is very convenient when working in C#.

To try this out, add the following Create() method to the code. Create() receives an object from a JavaScript client and saves it to the database. The object returned to the client contains the Id that is generated when it is added to the database.

```csharp
[HttpPost]
public IActionResult Create([FromBody]ToDo todo) {
    if (todo.Description == null || todo.Description == "") {
        return BadRequest();
    }
    _context.ToDos.Add(todo);
    _context.SaveChanges();
    return new ObjectResult(todo);
}
```

___

<a name="7"></a>
## 5) Update API Method (PUT)
As explained earlier, POST methods can be used to create objects. However, PUT methods are the proper way to update existing objects.  (You can update objects using a POST though if you really wanted). 

The following example shows the code needed to update the ToDo list with a PUT request.  Notice also here that a Route attribute is used. The Route attribute indicates that myedit must be used in the URL.
 
For this case, notice the only item that can be edited is the IsComplete attribute. The ability to modify other attributes can be added if required.
```csharp
[HttpPut]
[Route("MyEdit")] // Custom route
public IActionResult GetByParams([FromBody]ToDo todo)
{
    var item = _context.ToDos.Where(t => t.Id == todo.Id).FirstOrDefault();
    if (item == null)
    {
        return NotFound();
    }
    else
    {
        item.IsComplete = todo.IsComplete;
        _context.SaveChanges();
    }
    return new ObjectResult(item);
}
```

___

<a name="8"></a>
## 6) Delete API Method (DELETE)
The code for deleting the item from the database can be added to the Todo controller as well. Here we are using the HttpDelete attribute to enable a DELETE request. The Route attribute requires the URL to include the link to the MyDelete method.
This example shows how to enable a DELETE request by adding this code to the Todo controller. Notice too that the URL parameters are used to capture the parameter value and the `[FromBody]` attribute is absent.
```csharp
[HttpDelete]
[Route("MyDelete")] // Custom route
public IActionResult MyDelete(long Id)
{
    var item = _context.ToDos.Where(t => t.Id == Id).FirstOrDefault();
    if (item == null)
    {
        return NotFound();
    }
    _context.ToDos.Remove(item);
    _context.SaveChanges();
    return new ObjectResult(item);
}
```

___

<a name="9"></a>
## 7) Testing With Postman
When you are trying to figure out how to code your Web API controller, you may not have a JavaScript client handy to test it.  However, you can use an application like Postman to test page requests.  Postman provides a nice objective way to connect to your API so you don’t have to worry about your own JavaScript errors and you also will not have to deal with any CORS issues (more about that later).

You can download and install Postman from here:
[https://www.postman.com/downloads](https://www.postman.com/downloads/)


### Test all of your API Endpoints
Use Postman to test all of your endpoints (be sure that your api is running locally):

## GET
![](https://i.imgur.com/oIFvyux.png)

## POST
![](https://i.imgur.com/8BxynIi.png)

## PUT
![](https://i.imgur.com/buD6ceJ.png)

## DELETE
![](https://i.imgur.com/Xi2oWIw.png)

___

### Certificate Verification Error
You may receive the error “Could not get any response”. If you receive this error, try the adjustment that is highlighted in green below.

![](https://i.imgur.com/XaNlJEi.png)
 
You can click on the Gear icon and go to Settings | General. Then turn off SSL certificate verification.

![](https://i.imgur.com/KGJfC1s.png)

___

<a name="10"></a>
## 8) Enabling Cross Origin Resource Sharing
Once you are satisfied that your Web API works using a tool like Postman, you can develop a client separately to interact with the API.  Your client could be written in any language you want: C++, Python, Java, JavaScript, React, Angular, JQuery, Swift – anything you want.

However, to enable any web browser-based application to connect with a server on a different domain, Cross Origin Resource Sharing (CORS) must be enabled in the server application. Without enabling CORS, the web browsers will reject the transfer of JSON to and from a server – this is a standard security precaution that web browser vendors take to help prevent third party developers from creating scripts which upload sensitive data from your browser to a rogue server.
```bash
Install-Package Microsoft.AspNetCore.Cors
```

Configure our app to use CORS in the Startup.cs file.
```csharp
app.UseCors("AllowAll");
```

After, in the ConfigureServices() method, add the following code before AddControllers() is called:
```csharp
// Call this before AddControllers()
services.AddCors(options =>
{
    options.AddPolicy("AllowAll",
        builder =>
        {
            builder
            .AllowAnyOrigin()
            .AllowAnyMethod()
            .AllowAnyHeader();
        });
});
```

CORS should now be enabled on your server. Now you are ready to test the server API with a JavaScript client.

<a name="11"></a>
## 9) Create a React Client to Consume the API
Create a React client to test our API endpoints with CORS enabled.

![](https://i.imgur.com/iKVfxjI.png)

### Install Additional Dependencies
react-router-dom has the react router bindings to allow us to set routes to our components in a navigation bar:
![](https://i.imgur.com/AAlip8J.png)

bulma is a free open source css framework:
![](https://i.imgur.com/PAMcBrf.png)

 
Add the following to index.js
```javascript
import 'bulma/css/bulma.min.css';
```

### Add Navigation Driven by App.js
Overwrite your app.js with the following App Component that renders a routing navbar.
```javascript
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import './App.css';
import Navbar from './components/Navbar';
import Home from './components/Home';
import Todos from './components/Todos';

function App() {
  return (
    <div className="App">
        <Router>
          <div>
            <Navbar />
            <Switch>
              <Route exact path="/" component={Home} />
              <Route exact path="/todos" component={Todos} />
            </Switch>
          </div>
        </Router>
      </div>
  );
}

export default App;

```

### Add Components
Add a components folder to your src and create the following js files:
`Home.js`
```javascript
import React from 'react';

export default function Home() {
  return (
    <div>
        <h1>Welcome</h1>
    </div>
  )
}
```

### Navbar.js
```javascript
import React from 'react';

export default function Navbar() {
  return (
    <nav className="navbar" role="navigation" aria-label="main navigation">
        <div id="navbarBasicExample" className="navbar-menu">
          <div className="navbar-start">
            <a href="/" className="navbar-item">
              Home
            </a>
            <a href="/todos" className="navbar-item">
              To Dos
            </a>
          </div>
        </div>
      </nav>

  )
}
```
<a name="12"></a>
### Todos.js (GET)

*Note: Be sure to use the appropriate property names and url to your api which must be running to test.
```javascript
import React, { useState, useEffect } from 'react';
const BASE_URL = 'https://localhost:44326/api/';

export default function Todos() {
    const [todos, setTodos] = useState([]);
    
    const fetchTodos = () => {
        fetch(BASE_URL+'todo')
        .then(response => response.json())
        .then((data) => {
            setTodos(data);
        })
        .catch ((err) => {
            console.log(`An error has occurred: ${err}`);
      });
    }
    
    useEffect(() => {
        fetchTodos();
    }, []);
    // add empty [] dependancy list to stop infinite loop
    //https://dev.to/ari_o/the-traps-of-useeffect-infinite-loops-836
    
    return (
        <div>
            <h1>To Do</h1>
            <table className="table is-fullwidth">
                <thead>
                    <tr>
                        <th>Is Complete</th>
                        <th>Id</th>
                        <th>Description</th>
                        <th>Priority</th>
                        <th>Created On</th>
                    </tr>
                </thead>
                <tbody>
                    {todos.map(todo =>
                        <tr key={todo.id}>
                            <td>{todo.isComplete && (<span>Yes</span>)}</td>
                            <td>{todo.id}</td>
                            <td>{todo.description}</td>
                            <td>{todo.priority}</td>
                            <td>{todo.createdOn}</td>
                        </tr>
                    )}
                </tbody>
            </table>
        </div>
    )
}
```
<a name="13"></a>
### Create ToDo (POST)
We can update our Todo.js component to handle the addition of a ToDo via the API Post method. We will need to add input(s) and a submit button as well as an onClick method that will trigger the POST fetch with the user’s input.

Add a state variable and input handler to capture the user input:
```javascript
const [description, setDescription] = useState('');

const handleDescriptionInput = e => {
        setDescription(e.target.value);
    };
```

Add the following to the JSX above the table of Todos:
```html
<input type="text" value={description} onChange={handleDescriptionInput} />
<button className="button is-success is-light is-small" onClick={createToDo}>Create</button>
```

Add the createToDo method that sends the post request:
```javascript
// Executes when button pressed.
const createToDo = () => {
    fetch(BASE_URL+'todo', {
        method: 'POST',
        headers: {
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            IsComplete:  false, // Set default to false.
            Description: description,
            Priority: 2,
            CreatedOn: new Date()
        })
    })
    // Response received.
    .then(response => response.json())
        // Data retrieved.
        .then(json => {
            console.log(JSON.stringify(json));
            setDescription(''); // Clear input. 
            fetchTodos();
        })
        // Data not retrieved.
        .catch(function (error) {
            console.log(error);
        }) 
}
```


<a name="14"></a>
### Delete a ToDo (DELETE)
Add a delete method to the component:
```javascript
const deleteToDo = (id) => {
    fetch(BASE_URL + 'todo/MyDelete?Id=' + id, {
        method: 'DELETE',
        headers: {
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        }
    })        
    .then(response => response.json())
        // Data retrieved.
        .then(json => {
            console.log(JSON.stringify(json));
            fetchTodos();
        })
        // Data not retrieved.
        .catch(function (error) {
            console.log(error);
        })
}
```

Add a delete link in the JSX table to trigger the delete. (Don’t forget to add a corresponding empty `<th>`)
```javascript
<td><button className="button is-danger is-light" onClick={() => deleteToDo(todo.id)}>Delete</button></td>
```
<a name="15"></a>
### Update a ToDo (PUT)
Add an updateToDo method to the component class:
```javascript
const updateToDo = (id, checked) => {
    fetch(BASE_URL+'todo/myedit', {
        method: 'PUT',
        headers: {
            'Accept': 'application/json',
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            Id: id,
            IsComplete: checked
        })
    })     
    .then(response => response.json())
    // Data retrieved.
    .then(json => {
        console.log(JSON.stringify(json));
        fetchTodos();
    })
    // Data not retrieved.
    .catch(function (error) {
        console.log(error);
    })
}
```

And update the "Is Complete" td in the JSX table to trigger the api call onChange:
```javascript
<td>
    <input type='checkbox' 
    value={todo.isComplete} 
    checked={todo.isComplete} 
    onChange={(e) => updateToDo(todo.id, e.target.checked)} />
</td>
```

