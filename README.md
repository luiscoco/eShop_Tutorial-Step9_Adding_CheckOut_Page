# Building 'eShop' from Zero to Hero: Adding CheckOut Page

This sample scope of work is focus on adding the **CheckOut** web page in the **eShop** solution

![image](https://github.com/user-attachments/assets/96676f71-4727-4a65-a5a6-8432e0be5c24)

## 1. We Download the Solution from Github repo

The starting point for this sample is based on the following github repo:

https://github.com/luiscoco/eShop_Tutorial-Step8_Adding_Basket_API

## 2. We Rename the Solution

![image](https://github.com/user-attachments/assets/4b53ce53-9a5a-4c2e-a15c-b62c77e85596)

## 3. We Add a new razor component "Checkout" in the WebApp project

We create a new folder inside the Component->Pages folder

![image](https://github.com/user-attachments/assets/2735be5a-305b-4119-8222-9359257dcd83)

We also add a new razor component inside the Checout folder. This is the ChecOut web page code:

### 3.1. Checkout.razor source code

```razor
@page "/checkout"
@using System.Globalization
@using System.ComponentModel.DataAnnotations
@using Microsoft.AspNetCore.Authorization
@inject BasketState Basket
@inject NavigationManager Nav
@attribute [Authorize]

<PageTitle>Checkout | AdventureWorks</PageTitle>
<SectionContent SectionName="page-header-title">Checkout</SectionContent>

<div class='checkout'>
    <EditForm EditContext="@editContext" FormName="checkout" OnSubmit="@HandleSubmitAsync" Enhance>
        <DataAnnotationsValidator />
        <div class="form">
            <div class="form-section">
                <h2>Shipping address</h2>
                <label>
                    Address
                    <InputText @bind-Value="@Info.Street" />
                    <ValidationMessage For="@(() => Info.Street)" />
                </label>
                <div class="form-group">
                    <div class="form-group-item">
                        <label>
                            City
                            <InputText @bind-Value="@Info.City" />
                            <ValidationMessage For="@(() => Info.City)" />
                        </label>
                    </div>
                    <div class="form-group-item">
                        <label>
                            State
                            <InputText @bind-Value="@Info.State" />
                            <ValidationMessage For="@(() => Info.State)" />
                        </label>
                    </div>
                    <div class="form-group-item">
                        <label>
                            Zip code
                            <InputText @bind-Value="@Info.ZipCode" />
                            <ValidationMessage For="@(() => Info.ZipCode)" />
                        </label>
                    </div>
                </div>
                <div>
                    <label>
                        Country
                        <InputText @bind-Value="@Info.Country" />
                        <ValidationMessage For="@(() => Info.Country)" />
                    </label>
                </div>
            </div>
            <div class="form-section">
                <div class="form-buttons">
                    <a href="cart" class="button button-secondary"><img role="presentation" src="icons/arrow-left.svg" />Back to the shopping bag</a>
                    <button class="button button-primary" type="submit">Place order</button>
                </div>
            </div>
        </div>
        <ValidationSummary />
    </EditForm>
</div>

@code {
    private EditContext editContext = default!;
    private ValidationMessageStore extraMessages = default!;

    [SupplyParameterFromForm]
    public BasketCheckoutInfo Info { get; set; } = default!;

    [CascadingParameter]
    public HttpContext HttpContext { get; set; } = default!;

    protected override void OnInitialized()
    {
        if (Info is null)
        {
            PopulateFormWithDefaultInfo();
        }

        editContext = new EditContext(Info!);
        extraMessages = new ValidationMessageStore(editContext);
    }

    private void PopulateFormWithDefaultInfo()
    {
        Info = new BasketCheckoutInfo
        {
            Street = ReadClaim("address_street"),
            City = ReadClaim("address_city"),
            State = ReadClaim("address_state"),
            Country = ReadClaim("address_country"),
            ZipCode = ReadClaim("address_zip_code"),
            RequestId = Guid.NewGuid()
        };

        string? ReadClaim(string type)
            => HttpContext.User.Claims.FirstOrDefault(x => x.Type == type)?.Value;
    }

    private async Task HandleSubmitAsync()
    {
        await PerformCustomValidationAsync();

        if (editContext.Validate())
        {
            await HandleValidSubmitAsync();
        }
    }

    private async Task HandleValidSubmitAsync()
    {
        Info.CardTypeId = 1;
        await Basket.CheckoutAsync(Info);
        Nav.NavigateTo("user/orders");
    }

    private async Task PerformCustomValidationAsync()
    {
        extraMessages.Clear();

        if ((await Basket.GetBasketItemsAsync()).Count == 0)
        {
            extraMessages.Add(new FieldIdentifier(Info, ""), "Your cart is empty");
        }
    }

    private static DateTime? ParseExpiryDate(string? mmyy)
        => DateTime.TryParseExact($"01/{mmyy}", "dd/MM/yy", null, DateTimeStyles.None, out var result) ? result.ToUniversalTime() : null;
}
```

### 3.2. Key Features in the above code

**Data Binding**: Dynamically binds user input to the Info model

**Validation**: Combines Blazor's built-in validation with custom logic for a seamless user experience

**Authorization**: Ensures only authenticated users can access the checkout page

**Navigation**: Automatically redirects to the orders page upon successful submission


### 3.3. Page Configuration

**@page "/checkout"**: Specifies the route URL (/checkout) for this component

**@attribute [Authorize]**: Restricts access to authenticated users only

**@inject Statements**: Injects services for use within the component:

**BasketState Basket**: Manages the user's shopping basket

**NavigationManager Nav**: Handles navigation between pages

### 3.4. Page Content

**Page Header**:

```<PageTitle>```: Sets the browser tab title to "Checkout | AdventureWorks"

```<SectionContent>```: Displays "Checkout" as the header for this section

**Form**:

```<EditForm>```: Handles user input and validation for the checkout form

**EditContext**: Tracks the form's state

**OnSubmit="@HandleSubmitAsync"**: Specifies the event handler for form submission

**Form Fields**:

**Includes fields for the shipping address**: Address, City, State, Zip Code, and Country

**Data Binding**: Uses @bind-Value to bind form fields to the Info object (e.g., Info.Street, Info.City)

**Validation**: Uses ```<ValidationMessage>``` and ```<DataAnnotationsValidator>``` to validate user input and display error messages

**Form Buttons**: Back to shopping bag, Links back to the cart page

**Place order**: Submits the form

## 6. We run the application and we verify the results

This sample scope of work is focus on adding the **CheckOut** web page in the **eShop** solution

To visit the ChecOut web page, first we have to add an item into the shopping basket and after press the basket icon to navigate to it

![image](https://github.com/user-attachments/assets/e0e2d99e-8b78-486d-b559-a05dc884e77d)

Inside the basket we see all the product we already have bought and we can press the ChecOut button

![image](https://github.com/user-attachments/assets/117f7a19-95cb-486d-9322-69588a339b2d)

This is the CheckOut web page

![image](https://github.com/user-attachments/assets/96676f71-4727-4a65-a5a6-8432e0be5c24)

