# .NET Essentials 2 - Session 6 (Server)
## Learning Outcomes

*  [Bearer Tokens for Authorization](#1)
*  [Configuring JWT Tokens on the Server](#2)
*  [Add Token Generation to Login](#3)
*  [Add JWT Token Authorization to Api GetAll](#4)

---

<a name="1"></a>
# 1) Bearer Tokens for Authorization
To manage authenticated sessions when there is a separate client and server it is common to use bearer tokens which are also known as JSON Web Tokens (Jwt). Web tokens allow the client application to verify to the server that the user is logged in.

To manage verification the server can issue a unique token to track the identity of the user during the course of their authenticated session. A token is just a long unique string of un-guessable characters. For example:
“asdfsafal;jxs9q3asdf2304ladsf932-as-93daa9ds2ff” is a token.

Both the server and client store a copy of the token during their session. The client then sends the token with every request during the session to identify itself.

## Managing Logged In Sessions Between Separate Client and Server Applications
The following diagram shows typical steps taken to manage logged in sessions between separate client and server applications.

![](https://i.imgur.com/bnrUYqK.png)

<a name="2"></a>
## 2) Configuring JWT Tokens on the Server
Add Identity if it doesn't already exist in your web api:

![](https://i.imgur.com/zPekRSy.png)

You do not need to override any files (unless you want to debug the auth).
Add an "AuthContext":

![](https://i.imgur.com/qFEV7uK.png)

You can delete the generated AuthContextConnection in `appsettings.json` and for simplicity, simply add your existing ToDoConnection to the `IdentityHostingStartup.cs`:

![](https://i.imgur.com/aKt0iVT.png)

Migrate your AuthContext into your existing ToDoApi database:
```bash
Add-Migration InitialAuthSchema -Context AuthContext -OutputDir "Areas/Identity/Data/Migrations"
```

```bash
Update-Database -Context AuthContext
```

![](https://i.imgur.com/lCX9aFE.png)


Add the following User Secrets:
```json
"JWT_SITEKEY": "ThisIsMySecretKey",
"JWT_ISSUER": "https:///localhost:44326//"
  ```

Install the JwtBearer Package:

![](https://i.imgur.com/2Qv38sa.png)

Add the following **BEFORE** services.AddDefaultIdentity<>():
```csharp
JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();

services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = context.Configuration["JWT_ISSUER"],
                ValidAudience = context.Configuration["JWT_ISSUER"],
                IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(context.Configuration["JWT_SITEKEY"]))
            };
        });
```

<a name="3"></a>
## 3) Add Token Generation to Login
We will want to add endpoints for our api users to login and logout. During this process we will send an recieve/validate the JWT tokens.

We will add simple login/logout forms to our client application once we have tested our server side functionality.

### Add a View Model To Define Form Data
Add the following VM so that we can bind our login requests to a model:
```csharp
public class LoginVM
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [DataType(DataType.Password)]
    public string Password { get; set; }

    [Display(Name = "Remember me?")]
    public bool RememberMe { get; set; }
}
```

### Add a Controller to Handle Your Auth Requests
Add an Empty Api Controller called `AuthController`. Add the following properties and constructor:

```csharp
// Route will be https://localhost:<port #>/Auth
[Route("[controller]")]
[ApiController]
public class AuthController : Controller
{
    private readonly SignInManager<IdentityUser> _signInManager;
    private readonly UserManager<IdentityUser> _userManager;
    private IConfiguration _config;
    private IServiceProvider _serviceProvider;

    public AuthController(SignInManager<IdentityUser> signInManager,
    UserManager<IdentityUser> userManager,
    IConfiguration config,
    IServiceProvider serviceProvider)
    {
        _signInManager = signInManager;
        _userManager = userManager;
        _config = config;
        _serviceProvider = serviceProvider;
    }
    
}
```

Add a Json package to help handle our tokens:

```bash
Install-Package Microsoft.AspNetCore.Mvc.NewtonsoftJson  -Version 3.1.10
```

Update the following in the startup.cs file where AddControllers is called:
```csharp
services.AddControllers().AddNewtonsoftJson();
```

Add a helper method to your AuthControler that will generate the tokens:


```csharp
string GenerateJSONWebToken(IdentityUser user)
{
    var securityKey
        = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["JWT_SITEKEY"]));
    var credentials
        = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);
    var claims = new List<Claim> {
        new Claim(JwtRegisteredClaimNames.Sub, user.Email),
        new Claim(JwtRegisteredClaimNames.Jti,
                    Guid.NewGuid().ToString()),
        new Claim(ClaimTypes.NameIdentifier, user.Id)
    };

    var token = new JwtSecurityToken(_config["JWT_ISSUER"],
        _config["JWT_ISSUER"],
        claims,
        expires: DateTime.Now.AddMinutes(120),
        signingCredentials: credentials);
    return new JwtSecurityTokenHandler().WriteToken(token);
}
 
```
Add the Register handler:
```csharp
[HttpPost("Register")]
public async Task<JsonResult> RegisterAsync([FromBody] RegisterVM registerVM)
{
    if (ModelState.IsValid)
    {
        //will need to implement email or turn off email verification
        var user = new IdentityUser { UserName = registerVM.Email, Email = registerVM.Email, EmailConfirmed = true };
        var result = await _userManager.CreateAsync(user, registerVM.Password);
        if (result.Succeeded)
        {
            return await SignInAsync(new LoginVM() { Email = registerVM.Email, Password = registerVM.Password });
        }
    }
    dynamic jsonResponse = new JObject();
    jsonResponse.token = "";
    jsonResponse.status = "Registration Failed";
    return Json(jsonResponse);
}
```
Add the Login handler:
```csharp
[HttpPost("Login")]
public async Task<JsonResult> SignInAsync([FromBody] LoginVM loginVM)
{
    dynamic jsonResponse = new JObject();
    if (ModelState.IsValid)
    {
        var result = await
                    _signInManager.PasswordSignInAsync(loginVM.Email.ToUpper(),
                    loginVM.Password, loginVM.RememberMe, lockoutOnFailure: true);
        if (result.Succeeded)
        {

            var user = await _userManager.FindByEmailAsync(loginVM.Email);

            if (user != null)
            {
                var tokenString = GenerateJSONWebToken(user);
                jsonResponse.token = tokenString;
                jsonResponse.status = "OK";
                return Json(jsonResponse);
            }
        }
        else if (result.IsLockedOut)
        {
            jsonResponse.token = "";
            jsonResponse.status = "Locked Out";
            return Json(jsonResponse);
        }
    }
    jsonResponse.token = "";
    jsonResponse.status = "Invalid Login";
    return Json(jsonResponse);
}
```

### Test in Swagger

<img src="https://i.imgur.com/8JPifM6.png" width="300" />
<img src="https://i.imgur.com/OnxNc0Q.png" width="600" />

---

<img src="https://i.imgur.com/EQZtpNK.png" width="300" />
<img src="https://i.imgur.com/BgjHvq2.png" width="600" />

### Test in Postman

![](https://i.imgur.com/T8JKkoe.png)

---

![](https://i.imgur.com/0hxw2JF.png)


<a name="4"></a>
## 4) Add JWT Token Authorization to Api GetAll
Once a valid token is received, it will only be possible to access the protected data when sending the token in the header of a GET request.

Similar to our Razor Pages app with Authentication, we simply have to decorate the class (or individual members in this case) with the Authorize attribute.

### Add the Authorize Attribute to your End Point
Decorate your GetAll method of the ToDoController with the following attribute (typically Authorize)
```csharp
// Since we have cookie authentication and Jwt authentication we must
// specify that we want Jwt authentication here.
[Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
```

Without the Authorization Token, the result will look like this:

![](https://i.imgur.com/SqRZWfo.png)

### Access The Authenticated Users Name and Id
Add this in side any Authorized method and you can access the users username and id:

```csharp
// access the claim from the HttpContext
var claim = HttpContext.User.Claims.ElementAt(0);
// use the claim to get the aspnetusers username
string userName = claim.Value;
// aspnetusers id
var userId = this.User.FindFirstValue(ClaimTypes.NameIdentifier);
```

### Test in Postman
Here we are using the Authorization tab of Postman. In the Type dropdown select "Bearer Token" and the Token field, paste the token value returned during Login.

![](https://i.imgur.com/8C29A4M.png)

This results in the following being added to your request Header (Note the Value is "Bearer eyjhb..." this is the word Bearer, followed by a space, followed by the token):

![](https://i.imgur.com/TmS5RI0.png)

### Further Testing
Try changing the value of your JWT_SITEKEY or JWT_ISSUER and run the same GET request. Change them back to the original values and run the GET request again.

Go to [https://jwt.io/](https://jwt.io/) and paste your token into the Encoded window and you can see the Decoded values (other than your secret key).
