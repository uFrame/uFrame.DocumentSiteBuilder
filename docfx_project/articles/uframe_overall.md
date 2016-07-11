# uFrame Overall

>Arthor : nitreo
>
>This article describes uFrame editor-side architecture as it was left after being open-sourced.
>
>This article covers only newer editor engine on which ECS runs and by all means ignores old MVVM-related engine.

Overall structure:

- Application Engine
- Diagram Engine
- Code Generation
- Database Engine

## Application Engine

### uFrame Editor Application

The key class is called InvertApplication which contains one magic property: Container

Container is one of the most important parts as it is shared throughout all the services and provides dependency injection and some other stuff.

At the moment that Container is first accessed from anywhere, all assemblies are analyzed to find types that inherit from CorePlugin. Those classes (plugins) are then initialized, registered inside the container and get ready for action. Thus, if you don’t touch container property, nothing will be loaded. The class that usually accesses container first is DesignerWindow. Thus, when you click Window -> Graph Designer, this window is created, container is touched and application is loaded.

uFrame follows plugin-based approach. Thus, you create a service/plugin by deriving your class from DiagramPlugin or CorePlugin. You then get access to entire uFrame architecture. Your class will be automatically loaded, if contained under “Editor” folder. It will also be registered in the container. DiagramPlugin and CorePlugin contain some lifecycle methods like Initialize and Loaded.

### Communication between plugins

Plugin communication is achieved via Interface-based event system which is further extended to support commands.

Basic approach is the following:

   InvertApplication.SignalEvent<ILoggedInEvent>(_ => _.LoggedIn()); 

Once this line is executed, LoggedIn() will be executed on every service (DiagramPlugin or CorePlugin) which implements ILoggedInEvent.

The more sophisticated approach is command extension which requires your service to implement special generic interface like this one: IExecuteCommand<LoginCommand>

You can then do InvertApplication.Execute(new LoginCommand());

This will invoke handlers on all services that implement  IExecuteCommand<LoginCommand>

This approach also triggeres ICommandExecuting and ICommandExecuted handlers. They are used to perform clean up or refresh some of the data based on what command has been executed.


## Diagram Engine
The Diagram Engine follows the following pattern:

NodeData (NodeModel) -> NodeViewModel -> NodeDrawer
ConnectionData (ConnectionModel) -> ConnectionViewModel -> ConnectionDrawer
Same applies for properties, slots, items, sections - everything.

Models are serialized into JSON, thus represent persistent data.
ViewModel represent runtime data and are bound to Models. 
Drawers are bound to viewmodels and are used to draw nodes.

Drawers are somewhat of an outstanding concept. 

Starting from ECS, we tend to do UI using the following pattern: 
Create interface IDrawSomething
Then, to draw something, you do InvertApplication.SignalEvent<IDrawSomething>...
This is how most of non-diagram UI currently works. 

However, Drawers were left as is, because they are generally faster.

###Note about Architect

Architect was(is) an internal project that let’s you generate Models, ViewModels and Drawers based on diagram data: i.e. it allows you to prototype your own node-based framework.
Architect, thus, can be built using architect :P

It’s is important to note, that architect still needs some polishing. It does not generate “Everything”. Afaik, some of the new features are not available directly from the architect. But it suffice to bootstrap your plugin. When using it, make sure you do not modify designer files and only mess up with editables.

##Code Generation

Code Generation framework started as a simple wrapper around code dome, but now it is sophisticated template-based codegen system. It can generate classes based off any data.

Templates support attribute helpers which are sufficient in most cases. Should you need any crazy code generation rules, manual setup is always available along with direct access to code dome.

##Database engine

It may not be obvious, but as uFrame was progressing, it was constantly being knocked back by some of Unity architectural problems/features. One one those is scriptable object.

Back in MVVM days, all diagram data was stored inside serializable scriptable object. As we moved along and developed complex relationships between nodes/items, uFrame became slower and slower and more error prone due to data being monolithic and hard to manage.

When transitioning to ECS, Database Repository was created, that stored all the diagram data entries as separated JSON files, packed into so called Database.
The key idea here is that every record is defined by it’s type and ID.Database then contains folders for each type. Inside those folders, json files are stored, named after it’s ID. ex:

MyStuff.db / MyNamespace.MyType / my-id.json

Not only this decoupled a lot of uframe concepts, but also boosted performance a lot, giving a lot of space for data analysis and validation.

This Database repository can potentially be used on client side, with some modifications around data paths and some internal, uframe specific events. 


----------

**Random notes to process later**

##Koinonia

Koinonia was started as a package manager for uFrame. It was supposed to load different plugins/scripts/frameworks similar to asset store, nuget or npm. 

Motivation: asset store is great for art assets that do not need to be supported. But it is not that great for scripts. Sometime you want version control. Dependency management, installation etc. This is not possible with how asset store approaches things. So we attempted to create something better, but never had free time to finish it. 

Right now, when uFrame is not commercial anymore, I think it is better to build on top of existing package managers. Thus Koinonia related files can be ignored/deleted.

##What needs to be done to make uFrame truly alive (IMHO)

- Get uFrame outside of Unity. uFrame was always half-way platform agnostic. It should not be too hard to place it inside Visual Studio as an extension or standalone app. Make sure to use Hardware GL for UI layer. Making it unity-agnostic will attract more users with diagram-code-gen needs.
- Implement good package management. This is critical, as we don’t want to end up with 100 frameworks/plugins inside single repo
- Rework diagram drawing. Unity editor GUI with it’s quirks affected UI layer a lot. Tons of hacks were placed inside the code to keep it platform-agnostic.
- Implement strong multitasking, to validate data ahead of time.
- Change how textures are stored: do not use bear pngs. Embed them inside dedicated DLL file and construct Unity textures on the fly. This will keep any texture related mess away from the user.
