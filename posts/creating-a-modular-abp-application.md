# Modular application development Part 1 (Single database)

In this post we will see how to develop a modular abp application. We will explore the Module template and then create sample modules. Each module will use the one database to store the modules data and separate database for the identity data.

## Modular application in ABP

Modular application template is a good starting point for your modular application and abp cli has good tooling available for creating new modules.

### Creating the modular application

```bash
abp new MainApp -t module
```

This will create a default abp modular template. Now we have 2 options hosting everything together or hosting things separate. We will choose to host everything together.

Navigate to the `MainApp.Web.Unified` and make sure the migrations are available and run the database update command to update the database.

```bash
dotnet ef database update
```

This will apply the migrations to the db and we can run the `Unified` project. This will host the UI and API for all the modules.

### Add a new Module

Now we will add a new module to our MainApp

```bash
abp add-module ModuleOne --new --add-to-solution-file
```

This command will create a new module and add the new module to the solution.
