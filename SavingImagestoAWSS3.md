# Saving Images to AWS S3
* [How It Works](#1)
* [Create S3 Bucket](#5)
* [Create The App With Database](#2)
* [Add Model to Map AWS Config Settings](#3)
* [Create Service Class To Connect to AWS](#4)


Today we will learn how to add upload images to an AWS S3 bucket, save the remote uri to our database, and render the image from the db uri.

[AWS S3](https://aws.amazon.com/s3/)

<a name="1"></a>
## How It Works
You must have an AWS account to use this service. I recommend creating a free tier account with your 'non-bcit' email address at this time (I have had challenges with the educate accounts due to limited access to IAM).

[AWS Free Tier Account](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc)

1. We will use our AWS account to create an IAM user with credentials and S3 Permissions.
2. We will create an S3 Bucket with a permission policy to allow for public viewing.
3. We will use our IAM User credentials in a .NET Core 3.1 Razor Pages app to upload a file to the S3 Bucket.
4. We will store the uri of the S3 image in our database and use it to render the image in our web page.

<a name="5"></a>
## Create S3 Bucket
Login to your AWS account as Root User.

#### Create an IAM User with credentials for your application

![](https://i.imgur.com/y8Xuvkf.png)

---

#### Grant them programmatic access

![](https://i.imgur.com/yfehBSu.png)

---

#### Give the user AmazonS3FullAccess

![](https://i.imgur.com/XFmBLan.png)

---

#### Create the User (be sure to copy the Key and Secret Key to appsettings)

![](https://i.imgur.com/NbMGWn3.png)

___

#### Create A New S3 Bucket

![](https://i.imgur.com/TBMtxhK.png)

![](https://i.imgur.com/SyOG6DA.png)

---

#### Add the Bucket Name and Region to appsettings
Note that region syntax is as follows:
```json
"BucketName": "a00948735demobucket2",
"BucketRegion": "us-west-2",
```

---

#### Make the Bucket Accessible

![](https://i.imgur.com/LkpDm34.png)

---

#### Add a Policy to Make the Objects Accessible
After you create the bucket you can add a policy to make the objects visible.

![](https://i.imgur.com/jvpJyVU.png)

Open the bucket and add the following Bucket Policy under the Permissions tab:
```json
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "AllowPublicRead",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::[YourBucketName]/*"
        }
    ]
}
```

<a name="2"></a>
## Create The App With Database
Create a simple razor pages web application with a database table to store the image uri's.

#### Models/DbImage.cs 
```csharp
public class DbImage
{
    [Key]
    public int Id { get; private set; }
    [Required]
    public string Uri { get; private set; }
    
    public DbImage(string uri){Uri = uri;}
}
```

#### Data/ApplicationDbContext.cs
```csharp
public DbSet<DbImage> DbImages { get; set; }
```

#### Migration
Add your connection string then add and run migrations:
```bash
Add-Migration CreateImageTable
Update-Database
```

#### index.cshtml.cs
```csharp
private readonly ApplicationDbContext _db;

public IndexModel(ApplicationDbContext db)
{
    _db = db;
}
public List<DbImage> Images { get; set; }

[BindProperty]
public IFormFile FileUpload { get; set; }

public void OnGet()
{
    Images = _db.DbImages.ToList();
}

public async Task<IActionResult> OnPostUploadAsync()
{
    if (ModelState.IsValid)
    {
                
        return RedirectToPage();
    }

    return Page();
}
```
#### index.cshtml
```html
<h1>AWS S3 Image Upload</h1>
<hr />
<form enctype="multipart/form-data" method="post">
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>
    <dl>
        <dt>
            <label asp-for="FileUpload">Upload an Image</label>
        </dt>
        <dd>
            <input asp-for="FileUpload" type="file">
            <span></span>
        </dd>
    </dl>
    <button asp-page-handler="Upload" type="submit">Submit</button>
</form>
<br />
@if (Model.Images.Any())
{
    foreach (var image in Model.Images)
    {
        <img src="@image.Uri" width="300" />
    }
}
```

<a name="3"></a>
## Add Model to Map AWS Config Settings

#### Models/ServiceConfiguration.cs
```csharp
public class ServiceConfiguration
{
    public AWSS3Configuration AWSS3 { get; set; }
}
public class AWSS3Configuration
{
    public string BucketName { get; set; }
    public string BucketRegion { get; set; }
    public string Key { get; set; }
    public string SecretKey { get; set; }   
}
```

#### appsettings.json or User Secrets
```json
"ServiceConfiguration": {
    "AWSS3": {
      "BucketName": "",
      "BucketRegion": "",
      "Key": "",
      "SecretKey": ""
    }
  }
```

#### Register in Startup.cs for Dependancy Injection
```csharp
services.Configure<ServiceConfiguration>(Configuration.GetSection("ServiceConfiguration"));
```

<a name="4"></a>
## Create Service Class To Connect to AWS
Use NuGet Package Manager to install the latest version of `AWSSDK.S3`

#### Services/AWSS3FileService.cs

```csharp
public interface IAWSS3FileService
{
    Task<string> UploadFile(IFormFile fileUpload);
}

public class AWSS3FileService : IAWSS3FileService
{
    private readonly ServiceConfiguration _settings;
    public AWSS3FileService(IOptions<ServiceConfiguration> settings)
    {
        this._settings = settings.Value;
    }
    public async Task<string> UploadFile(IFormFile fileUpload)
    {
        try
        {
            // Do not store your IAM credentials in code.
            // Create an AWS client
            var bucketRegion = Amazon.RegionEndpoint.GetBySystemName(_settings.AWSS3.BucketRegion);
            var s3 = new AmazonS3Client(_settings.AWSS3.Key, _settings.AWSS3.SecretKey, bucketRegion);

            // Create the request
            string key = $"{DateTime.Now.Ticks}{fileUpload.FileName}";
            PutObjectRequest request = new PutObjectRequest()
            {
                InputStream = fileUpload.OpenReadStream(),
                BucketName = _settings.AWSS3.BucketName,
                Key = key
            };
            
            // Use the client to send the request
            PutObjectResponse response = await s3.PutObjectAsync(request);
            if (response.HttpStatusCode == System.Net.HttpStatusCode.OK)
            // The response does not have the uri
            // We can make the uri without making a second request to aws
                return $"https://{_settings.AWSS3.BucketName}.s3.{_settings.AWSS3.BucketRegion}.amazonaws.com/{key}";
            else
                throw new Exception();
        }
        catch (Exception ex)
        {
            throw ex;
        }
    }
}

```

#### Call the Service Class from Index.cshtml.cs OnPostUpload
```csharp

if (ModelState.IsValid)
{
    if (FileUpload != null)
    {
        try
        {
            string imageUri = await _awsFileService.UploadFile(FileUpload);
            _db.DbImages.Add(new DbImage(imageUri));
            _db.SaveChanges();
            return RedirectToPage();
        }
        catch (Exception e) {
            ModelState.AddModelError("ImageUpload", e.Message);
        } 
    }
}
```


