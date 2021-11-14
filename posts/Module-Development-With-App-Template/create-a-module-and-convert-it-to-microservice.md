# Add a new Module and convert it to a microservice in ABP

In this post we will see how to create a modular abp application and convert it to microservice. We will add a new module to tiered abp app and then use the separate database to store the modules data and then convert the module to a microservice.

## Creating the abp application and run migrations

```bash
abp new MainApp -t app -u mvc --tiered
```

## Run Migrations

Change directory to src/MainApp.DbMigrator and run the migration project

```bash
dotnet run
```

This will apply the migrations to the db and we can run the `MainApp.Web` project. This will host the UI and API..

## Add a new Module

Now we will add a new module to our MainApp. Move back to the `root` folder of your project.

```bash
abp add-module ProjectService --new
```

This will create a ProjectService and it will be available in the modules folder.

## Add and configure the host to the module

We need to add a host for our module. first we have to navigate to the the src folder of the Module.

```bash
cd .\modules\ProjectService\src\
```

Now lets create a Web Api project to host our module.

```bash
dotnet new webapi -n ProjectService.HttpApi.Host --framework "net5.0"
```

Update the `appsettings.json` with the following

```json
{
  "App": {
    "SelfUrl": "{Host Url}",
    "CorsOrigins": "https://*.MainApp.com"
  },
  "AuthServer": {
    "Authority": "{ Identity Server Url }",
    "RequireHttpsMetadata": "true",
    "SwaggerClientId": "ProjectService_Swagger",
    "SwaggerClientSecret": "1q2w3e*"
  },
  "ConnectionStrings": {
    "ProjectService": "Server=(LocalDb)\\MSSQLLocalDB;Database=ProjectService;Trusted_Connection=True"
  },
  "Redis": {
    "Configuration": "127.0.0.1"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  }
}
```

Add the following packages to the `ProjectService.HttpApi.Host` project.

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="5.0.0" />
    <PackageReference Include="Serilog.AspNetCore" Version="4.1.0" />
    <PackageReference Include="Volo.Abp.EntityFrameworkCore" Version="4.4.4" />
    <PackageReference Include="Volo.Abp.AspNetCore.MultiTenancy" Version="4.4.4" />
    <PackageReference Include="Volo.Abp.Autofac" Version="4.4.4" />
    <PackageReference Include="Volo.Abp.Core" Version="4.4.4" />
    <PackageReference Include="Volo.Abp.EntityFrameworkCore.SqlServer" Version="4.4.4" />
    <PackageReference Include="Volo.Abp.Swashbuckle" Version="4.4.4" />
    <PackageReference Include="Volo.Abp.AspNetCore.Authentication.JwtBearer" Version="4.4.4" />
    <PackageReference Include="Volo.Abp.AspNetCore.Serilog" Version="4.4.4" />
    <PackageReference Include="Serilog.Extensions.Logging" Version="3.0.1" />
    <PackageReference Include="Serilog.Sinks.Async" Version="1.4.0" />
    <PackageReference Include="Serilog.Sinks.File" Version="4.1.0" />
    <PackageReference Include="Serilog.Sinks.Console" Version="3.1.1" />
</ItemGroup>
```

Add `ProjectService.Application`, `ProjectService.EntityFrameworkCore`, `ProjectService.HttpApi` projects as a reference to your `ProjectService.HttpApi.Host`

Create a `ProjectServiceHostModule` in the newly created `ProjectService.HttpApi.Host`.

Here is the sample for the `ProductService`.

```cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Cors;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.OpenApi.Models;
using ProjectService.EntityFrameworkCore;
using System;
using System.Collections.Generic;
using System.Linq;
using Volo.Abp;
using Volo.Abp.AspNetCore.MultiTenancy;
using Volo.Abp.AspNetCore.Mvc;
using Volo.Abp.AspNetCore.Serilog;
using Volo.Abp.Autofac;
using Volo.Abp.Modularity;
using Volo.Abp.Swashbuckle;

namespace ProjectService.HttpApi.Host
{

