# .NET Essentials 2 - Session 2
## Learning Outcomes
* [Adding Authentication with Roles](#1)
* [Preventing Brute Force Attacks](#2)
    * [Block Account For a Period of Time (Lockout)](#3)
    * [Delay Login Requests](#4)
    * [Use Google reCAPTCHA](#5)
    * [Two-Factor Authentication](#6)

---

<a name="1"></a>
## 1) Adding Authentication with Roles
### Scaffold in Identity
[.Net Core Identity](https://github.com/dotnet/AspNetCore/tree/master/src/Identity) is a membership based authentication system for ASP.NET Core web applications.

It is possible to implement Identity as an external Razor Pages Library, however for our purposes of debugging and customization we will scaffold it into our project.

![](https://i.imgur.com/U9Ro7Sj.png)

![](https://i.imgur.com/lQnWAnr.png)

For simplicity, we will "Override all files".

If you added Authentication when creating your project or have previously scaffolded Identity into your project, set the Data context class. If you are adding Identity Authentication for the first time in your project, click the "+" to add your Auth Context.

![](https://i.imgur.com/jXQAXLU.png)

<img src="https://i.imgur.com/ETiDUxB.png" width="350" />

---

**Note - If you have not done so already, ensure that:**
1. your auth connection string is set
2. your auth context is registered in a Startup file
3. you have run your auth migrations to create your auth database
4. ensure that `app.UseAuthentication();` exists inside the `Configure` method of the `Startup.cs` file

---
### Require Auth on a Page
In order to make a page require a user to be logged in to view it, simply add the `[Authorize]` attribute to the class.

```csharp
// example in the Privacy.cshtml.cs file
[Authorize]
public class PrivacyModel : PageModel
{ ...

```
**Note - Ensure that `app.UseAuthorization();` exists inside the `Configure` method of the `Startup.cs` file. This line must be added after `app.UseAuthentication();` but before `app.UseEndpoints()`.**

```csharp
// example at the bottom of the Configure method of the Startup.cs file...
app.UseAuthentication();
app.UseAuthorization();

app.UseEndpoints(endpoints =>
{
    endpoints.MapRazorPages();
});
```

If an unauthenticated user navigates to the Privacy page, they will be forwarded to the Login page.

![](https://i.imgur.com/YzxJdQF.png)

Note the addition of the ReturnUrl parameter in the url above that is referenced in the `Login.cshtml.cs` below:

![](https://i.imgur.com/YWrByxS.png)

### Add Roles to Auth Requirement
Identity also allows for the basic authentication to be extended with role based auth. In order to enable this in your application you must first register it in the Startup file:

![](https://i.imgur.com/aVFzS0V.png)

You can now optionally enhance the Authorization requirements within your code with the following:

Member of ANY role listed:
```csharp
[Authorize(Roles = "RoleName1, RoleName2")]
public class ClassNameModel : PageModel
```

Member of ALL roles listed:
```csharp
[Authorize(Roles = "RoleName1")]
[Authorize(Roles = "RoleName2")]
public class ClassNameModel : PageModel
```

In MVC applications you can also enhance on individual ActionMethods of a Controller:
```csharp
[Authorize(Roles = "Administrator, PowerUser")]
public class ControlPanelController : Controller
{
    //must be admin OR power to SetTime
    public ActionResult SetTime()
    {
    }
    
    //only admin can shutdown
    [Authorize(Roles = "Administrator")]
    public ActionResult ShutDown()
    {
    }
}
```

#### Test Role Based Auth
Ensure that you have registered at least two Users and they are visible in your `AspNetUsers` db table. Insert a Role into your `AspNetRoles` db table and add a record to the `AspNetUserRoles` db table for one of your users.
Unauthenticated users should be forwarded to the Login page, the user with the role should be redirected back to see the page with role based auth, while the user without the role should be redirected to an AccessDenied page.

---

<a name="2"></a>
## 2) Security against Brute Force Attacks
A brute force attack is a repeated login attempt. In this attack, the attacker cycles through a “password and user” library to guess a login and password combination.
There are some measures you can consider to prevent the brute force attack. Most of the time these measures can be effective but you cannot stop this attack altogether and it is important to understand the trade-offs.

Here are a few common considerations:

| Action | Trade-Off |
| -------- | -------- |
| Block by IP address     | Usually done on a case by case basis and after some logging and validation of misuse. Some tools allow attackers to spoof IP’s which can fool a block-by-IP scheme. |
| Block specific account for a period of time | Attackers can force denial of service (DoS) on certain accounts by locking them out on purpose.|
| Delay response for login request | An attacker from multiple attack points will not be affected as much. |
| Use captcha | Captcha can be by-passed with advanced attackers. Users may not like the extra steps required to use captcha for some sites. |
| Two-Factor Authentication | Similar to captcha, some users may not like the extra steps required for login. |

We will look at basic implementations for the last four.
<a name="3"></a>
### 3) Block Account For a Period of Time (Lockout)
One of the most basic but important features of a web framework to prevent brute force attacks is the ability to freeze an account after a number of failed login attempts for an amount of time. The identity framework provides this feature so user accounts are frozen after five consecutive failed attempts.

`Login.cshtml.cs`

![](https://i.imgur.com/6Hzn2ZY.png)

For ease of testing, update the Configure method of `IdentityHostingStartup.cs`:
```csharp
public void Configure(IWebHostBuilder builder)
{
    builder.ConfigureServices((context, services) =>
    {
        services.Configure<IdentityOptions>(options =>
        {
            options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromSeconds(10);
            options.Lockout.MaxFailedAccessAttempts = 3;
            options.Lockout.AllowedForNewUsers = true;
        });
    });
}
```

<a name="4"></a>
### 4) Delay Login Requests
To help stop brute force attacks, in addition to methods discussed so far, it is also recommended that you also add a short delay between the time a person submits their login request and your application response.  This delay will help to slow down a dictionary attack where requests are made in rapid succession.  You will notice that response delays are implemented at banks, pay pal, and institutions where security is critical.
While other security measures are also needed to stop brute force attacks, adding a delay between the request and response at login is easy and this measure should always be included with your security measures.

Add the following as the first line of the OnPost method in `Login.cshtml.cs`:
```csharp
// add 2 second delay to login to slow brute force attacks
System.Threading.Thread.Sleep(2000);
```

<a name="5"></a>
### 5) Use Google reCAPTCHA
[Register your site with Google](https://www.google.com/recaptcha/)

#### V2
Add values similar to the following
![](https://i.imgur.com/WMxaLDk.png)

Click Submit and copy your Keys to be added to your application:

![](https://i.imgur.com/K0ZSIx1.png)

Add your keys to your appsettings.json file (ensure that this file is in your .gitignore and changes are not being tracked):
```json
"RecaptchaV2": {
    "SiteKey": "...",
    "SecretKey": "..."
  }
```

Install a reCAPTCHA package to handle Google reCAPTCHA V2 requests, I recommend the following:

![](https://i.imgur.com/W1oVXWw.png)


Register the configuration as a service in the `Startup.cs` file:

```csharp
//using PaulMiami.AspNetCore.Mvc.Recaptcha;
services.AddRecaptcha(new RecaptchaOptions {
    SiteKey = Configuration["RecaptchaV2:SiteKey"],
    SecretKey = Configuration["RecaptchaV2:SecretKey"]});
```

You can make a namespace accessible in all your views by adding it to your `_ViewImports.cshtml` like so:
```csharp
@addTagHelper *, PaulMiami.AspNetCore.Mvc.Recaptcha
```

Add the following to contain the reCAPTCHA V2 within a form:
```html
<div id="ReCaptchaV2Container"></div><br />
<label id="lblCaptchaMessage"></label><br />
```
And add the following within the page scripts to handle the above elements:
```html
<script src="https://www.google.com/recaptcha/api.js?onload=renderRecaptcha&render=explicit" async defer></script>
<script src="https://code.jquery.com/jquery-3.2.1.min.js"></script>

<script type="text/javascript">
    // Style and draw reCAPTCHA.
    var renderRecaptcha = function () {
        grecaptcha.render('ReCaptchaV2Container', {
            'sitekey':  '@Model.CaptchaSiteKey',
            'callback': reCaptchaCallback,
            theme:      'light',    //light or dark
            type:       'image',    // image or audio
            size:       'normal'    //normal or compact
        });
    };

    var reCaptchaCallback = function (response) {
        // If reCAPTCHA is successful display it.
        if (response !== '') {
            jQuery('#lblCaptchaMessage').css('color', 'green').html('Success');
            $(':input[type="submit"]').prop('disabled', false);
        }
    };

    // Check reCAPTCHA validation.
    jQuery('button[type="button"]').click(function (e) {
        var message = 'Please check the checkbox';
        if (typeof (grecaptcha) != 'undefined') {
            var response = grecaptcha.getResponse();
            (response.length === 0) ? (message = 'Captcha verification failed')
                                    : (message = 'Success!');
        }
        jQuery('#lblCaptchaMessage').html(message);
        jQuery('#lblCaptchaMessage').css('color',
                              (message.toLowerCase() == 'success!') ? "green" : "red");
    });

    // Disable button when form loads.
    $(document).ready(function () {
        $(':input[type="submit"]').prop('disabled', true);
    });
</script>
```

Finally, enable the reCAPTCHA behaviour and pass the necessary key value from the server to the client as @Model.CaptchaSiteKey:
```csharp
using Microsoft.Extensions.Configuration;
using PaulMiami.AspNetCore.Mvc.Recaptcha;

// ...

// Add this attribute to enforce the Recaptcha 
// behaviour from the PaulMiami package
[ValidateRecaptcha]
public class ClassNameModel : PageModel

// ... 

// use dependency injection to access configuration var SiteKey
private readonly IConfiguration _configuration;
// don't forget to add to the constructor

// ...

// add the Model property to hold the SiteKey
public string CaptchaSiteKey { get; set; }

// ...

// set the SiteKey value in the OnGet
CaptchaSiteKey = _configuration["RecaptchaV2:SiteKey"];

// ...

// Reset the site key in the OnPost if there is an error.
CaptchaSiteKey = _configuration["RecaptchaV2:SiteKey"];
```
---

#### V3
V3 allows Google to judge a users behaviour on your site to determine how "human" they are...while neither approach is perfect, V2 places more of the owness on the user, while V3 places more responsibility on the site administrator, ie. how should the site respond based on varying scores provided by Google.
Here is the easiest way to implement reCAPTCHA V3 within a .NET Core 3.1 app in my opinion:

Register a new site on Google to implement reCAPTCHA V3:
![](https://i.imgur.com/2mpnRSm.png)

Add your keys to your appsettings.json file (ensure that this file is in your .gitignore and changes are not being tracked):
```json
"RecaptchaV3:SiteKey": "...",
"RecaptchaV3:SecretKey": "...",
```

Install a reCAPTCHA package to handle Google reCAPTCHA V3 requests, I recommend GoogleReCaptcha.V3:

![](https://i.imgur.com/aty3K4t.png)

Register the service in the `Startup.cs` file:

```csharp
services.AddHttpClient<ICaptchaValidator, GoogleReCaptchaValidator>();
```

Inject the service into the page:
```csharp
@page
@inject Microsoft.Extensions.Configuration.IConfiguration Configuration
```

Add a hidden input at the bottom of your form:
```html
<input type="hidden" name="captcha" id="captchaV3Input" value="" />
```

Add the javascript handler to the pages script section

***Note we are showing you how to read a config setting drectly within the page this time rather than passing it in a page model property.***
```html
<script src="https://www.google.com/recaptcha/api.js?render=@Configuration["RecaptchaV3:SiteKey"]"></script>
<script>
    grecaptcha.ready(function() {
        grecaptcha.execute('@Configuration["RecaptchaV3:SiteKey"]', { action: 'login' }).then(function (token) {
            $("#captchaV3Input").val(token);
        });
    });
</script>
```

Finally, handle the hidden input in the OnPost handler of the form:
```csharp
using GoogleReCaptcha.V3.Interface;

// ...

// use dependency injection to use the captcha 
// validator provided by the installed package
private readonly ICaptchaValidator _captchaValidator;
// don't forget to add to the constructor

// ...

// accept string captcha as an OnPost method handler argument
public async Task<IActionResult> OnPostAsync(string captcha, ...)
{
    // test to see what the V3 object looks like
    var captchaData = await _captchaValidator.GetCaptchaResultDataAsync(captcha);
    // test with error
    if (true)
    {
        ModelState.AddModelError("captcha", "Captcha validation failed");
        return Page();
    }

// ...
```

Replace the test data with a single call to see if "passed":
```csharp
// replace the "if(true)" condition with the following
if (!await _captchaValidator.IsCaptchaPassedAsync(captcha))
```

<a name="6"></a>
### 6) Two-Factor Authentication
Two factor authentication (2FA) authenticator apps, using a Time-based One-time Password Algorithm (TOTP), are the industry recommended approach for 2FA. 2FA using TOTP is preferred to SMS 2FA (sendting a code via text message).

#### Two Popular TOTP Authenticator Providers:
If you want to test this two factor auth, please download a two-factor authenticator app like:

Microsoft Authenticator for [Android](https://go.microsoft.com/fwlink/?Linkid=825072) and [iOS](https://go.microsoft.com/fwlink/?Linkid=825073) 

or 

Google Authenticator for [Android](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=en) and [iOS](https://itunes.apple.com/us/app/google-authenticator/id388497605?mt=8).

I personally use Google Authenticator.

If you run your app, login and navigate to the Manage page (by clicking your email address in the nav bar) you will see a Two-factor authentication section and an option to Add authenticator app...

![](https://i.imgur.com/w3Alecc.png)

Two-Factor Auth is accessible "out of the box" with Identity in .Net Core 3.1. If you would like you can test this by manually entering the code displayed on the page into your authenticator app. 

![](https://i.imgur.com/Fl6Xafk.png)

#### The next time you sign in, you will be forwarded to an authenticator code page for the second factor of authentication.

![](https://i.imgur.com/YZAfp5J.png)

---

To make this slightly more user friendly, let's enable QRCodes so that our users can simply scan the QRCode rather than manually typing the value...

Download the following and add the qrcode.js file to your `wwwroot\lib` folder.
[qrcodejs Library](https://davidshimjs.github.io/qrcodejs/)

Update the Scripts section of `Areas/Identity/Pages/Account/Manage/EnableAuthenticator.cshtml` to add a reference to the qrcodejs library you added and a call to generate the QR Code. It should look as follows:

```html
@section Scripts {
    @await Html.PartialAsync("_ValidationScriptsPartial")

    <script type="text/javascript" src="~/lib/qrcode.js"></script>
    <script type="text/javascript">
        new QRCode(document.getElementById("qrCode"),
            {
                text: "@Html.Raw(Model.AuthenticatorUri)",
                width: 150,
                height: 150
            });
    </script>
}
```

![](https://i.imgur.com/SCwB0DW.png)

