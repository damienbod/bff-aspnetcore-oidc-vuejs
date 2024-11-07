# bff-aspnetcore-vuejs

[![.NET and npm build](https://github.com/damienbod/bff-aspnetcore-oidc-vuejs/actions/workflows/dotnet.yml/badge.svg)](https://github.com/damienbod/bff-aspnetcore-oidc-vuejs/actions/workflows/dotnet.yml) [![License](https://img.shields.io/badge/license-Apache%20License%202.0-blue.svg)](https://github.com/damienbod/bff-aspnetcore-oidc-vuejs/blob/main/LICENSE)

## Setup Server 

The ASP.NET Core project is setup to run in development and production. In production, it uses the Vue.js production build deployed to the wwwroot. In development, it uses MS YARP reverse proxy to forward requests.

> [!IMPORTANT]  
> In production, the Vue.js project is built into the **wwwroot** of the .NET project.

![BFF production](https://github.com/damienbod/bff-aspnetcore-oidc-vuejs/blob/main/images/vue-aspnetcore-bff_01.png)

Configure the YARP reverse proxy to match the Vue.js URL. This is only required in development. I always use HTTPS in development and the port needs to match the Vue.js developement env (vite.config.js).

```json
 "UiDevServerUrl": "https://localhost:4201",
  "ReverseProxy": {
    "Routes": {
      "route1": {
        "ClusterId": "cluster1",
        "Match": {
          "Path": "{**catch-all}"
        }
      }
    },
    "Clusters": {
      "cluster1": {
        "HttpClient": {
          "SslProtocols": [
            "Tls12"
          ]
        },
        "Destinations": {
          "cluster1/destination1": {
            "Address": "https://localhost:4201/"
          }
        }
      }
    }
  }
```


## Setup Vue.js Vite project

Add the certificates to the nx project for example in the **/certs** folder

Update the vite.config.ts file:

```typescipt
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import fs from 'fs';

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue()],
  server: {
    https: {
      key: fs.readFileSync('./certs/dev_localhost.key'),
      cert: fs.readFileSync('./certs/dev_localhost.pem'),
	},
    port: 4202,
    strictPort: true, // exit if port is in use
    hmr: {
      clientPort: 4202, // point vite websocket connection to vite directly, circumventing .net proxy
    },
  },
  optimizeDeps: {
    force: true,
  },
  build: {
    outDir: "../server/wwwroot",
    emptyOutDir: true
  },
})
```


> [!NOTE]  
> The ASP.NET Core project setup uses port 4202, this needs to match the YARP reverse proxy settings for development.


## Setup development

The development environment is setup to use the default tools for each of the tech stacks. Vue.js is used like recommended. I use Visual Studio code. A YARP reverse proxy is used to integrate the Vue.js development into the backend application.

![BFF development](https://github.com/damienbod/bff-aspnetcore-oidc-vuejs/blob/main/images/vue-aspnetcore-bff-yarp-dev_01.png)

> [!NOTE]  
> Always run in HTTPS, both in development and production

```
npm start
```

The OpenID Connect client is setup using the default ASP.NET Core OpenID connect handler.

```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    builder.Configuration.GetSection("OpenIDConnectSettings").Bind(options);
    options.Authority = builder.Configuration["OpenIDConnectSettings:Authority"];
    options.ClientId = builder.Configuration["OpenIDConnectSettings:ClientId"];
    options.ClientSecret = builder.Configuration["OpenIDConnectSettings:ClientSecret"];

    options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.ResponseType = OpenIdConnectResponseType.Code;

    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        NameClaimType = "name"
    };
});

services.AddControllersWithViews(options =>
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute()));

services.AddRazorPages().AddMvcOptions(options =>
{
    var policy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    options.Filters.Add(new AuthorizeFilter(policy));
});
```

Add the Azure App registration settings to the **appsettings.Development.json** and the **ClientSecret** to the user secrets.

```json
"OpenIDConnectSettings": {
    "Authority": "https://localhost:44318",
    "ClientId": "oidc-pkce-confidential",
    "ClientSecret": "oidc-pkce-confidential_secret"
},
```

## Debugging

Start the Vue.js project from the **ui** folder

```
npm start
```

Start the ASP.NET Core project from the **server** folder

```
dotnet run
```

Or just open Visual Studio and run the solution.

## Github actions build

Github actions is used for the DevOps. The build pipeline builds both the .NET project and the Vue.js project using npm. The two projects are built in the same step because the UI project is built into the wwwroot of the server project.

```yaml

name: .NET and npm build

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x

      - name: Restore dependencies
        run: dotnet restore

      - name: npm setup
        working-directory: ui
        run: npm install

      - name: ui-build
        working-directory: ui
        run: npm run build

      - name: Build
        run: dotnet build --no-restore
      - name: Test
        run: dotnet test --no-build --verbosity normal
```

## Credits and used libraries

- NetEscapades.AspNetCore.SecurityHeaders
- Yarp.ReverseProxy
- Microsoft.Identity.Web
- ASP.NET Core
- Vue.js
- Vite

## Links

https://vuejs.org/

https://vitejs.dev/

https://github.com/vuejs/create-vue

https://learn.microsoft.com/en-us/aspnet/core/introduction-to-aspnet-core

https://github.com/AzureAD/microsoft-identity-web

https://www.koderhq.com/tutorial/vue/vite/

https://github.com/isolutionsag/aspnet-react-bff-proxy-example

https://github.com/damienbod/bff-aspnetcore-angular

https://github.com/damienbod/bff-auth0-aspnetcore-angular

https://github.com/damienbod/bff-openiddict-aspnetcore-angular

https://github.com/damienbod/bff-azureadb2c-aspnetcore-angular

https://github.com/damienbod/bff-MicrosoftEntraExternalID-aspnetcore-angular
