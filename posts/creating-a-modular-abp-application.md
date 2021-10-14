# Modular application development Part 1 (Single database)

In this post we will see how to develop a modular abp application. We will explore the Module template and then create sample modules. Each module will use the one database to store the modules data and separate database for the identity data.

## Modular application in ABP

Modular application template is a good starting point for your modular application and abp cli has good tooling available for creating new modules.

### Creating the modular application

```bash
abp new MainApp -t module
```