    [DependsOn(
    typeof(ProjectServiceHttpApiModule),
    typeof(ProjectServiceApplicationModule),
    typeof(ProjectServiceEntityFrameworkCoreModule),
    typeof(AbpAspNetCoreMultiTenancyModule),
    typeof(AbpAutofacModule),
    typeof(AbpAspNetCoreSerilogModule),
    typeof(AbpSwashbuckleModule)
    )]
    public class ProjectServiceHostModule : AbpModule
    {
        public override void ConfigureServices(ServiceConfigurationContext context)
        {
            var configuration = context.Services.GetConfiguration();
            context.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddJwtBearer(options =>
                {
                    options.Authority = configuration["AuthServer:Authority"];
                    options.RequireHttpsMetadata = Convert.ToBoolean(configuration["AuthServer:RequireHttpsMetadata"]);
                    options.Audience = "ProjectService";
                });

            context.Services.AddAbpSwaggerGenWithOAuth(
                configuration["AuthServer:Authority"],
                new Dictionary<string, string>
                {
                    {"ProjectService", "ProjectService API"}
                },
                options =>
                {
                    options.SwaggerDoc("v1", new OpenApiInfo { Title = "ProjectService API", Version = "v1" });
                    options.DocInclusionPredicate((docName, description) => true);
                    options.CustomSchemaIds(type => type.FullName);
                });

            Configure<AbpAspNetCoreMvcOptions>(options =>
            {
                options.ConventionalControllers.Create(typeof(ProjectServiceApplicationModule).Assembly);
            });

            context.Services.AddCors(options =>
            {
                options.AddDefaultPolicy(builder =>
                {
                    builder
                        .WithOrigins(
                            configuration["App:CorsOrigins"]
                                .Split(",", StringSplitOptions.RemoveEmptyEntries)
                                .Select(o => o.Trim().RemovePostFix("/"))
                                .ToArray()
                        )
                        .WithAbpExposedHeaders()
                        .SetIsOriginAllowedToAllowWildcardSubdomains()
                        .AllowAnyHeader()
                        .AllowAnyMethod()
                        .AllowCredentials();
                });
            });
        }

        public override void OnApplicationInitialization(ApplicationInitializationContext context)
        {
            var app = context.GetApplicationBuilder();
            var env = context.GetEnvironment();

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseCorrelationId();
            app.UseCors();
            app.UseAbpRequestLocalization();
            app.UseStaticFiles();
            app.UseRouting();
            app.UseAuthentication();
            app.UseAbpClaimsMap();
            app.UseMultiTenancy();
            app.UseAuthorization();
            app.UseSwagger();
            app.UseAbpSwaggerUI(options => {
                options.SwaggerEndpoint("/swagger/v1/swagger.json", "ModuleA Service API");
                var configuration = context.ServiceProvider.GetRequiredService<IConfiguration>();
                options.OAuthClientId(configuration["AuthServer:SwaggerClientId"]); 
                options.OAuthClientSecret(configuration["AuthServer:SwaggerClientSecret"]); 
            });
            app.UseAbpSerilogEnrichers();
            app.UseAuditing();
            app.UseUnitOfWork();
            app.UseConfiguredEndpoints();
        }
    }
}
```

Update the `Program.cs`

```cs
using System;
using System.IO;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Serilog;
using Serilog.Events;

namespace ProjectService
{
    public class Program
    {
        public static int Main(string[] args)
        {
            Log.Logger = new LoggerConfiguration()
#if DEBUG
                .MinimumLevel.Debug()
#else
                .MinimumLevel.Information()
#endif
                .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
                .MinimumLevel.Override("Microsoft.EntityFrameworkCore", LogEventLevel.Warning)
                .Enrich.FromLogContext()
                .WriteTo.Async(c => c.File("Logs/logs.txt"))
#if DEBUG
                .WriteTo.Async(c => c.Console())
#endif
                .CreateLogger();

            try
            {
                Log.Information("Starting ModuleA.Host.");
                CreateHostBuilder(args).Build().Run();
                return 0;
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "Host terminated unexpectedly!");
                return 1;
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }

        internal static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureAppConfiguration(build =>
                {
                    build.AddJsonFile("appsettings.secrets.json", optional: true);
                })
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                })
                .UseAutofac()
                .UseSerilog();
    }

}
```

Update the `Startup.cs`

```cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using ProjectService.HttpApi.Host;

namespace ProjectService
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddApplication<ProjectServiceHostModule>();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILoggerFactory loggerFactory)
        {
            app.InitializeApplication();
        }
    }
}
```

Create a `HomeController.cs` in the `Controller` folder and update wit the following text.

```cs
using Microsoft.AspNetCore.Mvc;
using Volo.Abp.AspNetCore.Mvc;

namespace ProjectService.HttpApi.Host.Controllers
{
    public class HomeController : AbpController
    {
        public ActionResult Index()
        {
            return Redirect("~/swagger");
        }
    }
}
```

