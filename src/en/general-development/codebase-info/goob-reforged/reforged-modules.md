# Reforged Modules

Goobstation provides a way to isolate your code into separate, custom modules. These custom modules retain full functionality like the core modules, while solving the problem of having to recompile the entire game when making changes specific to your downstream.

## Why

As stated above, there are two primary reasons:

1. **Streamlined Testing and Deployment**: By separating downstream-specific code from the rest of the game, you can compile and test your custom features independently.

2. **Enhanced Modularity**: Instead of managing files and folders within the already bloated Content.Client/Shared/Server projects, you can organize your work into four distinct, fork-specific projects.

This approach is specifically designed to accommodate [Robust Modularity](https://github.com/space-wizards/docs/pull/152) should it be implemented in the future.

## Creating new modules

Goob Reforged repository has a script to automatically create a module and connect it to the game in `/Tools/create_module.py`. Launch it through a command line, specify the name, and you have your module set up and ready!

## Terminology

- **Core Module**: Modules belonging to the base space-station-14 repository. These include `Content.Client`, `Content.Server`, `Content.Shared`, and their dependencies like `Content.Shared.Database`.

- **Custom Module**: Modules that are part of the Client or Server process but don't belong to Core Modules. For example, in Goobstation's case, these are prefixed with `Content.Goobstation.`.

- **Engine Module**: Modules that are part of RobustToolbox (the game engine). These are prefixed with `Robust.`.

- **.Common**: A module type intended to be shared between Custom and Core modules. These hold interfaces, components, and other types that both module types need to share.

- **Infix**: The module identifier, which is the middle part of a module name. For example, "Goobstation" in `Content.Goobstation.Common`.

## How

### Module manager

The client loader, unless configured otherwise, loads every Assembly from the `Assemblies` folder with a specific prefix.
This prefix can be defined in the `manifest.yml` file and defaults to `Content.`.

By creating new projects with the `Content.` prefix, you can have the game automatically load them at startup alongside the core modules.

### Dependency tree

Here's an example of the module dependency tree for Goobstation modules:

```mermaid
flowchart TD
    CoreServer[Content.Server]
    style CoreServer stroke:#6b9bb3
    CoreClient[Content.Client]
    style CoreClient stroke:#6b9bb3
    CoreShared[Content.Shared]
    style CoreShared stroke:#6b9bb3

    GoobServer[Content.Goobstation.Server]
    style GoobServer stroke:#3498db
    GoobClient[Content.Goobstation.Client]
    style GoobClient stroke:#3498db
    GoobShared[Content.Goobstation.Shared]
    style GoobShared stroke:#3498db
    GoobCommon[Content.Goobstation.Common]
    style GoobCommon stroke:#3498db

    ModuleServer[Content.Modules.Server]
    style GoobShared stroke:#3498db
    ModuleClient[Content.Modules.Client]
    style GoobCommon stroke:#3498db
    
    subgraph CoreModules["Core Modules"]
        CoreServer
        CoreClient
        CoreShared
    end
    style CoreModules stroke:#e91e63

    CoreShared --> CoreServer
    CoreShared --> CoreClient

    subgraph GoobModules["Goob Modules"]
        GoobServer
        GoobClient
        GoobShared
        GoobCommon
    end
    style GoobModules stroke:#e91e63

    GoobShared --> GoobServer
    GoobShared --> GoobClient

    GoobCommon --> CoreShared
    CoreShared --> GoobShared
    CoreServer --> GoobServer
    CoreClient --> GoobClient
    GoobServer --> ModuleServer
    GoobClient --> ModuleClient
```

This means:
- Custom modules directly reference their core module counterparts.
- Core modules cannot directly access code from Custom modules if they are of the same type (Core.Shared-Custom.Shared), as this would create a circular dependency.
- If the types are the same, core modules must communicate with Custom modules through an intermediary `.Common` module using events.
- The Common module must Not depend on either core or Custom modules and should be able to build standalone.

### Module compilation

However, the module loader does not ensure that the modules actually compile in the right order.

By default, when a project is run, MSBuild will compile all references of that project first, and then run it. The issue is that custom modules depend on the core modules, so they don't compile automatically by default!

In order to fix this, each Core project should have a special .csproj `Target` that waits until the project is fully compiled, and then compiles all custom modules connected to this core module. After that the results are copied into the bin folder where all other assemblies are located.

### Up-to-date compilation problem

Unfortunately, the method of Module Compilation described above doesn't solve the issue entirely. When no changes are made to any files in the core module, the .csproj `Target` doesn't trigger, and modules are ignored even if they have some changes.

This problem is solved by adding the `Content.Modules.Server` and `Content.Modules.Client` projects. They reference all modules at once as a direct Project reference, so when a change is made to any module, it gets recompiled correctly.

That's why you should **always use `Content.Modules` projects for development!**
