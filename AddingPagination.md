# Server-Side Pagination
* [How It Works](#1)
* [Add Pagination Query to Back-End](#2)
    * [Handle Required Parameters on the Server](#3)
    * [Handle the Parameters in the Service](#4)
    * [Use Skip() and Take()](#5)
    * [Test the Query](#6)
* [Add Pagination Controls to Front-End](#7)
* [Dynamic Pagination Controls](#8)
* [Prevent Pagination Navigation Errors](#9)
* [Persistence of Existing GET Parameters](#10)

---

Today we will learn how to incorporate pagination to the server-side query of a .NET collection. Whether you have are rendering the resulting collection on the server or in the client following an API call, the implementation of the query code can be the same.

---

<a name="1"></a>
## How It Works
We will require 2 new parameters in order to effectively manage a paginated collection:
1. Page Index - the page index that will be displayed
2. Items Per Page - the maximum number of items that can be displayed on one page.

Once we have the two parameters, we can use the [Skip()](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable.skip?view=netcore-3.1) and [Take()](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable.take?view=netcore-3.1) methods to return the appropriate items in a collection.

If we consider that we could want a maximum of 5 items per page and we have a collection of n items where n is any number greater than 5. The number of item pages would be the rounded up result or n/5 or:
```csharp
int.Parse(Math.Ceiling((decimal)n / 5));
```

The items on page 3 (index 2 if we zero index the pages) would look somtething like this:
```csharp
var itemsPerPage = 5;
var pageIndex = 2;
var pagedItems = _db.TableName.Skip(itemsPerPage * pageIndex).Take(itemsPerPage);
```

---

<a name="2"></a>
## Add Pagination Query to Back-End
Now we will add the pagination logic to the product list that is on the home page of our eShopOnWeb application.

<a name="3"></a>
### Handle Required Parameters on the Server
The `Index.cshtml.cs` OnGet method is what handles our client requests to render the product collection on the page. Update the OnGet method to look like this:
```csharp
const int ITEMS_PER_PAGE = 3;

public async Task OnGet(ProductIndexVM productIndex, int? pageIndex)
{
    ProductIndex = _productVMService.GetProductsVM(pageIndex ?? 0, ITEMS_PER_PAGE, productIndex.TypesFilterApplied);      
}
```

We make the pageIndex optional and in the case that it is null we explicitly pass it as 0, meaning page 1. We are also setting the maximum number of items per page as a constant value, however you could add a select input to your page to allow this value to be variable.

<a name="4"></a>
### Handle the Parameters in the Service
Because we have added two arguments to the GetProductsVM method call, we will need to update both the interface definition and service implementation of the method:

```csharp
GetProductsVM(int pageIndex, int itemsPerPage, int? typeId)
```

<a name="5"></a>
### Use Skip() and Take()
Once we have applied any filter logic and determined the collection that "could" be queried and returned to the client, use the Skip() and Take() methods to skip any items from "previous" pages and take the number of items for the current page:
```csharp
IQueryable<Product> products = _productRepo.GetAll();
// apply filter logic
if (typeId != null)
    products = products.Where(p=>p.ProductTypeId == typeId);

// apply pagination
products = products.Skip(pageIndex * itemsPerPage).Take(itemsPerPage);
```

<a name="6"></a>
### Test the Query
If you run your application now, the home page will only render the first three products. If you attach a SQL Profiler, you will also see that we used IQueryable and only one query was run to get the single page of products:
```sql
exec sp_executesql 
N'SELECT [p].[Id], [p].[Name], [p].[Price], [p].[PictureUri], 1 AS [Quantity]
FROM [Products] AS [p]
ORDER BY (SELECT 1)
OFFSET @__p_0 ROWS FETCH NEXT @__p_1 ROWS ONLY',N'@__p_0 int,@__p_1 int',@__p_0=0,@__p_1=3

-- this simplifies to:
SELECT * FROM Products
ORDER BY (SELECT 1)
OFFSET 0 ROWS FETCH NEXT 3 ROWS ONLY;
```

A default ORDER BY clause is added because it is necessary so that we can access ROW numbers.

#### Apply the Mugs Filter
Apply the mugs filter and examine the sql that is executed by using SQL Profiler.

![](https://i.imgur.com/0tJ6wdB.png)

---

<a name="7"></a>
## Add Pagination Controls to Front-End
Now that we have effectively implemented the query logic, now we can add controls to the application front-end to navigate pages.

Let's create a partial that will have our pagination controls. The benefit of using a partial (or reusable JS Component for API consumers) is that we can add pagination to any collection in our application and simply render the reusable controls on multiple pages.

Create the following `Pages/Shared/_pagination.cshtml`:
```html
<div class="row">
    <div class="pagination-nav-item col-md-4">
        <a id="Previous" asp-route-pageIndex=0>Previous</a>
    </div>
    <div class="pagination-nav-item col-md-4">
        <span>
            Showing [x] of [n] products - Page [y]
        </span>
    </div>
    <div class="pagination-nav-item col-md-4">
        <a id="Next" asp-route-pageIndex=1>Next</a>
    </div>
</div>
```

Add the following to `site.css`:
```css
.pagination-nav-item {
    text-align: center;
    width: 33%;
    display: inline-block;
}
```

Add the following above and below the app-product-items in `Index.cshtml`:
```html
<partial name="_pagination" />
```

---

<a name="8"></a>
## Dynamic Pagination Controls
In order to make the pagination controls dynamically managed by the user actions (clicking previous vs next) we will need to add a model so that we can easily access the current page and the total number of pages.

Add a PaginationInfoVM view model:
```csharp
public class PaginationInfoVM
{
    public int TotalItems { get; set; }
    public int ItemsPerPage { get; set; }
    public int PageIndex { get; set; }
    public int TotalPages { get; set; }
    public string Previous { get; set; }
    public string Next { get; set; }
}
```

Add the PaginationInfoVM to the view model that is sent to the home page (in my case, this is my ProductIndexVM class):
```csharp
public PaginationInfoVM PaginationInfo { get; set; }
```

Populate the PaginationInfoVM property in the

```csharp
public ProductIndexVM GetProductsVM(int pageIndex, int itemsPerPage, int? typeId)
{
    IQueryable<Product> products = _productRepo.GetAll();
    if (typeId != null)
        products = products.Where(p=>p.ProductTypeId == typeId);
    
    int totalItems = products.Count();
    
    products = products.Skip(pageIndex * itemsPerPage).Take(itemsPerPage);
    
    var vm = new ProductIndexVM()
    {
        Products = products.Select(p => new ProductVM
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            PictureUri = p.PictureUri,
            Quantity = 1
        }).ToList(),
        Types = GetTypes().ToList(),
        PaginationInfo = new PaginationInfoVM()
        {
            PageIndex = pageIndex,
            ItemsPerPage = products.Count(),
            TotalItems = totalItems,
            TotalPages = int.Parse(Math.Ceiling(((decimal)totalItems / itemsPerPage)).ToString())
        }
    };
    return vm;
}
```

Update the two partial tags in `Index.cshtml`:
```html
<partial name="_pagination" for="@Model.ProductIndex.PaginationInfo" />
```

Assign the model in the `_pagination.cshtml` partial and test the display of the properties:
```html
@model ViewModels.PaginationInfoVM

<div class="row">
    <div class="pagination-nav-item col-md-4">
        <a id="Previous" asp-route-pageIndex=@(Model.PageIndex-1)>Previous</a>
    </div>
    <div class="pagination-nav-item col-md-4">
        <span>
            Showing @Model.ItemsPerPage of @Model.TotalItems products - Page @(Model.PageIndex + 1) - @Model.TotalPages
        </span>
    </div>
    <div class="pagination-nav-item col-md-4">
        <a id="Next" asp-route-pageIndex=@(Model.PageIndex+1)>Next</a>
    </div>
</div>
```

![](https://i.imgur.com/l8M6bxG.jpg)

### Test Navigation With Previous from First Page

![](https://i.imgur.com/3z2Olfv.png)

### Test Navigation With Next from Last Page

![](https://i.imgur.com/ZE70IRR.png)

---

<a name="9"></a>
## Prevent Pagination Navigation Errors
To prevent the behaviour displayed above, we will add a css flag to disable the navigation links.

Use the Previous and Next properties of the VM to store a css class:
```csharp
vm.PaginationInfo.Next = (vm.PaginationInfo.PageIndex == vm.PaginationInfo.TotalPages - 1) ? "is-disabled" : "";
vm.PaginationInfo.Previous = (vm.PaginationInfo.PageIndex == 0) ? "is-disabled" : "";
```

Add the css for the is-disabled class:
```css
.is-disabled {
    opacity: .5;
    pointer-events: none;
}
```

Add the classes to the navigation anchor tags in the partial:
```html
class="@Model.Previous"

...

class="@Model.Next"
```

![](https://i.imgur.com/p61f81h.jpg)

---

<a name="10"></a>
## Persistence of Existing GET Parameters
You may notice that the partial's use of asp-route-pageIndex erases any existing filter parameters that were present from the parent view. We can avoid this by managing the method in which we add and remove the pageIndex parameter within the partial.

Replace the asp-route-pageIndex references with `asp-all-route-data` and set the values to "previous" and "next" accordingly.
```html
<div class="pagination-nav-item col-md-4"><a class="@Model.Previous" asp-all-route-data="previous">Previous</a></div>
...   
<div class="pagination-nav-item col-md-4"><a class="@Model.Next" asp-all-route-data="next">Next</a></div>
```

Add the following Razor code to the top of the `_pagination.cshtml` partial which adds and removes the pageIndex values as independant Dictionary items:
```html
@{
    var previous = Context.Request.Query.ToDictionary(x=>x.Key, x=> x.Value.ToString());
    var next = Context.Request.Query.ToDictionary(x => x.Key, x => x.Value.ToString());
    
    if (previous.ContainsKey("pageIndex"))
        previous.Remove("pageIndex");
    previous.Add("pageIndex", (Model.PageIndex - 1).ToString());

    if (next.ContainsKey("pageIndex"))
        next.Remove("pageIndex");
    next.Add("pageIndex", (Model.PageIndex + 1).ToString());
 }
```



