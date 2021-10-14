# Modular application development Part 1 (Single database)

In this post we will see how to develop a modular abp application. We will explore the Module template and then create sample modules. all module will use the same database to store the modules data and the identity data.

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

This will apply the migrations to the db and we can run the `MainApp.Web.Unified` project. This will host the UI and API for all the modules.

### Add a new Module

Now we will add a new module to our MainApp

```bash
abp add-module ModuleOne --new --add-to-solution-file
```

This command will create a new module and add the new module to the solution.

Now you can run the `MainApp.Web.Unified` and see the Api and UI available in the app.

## Add new Entity to the ModuleOne

We will create a new Entity inside the ModuleOne called `TodoOne`.

## 1. Create an [Entity](https://docs.abp.io/en/abp/latest/Entities)

First step is to create an Entity. Create the Entity in the `Domain` project

```cs
public class TodoOne : Entity<Guid>
{
    public string Content { get; set; }
    public bool IsDone { get; set; }
}
```

## 2. Add Entity to [ef core](https://docs.abp.io/en/abp/latest/Entity-Framework-Core)

Next is to add Entity to the EF Core. you will find the DbContext in the `EntityFrameworkCore` project. Add the DbSet to the DbContext

```cs
public DbSet<TodoOne> TodoOnes { get; set; }
```

## 3. Configure Entity in [ef core](https://docs.abp.io/en/abp/latest/Entity-Framework-Core#configurebyconvention-method)

Configuration is done in the `DbContextModelCreatingExtensions` class. This should be available in the `EntityFrameworkCore` project

```cs
builder.Entity<TodoOne>(b =>
{
    b.ToTable(TodoOnesConsts.DbTablePrefix + "TodoOnes", TodoOnesConsts.DbSchema);
    b.ConfigureByConvention(); //auto configure for the base class props
});
```

## 4. Adding Migrations

Now the Entity is configured we can add the migrations.

Go the `EntityFrameworkCore.DbMigrations` project in the terminal and create migrations.

To create migration run this command:

```bash
dotnet ef migrations add created_TodoOne
```

Verify the migrations created in the migrations folder.

To update the database run this command

```bash
dotnet ef database update
```

## 5. Create a Entity Dto

Dto are placed in `Contracts` project

```cs
public class TodoOneDto : EntityDto<Guid>
{
    public string Content { get; set; }
    public bool IsDone { get; set; }
}
```

## 6. Map Entity to Dto

Abp uses AutoMapper to map Entity to Dto. you can find the `ApplicationAutoMapperProfile` file which is used by the AutoMapper in the `Application` project.

```cs
CreateMap<TodoOne, TodoOneDto>();
CreateMap<TodoOneDto, TodoOne>();
```

## 7. Create an [Application Services](https://docs.abp.io/en/abp/latest/Application-Services)

Application service are created in the `Application` project

```cs
public class TodoOneAppService : ModuleOneAppService
{
    private readonly IRepository<TodoOne, Guid> todoOneRepository;

    public TodoOneAppService(IRepository<TodoOne, Guid> todoOneRepository)
    {
        this.todoOneRepository = todoOneRepository;
    }

    public async Task<List<TodoOneDto>> GetAll()
    {
        return ObjectMapper.Map<List<TodoOne>, List<TodoOneDto>>(await todoOneRepository.GetListAsync());
    }

    public async Task<TodoOneDto> CreateAsync(TodoOneDto todoOneDto)
    {
        var TodoOne = ObjectMapper.Map<TodoOneDto, TodoOne>(todoOneDto);
        var createdTodoOne = await todoOneRepository.InsertAsync(TodoOne);
        return ObjectMapper.Map<TodoOne, TodoOneDto>(createdTodoOne);
    }

    public async Task<TodoOneDto> UpdateAsync(TodoOneDto todoOneDto)
    {
        var TodoOne = ObjectMapper.Map<TodoOneDto, TodoOne>(todoOneDto);
        var createdTodoOne = await todoOneRepository.UpdateAsync(TodoOne);
        return ObjectMapper.Map<TodoOne, TodoOneDto>(createdTodoOne);
    }

    public async Task<bool> DeleteAsync(Guid id)
    {
        var TodoOne = await todoOneRepository.FirstOrDefaultAsync(x=> x.Id == id);
        if(TodoOne != null)
        {
            await todoOneRepository.DeleteAsync(TodoOne);
            return true;
        }
        return false;
    }
}
```

## 8. Update `AddAbpDbContext` method in the `ModuleOneEntityFrameworkCoreModule`

```cs
options.AddDefaultRepositories(includeAllEntities: true);
```

## 9. Update the `OnModelCreating` in the `UnifiedDbContext` in the `MainApp.Web.Unified` project

```cs
modelBuilder.ConfigureModuleOne();
```
