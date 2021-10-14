# Modular application development Part 2 (Two database)

In this post we will see how to develop a modular abp application. We will explore the Module template and then create sample modules. Module will use a separate database to store the modules data and separate database for the identity data.

## Modular application in ABP

Modular application template is a good starting point for your modular application and abp cli has good tooling available for creating new modules.

### Creating the modular application

```bash
abp new MainApp -t module
```

This will create a default abp modular template. Now we have 2 options hosting everything together or hosting things separate. We will choose to host everything separate.
