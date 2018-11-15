# Exercise #2

## Goal

Explore Externalized Configuration by working with Spring Cloud Config Server

## Expected Results

Create an instance of Spring Cloud Services Configuration Server and bind our API application to that instance of the Configuration Server.

## Introduction

In this exercise we explore how Configuration Server pulls configuration from a backend repository.  Also observe how we utilize Steeltoe to connect to Configuration Server and manipulate how we retrieve configuration.

1. Create a directory for our new API with the following command:  `mkdir bootcamp-webapi`

2. Navigate to the newly created directory using the following command: `cd bootcamp-webapi`

3. Use the Dotnet CLI to scaffold a basic Web API application with the following command: `dotnet new webapi`.  This will create a new application with name bootcamp-webapi.

4. Navigate to the project file and edit it to add the following nuget packages:

    ```xml
    <PackageReference Include="NSwag.AspNetCore" Version="11.19.2" />
    <PackageReference Include="Pivotal.Extensions.Configuration.ConfigServerCore" Version="2.1.1" />
    ```

5. In the Program.cs class add the following using statement and edit the CreateWebHostBuilder method in the following way
    ```c#
    using Pivotal.Extensions.Configuration.ConfigServer;
    ```
    ```c#
    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseCloudFoundryHosting()
            .ConfigureAppConfiguration(b => b.AddConfigServer(new LoggerFactory().AddConsole(LogLevel.Trace)))
            .UseStartup<Startup>();
    ```

    **Take note of the UseCloudFoundryHosting and AddConfigServer methods which are extensions that allow us to listen on a configured port and adds spring cloud config server as a configuration source respectively.**

6. Navigate to the Startup class and set the following using statements:

    ```c#
    using Pivotal.Extensions.Configuration.ConfigServer;
    using NJsonSchema;
    using NSwag.AspNetCore;
    ```

7. In the ConfigureServices method use an extension method to add swagger and the Config Server provider to the DI Container with the following line of code.

    ```c#
    services.AddConfiguration(Configuration);
    services.AddSwagger();
    ```

8. In the Configure method add swagger to the middleware pipeline by adding the following code snippet just before the `app.UseMvc();` line 

    ```c#
    app.UseSwaggerUi3WithApiExplorer(settings =>
        {
            settings.GeneratorSettings.DefaultPropertyNameHandling = 
                PropertyNameHandling.CamelCase;
            settings.PostProcess = document => 
            {
                document.Info.Version = "v1";
                document.Info.Title = "Bootcamp API";
                document.Info.Description = "A simple ASP.NET Core web API";
                document.Schemes.Clear();
                document.Schemes.Add(NSwag.SwaggerSchema.Https);
            };
            settings.SwaggerUiRoute = "";
        });
    ```

9. In the controllers folder create a new class and name it ProductsController.cs
Paste the following contents into the file:

    ```c#
    using System;
    using System.Collections.Generic;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Extensions.Configuration;

    namespace bootcamp_webapi.Controllers
    {   
        [Route("api/[controller]")]
        [ApiController]
        public class ProductsController : Controller
        {
            private readonly IConfigurationRoot _config;
            public ProductsController(IConfigurationRoot config)
            {
                _config = config;
            }

            // GET api/products
            [HttpGet]
            public IEnumerable<string> Get()
            {
                Console.WriteLine($"product1 is {_config["product1"]}");
                Console.WriteLine($"product2 is {_config["product2"]}");
                return new[] {$"{_config["product1"]}", $"{_config["product2"]}"};
            }
        }
    }
    ```

10. In the application root create a file and name it config.json and edit it in the following way to configure the service instance of configuration server to connect to a git repo at the configured uri.

    ```json
    {
        "git": {
        "uri": "https://github.com/tezizzm/cloud-native-net-configs"
        }
    }
    ```

11. Run the following command to create an instance of Spring Cloud Config Server with settings from config.json **note: service name and type may be different depending on platform/operator configuration**

    ```
    cf create-service p-config-server standard myConfigServer -c .\config.json
    ```

12. You are ready to now “push” your application.  Create a file at the root of your application name it manifest.yml and edit it as follows:

    ```yml
    ---
    applications:
    - name: dotnet-core-api
        random-route: true
        buildpack: https://github.com/cloudfoundry/dotnet-core-buildpack
        instances: 1
        memory: 256M
        env:
        ASPNETCORE_ENVIRONMENT: development
        services:
        - myConfigServer
    ```

12. Run the cf push command to build, stage and run your application on PCF.  Ensure you are in the same directory as your manifest file and type `cf push`.

13. Once the `cf push` command has completed navigate to the given url and you should see the Swagger page.  If you navigate to the api/products path you should see a list of products that are pulled from configuration.  To confirm the configuration have a look at the configured git repo in the config.json file.