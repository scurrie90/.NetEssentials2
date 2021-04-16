# .NET Essentials 2 - Session 1

## Introduction to Identity Authentication
[Microsoft Docs: Intro to Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-5.0&tabs=visual-studio)

## Add Identity to Existing Projects
When possible, you should consider your needs for both Authentication and Authorization **BEFORE** creating your project. Adding your auth model earlier in the development process will make it easier to ensure that it is implemented appropriately and in all intended places without gaps. That said, requirements can often change and you may need to implement an auth model to an existing project. Here are the steps to add Identity Authentication to your existing .Net Core Web Application.

### Use the built in scaffolding feature
Visual Studio CodeGeneration can not only enable you to scaffold in EntityFramework CRUD methods and views, but can also allow you to "scaffold in" the required parts of Identity (External Razor Pages Library).

1. Right click on the Project node in solution explorer and select `Add->New Scaffolded Item...`
2. On the left side panel you should see an Identity option, select it and click Add.
 
![](https://i.imgur.com/C4EKoDH.png)

3. Select the Razor Pages that you want to import into your project. Here you can also customize things like Layout, Data Context class, and User class, for now simply:
    * Leave the Layout empty
    * Override all files
    * Add a new Data Context like `AuthDbContext`
    * Leave the user class
Once added this will add the necessary dependancies and code files to your project. Inspect the Areas/Identity folder that has been added your project.

4. To make this app have a similar setup to if you would have added Identity during project creation, move the following (similar) lines from `IdentityHostingStartup.cs` to `Startup.cs`:
```csharp
services.AddDbContext<AuthDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("AuthConnection")));

services.AddDefaultIdentity<IdentityUser>(options =>
        options.SignIn.RequireConfirmedAccount = true)
    .AddEntityFrameworkStores<AuthDbContext>();
```
You can now delete `IdentityHostingStartup.cs`.

5. Update your connection string in `appsettings.json`:
```json
"ConnectionStrings": {
    "AuthConnection": "Server=Your\\ServerName;Database=YourDbName;Trusted_Connection=True;MultipleActiveResultSets=true"
  }
```

6. Create Your Auth Database
#### Add Migration
```bash
Add-Migration InitialAuthSchema -Context AuthDbContext -OutputDir "Areas/Identity/Data/Migrations"
```
#### Run Migration
```bash
Update-Database -Context AuthDbContext
```

7. Finally, add the login and register links to your `_Layout.cshtml`
```htmlmixed
<div class="navbar-collapse collapse d-sm-inline-flex flex-sm-row-reverse">
    <partial name="_LoginPartial" />
    <ul class="navbar-nav flex-grow-1">
    ...
    </ul>
</div>
```
---

## Afternoon Exercise
Update your SSDeShopOnWeb App so that:
* any user can view the home (list of products) page both as a logged in user or a non-logged in user
* any user can add items to a shopping cart (logged in user carts are additionally updated in the db)
* a user can "Checkout" from their Cart page
 * if not signed in, they are forced to register and/or sign in
 * a simple landing page Thanking them for Checking out that shows the cart items and total amount

### BONUS

Clone the actual eShopOnWeb project and start it locally.

Review the Checkout process and other features such as Brands and start to implement those features (ON BRANCHES).
