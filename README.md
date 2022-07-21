# blazor-wasm-auth

This repo shows how to implement a Blazor WASM hosted app with a custom AuthorizationMessageHandler class for accessing the Graph API.

## Disclaimer

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

## How it works

### Default backend API requests

In the `/web/Program.cs` file, you set up the default HTTP client with the backend address.

```csharp
builder.Services.AddHttpClient("web.ServerAPI", client => client.BaseAddress = new Uri(builder.HostEnvironment.BaseAddress))
    .AddHttpMessageHandler<BaseAddressAuthorizationMessageHandler>();

builder.Services.AddScoped(sp => sp.GetRequiredService<IHttpClientFactory>().CreateClient("web.ServerAPI"));

builder.Services.AddMsalAuthentication(options =>
{
    builder.Configuration.Bind("AzureAd", options.ProviderOptions.Authentication);
    options.ProviderOptions.DefaultAccessTokenScopes.Add("api://a71fc8d7-4a88-4de3-816c-e19fd131339e/API.Access");
});
```

In the `/web/Client/Pages/FetchData.razor` page, you request the HTTP client & issue the GET call.

```csharp
@inject HttpClient Http

...

@code {
    protected override async Task OnInitializedAsync()
    {
        forecasts = await Http.GetFromJsonAsync<WeatherForecast[]>("WeatherForecast");
    }
```

### Graph API calls (not default backend API requests)

For API calls that are not the default backend, you should create a custom AuthorizationMessageHandler class for each different API.

In the `/web/GraphApiAuthorizationMessageHandler.cs` file, we add the backend URL & the required scopes.

```csharp
 public class GraphApiAuthorizationMessageHandler : AuthorizationMessageHandler
  {
      public GraphApiAuthorizationMessageHandler(IAccessTokenProvider provider, NavigationManager navigationManager) : base(provider, navigationManager)
      {
          ConfigureHandler(
              authorizedUrls: new[] { "https://graph.microsoft.com/" },
              scopes: new[] { 
                  "https://graph.microsoft.com/User.Read",  
                  "https://graph.microsoft.com/Group.Read.All",  
                  "https://graph.microsoft.com/email", 
                  "https://graph.microsoft.com/openid", 
                  "https://graph.microsoft.com/profile" });
      }
  }
```

In the `/web/Program.cs` file, we need to register this handler with a new HTTP client (note the magic string **GraphAPI** that refers to this specific HTTP handler).

```csharp
builder.Services.AddScoped<GraphApiAuthorizationMessageHandler>();
builder.Services.AddHttpClient("GraphAPI",
    client => client.BaseAddress = new Uri("https://graph.microsoft.com"))
    .AddHttpMessageHandler<GraphApiAuthorizationMessageHandler>();
```

In the `/web/Client/Pages/UserProfile.razor` file, you will need to request an access token the specific backend, then request the specific HTTP client you need.

```csharp
@inject IAccessTokenProvider TokenProvider
@inject IHttpClientFactory ClientFactory

...

protected override async Task OnInitializedAsync()
{
    var tokenResult = await TokenProvider.RequestAccessToken(
        new AccessTokenRequestOptions
            {
                Scopes = new[] { "https://graph.microsoft.com/User.Read" }
            }
    );        

    if (tokenResult.TryGetToken(out var token))
    {
        var client = ClientFactory.CreateClient("GraphAPI");

        userProfileModel = await client.GetFromJsonAsync<UserProfileModel>("v1.0/me");
    }
```

## Links

- https://docs.microsoft.com/en-us/aspnet/core/blazor/security/webassembly/hosted-with-azure-active-directory?view=aspnetcore-6.0
- https://docs.microsoft.com/en-us/aspnet/core/blazor/security/webassembly/additional-scenarios?view=aspnetcore-6.0
- https://docs.microsoft.com/en-us/aspnet/core/blazor/security/webassembly/graph-api?view=aspnetcore-6.0
- https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals
- https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-how-applications-are-added
- https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/consent-and-permissions-overview
