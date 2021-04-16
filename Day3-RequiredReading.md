# .NET Essentials 2 - Session 3
## Learning Outcomes
* [Preventing Open Redirect Attacks](#1)
* [Preventing Cross Site Scripting (XSS)](#2)
* [Anti Request Forgery (XSRF/CSRF)](#3)
* [Enabling Email for Account Confirmation and Password Recovery](#4)
* [User Secrets for Development](#5)
* [Environment Variables for Production](#6)

---

<a name="1"></a>
## 1) Preventing Open Redirect Attacks
[Microsoft Docs: Prevent open redirect attacks in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/preventing-open-redirects?view=aspnetcore-3.1)

A web app that redirects to a URL that's specified via the request such as the querystring or form data can potentially be tampered with to redirect users to an external, malicious URL. This tampering is called an open redirection attack.

Whenever your application logic redirects to a specified URL, you must verify that the redirection URL hasn't been tampered with. ASP.NET Core has built-in functionality to help protect apps from open redirect (also known as open redirection) attacks.

Imagine you have an account at a VERY popular website called openredirects. You recieve the following email with a link to the REAL opendirects website:

***"You won a Prize!  Your privacy is important to us, please login to your account to claim your prize.  [Click Here to Claim your Prize](https://openredirects.azurewebsites.net/Identity/Account/Login?returnUrl=https://openredirect.azurewebsites.net/Identity/Account/Login)"***

Options for preventing these attacks include:
```csharp
// will error out when the url is not local
return LocalRedirect(returnUrl);

// validates the returnUrl to be local before Redirecting
if (Url.IsLocalUrl(returnUrl))
{
    return Redirect(returnUrl);
}
else
{
    return Page();
}

// will error out the url is not an action of your app
return RedirectToPage(returnUrl);

// DO NOT DO THIS...
return Redirect(returnUrl);
```

___

<a name="2"></a>
## 2) Preventing Cross Site Scripting (XSS)
[Microsoft Docs: Prevent Cross-Site Scripting (XSS) in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/cross-site-scripting?view=aspnetcore-3.1)

Cross-Site Scripting (XSS) is a security vulnerability which enables an attacker to place client side scripts (usually JavaScript) into web pages. When other users load affected pages the attacker's scripts will run, enabling the attacker to steal cookies and session tokens, change the contents of the web page through DOM manipulation or redirect the browser to another page. XSS vulnerabilities generally occur when an application takes user input and outputs it to a page without validating, encoding or escaping it.


Steps to prevent these attacks:
1. Never put untrusted data into your HTML input, unless you follow the rest of the steps below. Untrusted data is any data that may be controlled by an attacker, HTML form inputs, query strings, HTTP headers, even data sourced from a database as an attacker may be able to breach your database even if they cannot breach your application.

2. Before putting untrusted data inside an **HTML element** ensure it's **HTML encoded**. HTML encoding takes characters such as < and changes them into a safe form like &lt;

3. Before putting untrusted data into an **HTML attribute** ensure it's **HTML encoded**. HTML attribute encoding is a superset of HTML encoding and encodes additional characters such as " and '.

4. Before putting untrusted data into **JavaScript** place the data in an HTML element whose contents you retrieve at runtime. If this isn't possible, then ensure the data is **JavaScript encoded**. JavaScript encoding takes dangerous characters for JavaScript and replaces them with their hex, for example < would be encoded as \u003C.

5. Before putting untrusted data into a **URL query string** ensure it's **URL encoded**.

### Create a new Razor Pages Web Application and test the following:

#### HTML Encoding HTML Elements and Attributes
The Razor engine automatically encodes all output sourced from variables, unless you work really hard to prevent it from doing so. It uses HTML attribute encoding rules whenever you use the `@` directive. As HTML attribute encoding is a superset of HTML encoding this means you don't have to concern yourself with whether you should use HTML encoding or HTML attribute encoding. You must ensure that you only use `@` in an HTML context, not when attempting to insert untrusted input directly into JavaScript. Tag helpers will also encode input you use in tag parameters.

```html
@{
    var untrustedInput = "<img src=\"https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRTB8SXoE4bnWxoo0qD7Diq34ydmFaz2Qk9vQ&usqp=CAU\" />";
    var untrustedAttribute = "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRTB8SXoE4bnWxoo0qD7Diq34ydmFaz2Qk9vQ&usqp=CAU";
}
<h4>HTML Encoded by Razor @@</h4>
<h6>Raw HTML</h6>
@Html.Raw(@untrustedInput)
<br />
<br />
<h6>Encoded HTML</h6>
@untrustedInput
<br />
<br />
<br />
<br />
<h4>HTML Encoded by @@Html.Encode</h4>
<h6>Raw  Element with Attribute</h6>
@Html.Raw("<img src=" + @untrustedAttribute + " />")
<br />
<br />
<h6>Encoded Element with Attribute</h6>
@Html.Encode("<img src=" + @untrustedAttribute + " />")
```

#### JavaScript Encoding JavaScript
Do NOT concatenate untrusted input in JavaScript to create DOM elements or use document.write() on dynamically generated content.

Use one of the following approaches to prevent code from being exposed to DOM-based XSS:

* createElement() and assign property values with appropriate methods or properties such as node.textContent= or node.InnerText=`.
* document.CreateTextNode() and append it in the appropriate DOM location.
* element.SetAttribute()
* element[attribute]=

```html
@* untrusted input *@
<div id="injectedData" data-untrustedinput="<script>alert(1)</script>" />

<div id="injectionContainer">
    <h6>Encoded by Document.createTextNode</h6>
    <div id="safeScriptedWrite"></div><br />

    <h6>Encoded by js Node.textContent</h6>
</div>
<script>
    // input retrieval
    var untrustedInput = injectedData.dataset.untrustedinput;

    // target existing container and createTextNode on the element to ensure data is properly encoded.
    var x = document.getElementById("safeScriptedWrite");
    x.appendChild(document.createTextNode(untrustedInput));

    // createElement() to dynamically create document elements
    // This time we're using textContent to ensure the data is properly encoded.
    var y = document.createElement("div");
    y.textContent = untrustedInput;
    document.getElementById("injectionContainer").appendChild(y);

    // not encoded...DO NOT USE Document.write
    document.write(untrustedInput)

</script>
```

#### URL Encoding URL Query Strings
Don't use untrusted input as part of a URL path. Always pass untrusted input as an encoded query string value.

Add the following to your `Index.cshtml`
```html
<form method="post">
    <input type="text" name="badUrl" value='"Quoted Value with spaces and &"' />
    <button type="submit">Click Me</button>
</form>
```

Add the following to your `Index.cshtml.cs`
```csharp
public class IndexModel : PageModel
{
    UrlEncoder _urlEncoder;
    public IndexModel(UrlEncoder urlEncoder)
    {
        _urlEncoder = urlEncoder;
    }

    public void OnGet()
    {

    }
    public void OnPost(string badUrl)
    {
        var encodedValue = _urlEncoder.Encode(badUrl);
    }
}
```
___

<a name="3"></a>
## 3) Anti Request Forgery (XSRF/CSRF)
Cross-site request forgery (also known as XSRF or CSRF) is an attack against web-hosted apps whereby a malicious web app can influence the interaction between a client browser and a web app that trusts that browser. These attacks are possible because web browsers send some types of authentication tokens automatically with every request to a website. This form of exploit is also known as a one-click attack or session riding because the attack takes advantage of the user's previously authenticated session.

Using HTTPS doesn't prevent a CSRF attack. A malicious site can send a secure request just as easily as it can send an insecure request.

Users can guard against CSRF vulnerabilities by taking precautions:

* Sign off of web apps when finished using them.
* Clear browser cookies periodically.

**However, CSRF vulnerabilities are fundamentally a problem with the web app, not the end user.**

### CSRF Example
#### Set up the form host
Add this form to any razor page:
```html
<form method="post">
    <select name="recipientId">
        <option>Select an Org</option>
        <option value="1">Valid Org 1</option>
        <option value="2">Valid Org 2</option>
        <option value="3">Valid Org 3</option>
    </select>

    <label>Donation Amount:</label>
    <input name="amount" id="amount" type="range" min="0" max="100" step="1" value="50" />
    <label id="displayAmount">$ </label>
    <br />
    <button type="submit">Donate</button>
</form>

<script>
    var slider = document.getElementById("amount");
    var output = document.getElementById("displayAmount");
    output.innerHTML = "$"+slider.value;

    slider.oninput = function () {
        output.innerHTML = "$" + this.value;
    }
</script>
```

Add the form's OnPost handler method to the corresponding `cshtml.cs` file:
```csharp
public void OnPost(string recipientId, decimal amount)
{
    // all the site needs is a recipient and an amount...
    //_donationService(recipientId, amount);
}
```

Load the app page in a browser and inspect the form element. The default TagHelpers that are injected into all razor pages will add a hidden input for `__RequestVerificationToken` on any post form element:

![](https://i.imgur.com/WoWREu5.png)

Select an org and an amount and debug the OnPost:

![](https://i.imgur.com/5Xj3vhp.png)

#### Create the attacker
Create the following index.html page separate from the form host application:
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>My CSRF Attack</title>
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
</head>
  </head>
  <body>
    <script>
        $(document).ready(function(){
          $.post("https://localhost:44307/",
            {
                recipientId: "7",
                amount: 1000000.00
            });
        });
        </script>
  </body>
</html>
```

#### Test CSRF
Run your app with a break point on your OnPost method
1. Submit the form from your app. This should be a successful request.
2. Open your index.html. This should not be a succesful request.

Unless disabled, .NET will validate the antiforgery request token.

3. Inspect the element of your hosted form, delete the value of the `__RequestVerificationToken` and try submitting the form. Alternatively you can add this attribute to your form tag and restart your app `asp-antiforgery="false"`.

![](https://i.imgur.com/NOFv5Xn.png)

4. Add the following attribute above your PageModel class `[IgnoreAntiforgeryToken]` and restart your app.
5. Repeat step 3 and you should have a successful request
6. Repeat step 2 and you should have a successful CSFR attack

### Lessons Learned
1. Be aware that web requests without antiforgery protection in place are vulnerable to CSRF attacks.
2. This type of attack can be executed by visiting a malicious site or clicking on a malicious link.
3. Be extremely careful when disabling antiforgery protection.
4. This behaviour is implemented on POST requests and "state" should never be modified from a GET request as they are less secure.

---
<a name="4"></a>
## 4) Enabling Email for Account Confirmation and Password Recovery
.Net Core Identity comes with a default interface called IEmailSender which has a single member for implementation:
```csharp
Task SendEmailAsync(string email, string subject, string htmlMessage);
```
We are going to use a 3rd party mail client called SendGrid to add email functionality to our web application.
1. Create a free SendGrid account here: https://signup.sendgrid.com/
2. Login to your account, add and verify a Single Sender

![](https://i.imgur.com/n1fhN59.png)

3. Add an API Key to connect your .Net app to your SendGrid account (remember to copy it)

![](https://i.imgur.com/jUtJFhk.png)

<a name="5"></a>
4. Open or create a web app with Identity, right click on your Project node in the Visual Studio Solution Explorer and select Manage User Secrets

<img src="https://i.imgur.com/Tjca6di.png" width="300" />

5. Add the following to the secrets.json file
```json
"SENDGRID_KEY": "paste your api key here",
"SENDGRID_EMAIL": "the from email like: pweier@bcit.ca",
"SENDGRID_NAME": "the from name like: Phil Weier"
```

6. Add the SendGrid package to your solution

![](https://i.imgur.com/ceFglQU.png)

7. Create a Services folder with the following classes:
```csharp
// this will allow us to map our settings by name
public class EmailSenderOptions
{
    public string SENDGRID_KEY { get; set; }
    public string SENDGRID_EMAIL { get; set; }
    public string SENDGRID_NAME { get; set; }  
}
```

```csharp
public class EmailSender: IEmailSender
{
    public EmailSender(IOptions<EmailSenderOptions> optionsAccessor)
    {
        Options = optionsAccessor.Value;
    }
    public EmailSenderOptions Options { get; } //set only via Secret Manager
    public Task SendEmailAsync(string email, string subject, string message)
    {
        return Execute(Options.SENDGRID_KEY, email, subject, message);
    }
    public Task Execute(string apiKey, string email, string subject, string message)
    {
        var client = new SendGridClient(apiKey);

        // create the email
        var msg = new SendGridMessage()
        {
            From = new EmailAddress(Options.SENDGRID_EMAIL, Options.SENDGRID_NAME),
            Subject = subject,
            PlainTextContent = message,
            HtmlContent = message
        };
            
        // add recipient(s)
        msg.AddTo(new EmailAddress(email));

        // Disable click tracking.
        // See https://sendgrid.com/docs/User_Guide/Settings/tracking.html
        msg.SetClickTracking(false, false);

        // send the email
        return client.SendEmailAsync(msg);
    }
}
```

9. Add the following to register the classes for DI in the `startup.cs` file
```csharp
services.AddTransient<IEmailSender, EmailSender>();
services.Configure<EmailSenderOptions>(Configuration);
```

10. Remove the manual email confirmation from `RegisterConfirmation.cshtml.cs`:
```csharp
DisplayConfirmAccountLink = false;
```

Test your registration with email confirmation. Please note that some email providers (like hotmail) block un-registered DNS emails ie the email comes from sendgrid.net rather than yourdomain.com that you have whitelisted within your SendGrid account. Try testing with a gmail or your bcit account as I have not experienced challenges with these accounts.

![](https://i.imgur.com/leStqks.png)

___
<a name="6"></a>
## 5) Environment Variables for Production
Because the User Secrets live on your local machine, you will need to add configuration variables to your deployed application.

Publish your app to Azure (or any other hosting service). 

*Note, if you are using the Azure for Students Starter account, you are limited to one free Azure sql db per region. If you already have a db hosted for free you may need to host your services to a different region.*

<img src="https://i.imgur.com/8vhUsl5.png" width="500" />

You can add the settings to the deployed site from Visual Studio:

![](https://i.imgur.com/8ZEhF4t.png)

<img src="https://i.imgur.com/3mwXU5v.png" width="500" />

---

Or you can manage these settings online from your Azure Portal:
![](https://i.imgur.com/SBXZ7Z7.png)

---

![](https://i.imgur.com/VIiPO9w.png)

---

![](https://i.imgur.com/5R6QG6m.png)

---
