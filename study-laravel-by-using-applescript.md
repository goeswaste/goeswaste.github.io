---
title: Study Laravel by Using AppleScript
date: 2026-09-23 04:26:39
permalink: /Study-Laravel-by-Using-AppleScript/
---

A good way to approach this is **not** to recreate Laravel’s web stack literally. Instead, recreate its *architectural ideas* in AppleScript:

- Application bootstrap
- Service container
- Service providers
- Routing
- Middleware
- Controllers
- Configuration
- Events
- Facades
- A command-line entry point

Laravel’s request lifecycle is a useful model: the application boots providers, sends a request through middleware, dispatches it to a route/controller, then sends the response back through middleware. <citation src="5"></citation>

## 1. Define the learning goal

Build a small automation framework that can handle commands such as:

```bash
osascript app.scpt greet Alice
osascript app.scpt file:list ~/Documents
osascript app.scpt app:config
```

Instead of HTTP requests, your framework can initially process **command-line requests**.

The conceptual mapping would be:

| Laravel concept | AppleScript equivalent |
|---|---|
| `public/index.php` | `app.scpt` entry point |
| HTTP request | Command-line argument list |
| HTTP response | Returned record or printed text |
| Router | Command-to-handler map |
| Controller | Script object with handlers |
| Service container | Registry of constructors/instances |
| Service provider | Script that registers services |
| Middleware | Handlers wrapping execution |
| Configuration | Property list or AppleScript record |
| Artisan | Your command-line dispatcher |
| Facade | Global helper script or wrapper |

Start with this small scope:

```text
Input → Bootstrap → Providers → Middleware → Router → Controller → Output
```

Do not begin with views, databases, queues, or dependency injection. Those features will make more sense after the core lifecycle works.

## 2. Create a project structure

AppleScript does not provide Laravel-style namespaces, Composer autoloading, or conventional directories automatically, so you will need to establish your own conventions.

For example:

```text
StudyKit/
├── app.scpt
├── bootstrap.scpt
├── config/
│   └── app.scpt
├── framework/
│   ├── Application.scpt
│   ├── Container.scpt
│   ├── Router.scpt
│   ├── Middleware.scpt
│   ├── Request.scpt
│   └── Response.scpt
├── providers/
│   └── AppProvider.scpt
├── controllers/
│   └── GreetingController.scpt
└── tests/
    └── RouterTests.scpt
```

For a first prototype, you can keep everything in one script. Once the lifecycle works, split components into separate compiled scripts.

AppleScript’s main reusable unit is a `script` object containing properties and handlers:

```applescript
script GreetingController
	on greet(request)
		set personName to request's arguments's item 1
		return "Hello, " & personName & "!"
	end greet
end script
```

## 3. Build the request and response objects

Use records to represent framework objects.

### Request

```applescript
on makeRequest(commandName, commandArguments)
	return {command:commandName, arguments:commandArguments, attributes:{}}
end makeRequest
```

### Response

```applescript
on makeResponse(statusCode, body)
	return {status:statusCode, body:body, headers:{}}
end makeResponse
```

A request might look like this internally:

```applescript
{command:"greet", arguments:{"Alice"}, attributes:{}}
```

And a response:

```applescript
{status:200, body:"Hello, Alice!", headers:{}}
```

Using records rather than plain text will make later additions—headers, status codes, metadata, sessions—much easier.

## 4. Build the service container

Laravel’s service container manages dependencies and resolves services. Its central purpose is dependency management and dependency injection. <citation src="8"></citation>

In AppleScript, create a registry that stores either:

- A ready-made instance
- A constructor handler
- A singleton instance

A simple container could look like this:

```applescript
script Container
	property bindings : {}
	property instances : {}

	on bind(serviceName, factoryHandler)
		set end of bindings to {name:serviceName, factory:factoryHandler, singleton:false}
	end bind
	
	on singleton(serviceName, factoryHandler)
		set end of bindings to {name:serviceName, factory:factoryHandler, singleton:true}
	end singleton
	
	on make(serviceName)
		repeat with existingInstance in instances
			if (name of existingInstance) is serviceName then
				return value of existingInstance
			end if
		end repeat
		
		repeat with binding in bindings
			if (name of binding) is serviceName then
				set newInstance to factory of binding
				
				if singleton of binding then
					set end of instances to {name:serviceName, value:newInstance}
				end if
				
				return newInstance
			end if
		end repeat
		
		error "Service not bound: " & serviceName
	end make
end script
```

AppleScript handler references are awkward compared with PHP closures, so initially you can bind already-created script objects:

```applescript
container's bind("greeting.controller", GreetingController)
```

Later, experiment with handler references:

```applescript
script GreetingFactory
	on create()
		return GreetingController
	end create
end script
```

The container could then call the factory’s `create()` handler.

## 5. Add service providers

Laravel uses service providers to register and bootstrap major framework components. Providers generally have a registration phase and a boot phase. <citation src="5"></citation>

Model that directly:

```applescript
script AppProvider
	on register(app)
		app's container's bind("greeting.controller", GreetingController)
		app's container's bind("router", app's router)
	end register
	
	on boot(app)
		app's router's get("greet", "greeting.controller", "greet")
	end boot
end script
```

Your application object can load providers:

```applescript
script Application
	property container : missing value
	property router : missing value
	property providers : {}
	
	on create()
		set container to Container
		set router to Router
		return me
	end create
	
	on registerProvider(provider)
		set end of providers to provider
		provider's register(me)
	end registerProvider
	
	on boot()
		repeat with provider in providers
			provider's boot(me)
		end repeat
	end boot
end script
```

This teaches an important Laravel idea: application features should register themselves instead of being hardcoded into the central application object.

## 6. Implement the router

Laravel routes associate a verb and URI with a closure or controller action. <citation src="9"></citation> For your command-based framework, the “verb” can simply be the command name.

```applescript
script Router
	property routes : {}
	
	on get(routeName, controllerName, actionName)
		set end of routes to {name:routeName, controller:controllerName, action:actionName}
	end get
	
	on dispatch(request, container)
		repeat with route in routes
			if (name of route) is (command of request) then
				set controller to container's make(controller of route)
				return controller's |action|(request)
			end if
		end repeat
		
		return {status:404, body:"Command not found: " & (command of request), headers:{}}
	end dispatch
end script
```

AppleScript reserves some words and has unusual syntax for dynamic handler calls. A practical convention is to make controllers expose one generic handler:

```applescript
script GreetingController
	on |action|(request)
		set personName to arguments of request's item 1
		return {status:200, body:"Hello, " & personName & "!", headers:{}}
	end |action|
end script
```

Or dispatch explicitly based on the action name:

```applescript
if actionName is "greet" then
	return controller's greet(request)
end if
```

The second approach is less elegant but easier to debug while learning.

## 7. Add middleware

Middleware should wrap request handling. A middleware can:

1. Inspect the request
2. Reject it
3. Call the next handler
4. Modify the response afterward

That is also how Laravel describes middleware: layers that can examine a request, stop it, or pass it deeper into the application. <citation src="6"></citation>

Represent middleware as scripts:

```applescript
script LoggingMiddleware
	on handle(request, nextHandler)
		log "Starting command: " & (command of request)
		
		set response to nextHandler(request)
		
		log "Finished with status: " & (status of response)
		return response
	end handle
end script
```

Then create a pipeline:

```applescript
on runMiddleware(request, middlewareList, finalHandler)
	if (count of middlewareList) is 0 then
		return finalHandler(request)
	end if
	
	set currentMiddleware to middlewareList's item 1
	set remainingMiddleware to items 2 thru -1 of middlewareList
	
	script NextMiddleware
		on call(nextRequest)
			return runMiddleware(nextRequest, remainingMiddleware, finalHandler)
		end call
	end script
	
	return currentMiddleware's handle(request, NextMiddleware)
end runMiddleware
```

Conceptually:

```text
LoggingMiddleware
    → AuthenticationMiddleware
        → Router
            → Controller
```

The most important detail is that middleware can execute code both before and after the next handler:

```applescript
on handle(request, nextHandler)
	-- Before
	
	set response to nextHandler's call(request)
	
	-- After
	
	return response
end handle
```

## 8. Create the bootstrap sequence

Your `app.scpt` should have very little application logic. It should mainly assemble and run the framework.

```applescript
on run argv
	set app to Application's create()
	
	app's registerProvider(AppProvider)
	app's boot()
	
	set commandName to argv's item 1
	set commandArguments to items 2 thru -1 of argv
	
	set request to makeRequest(commandName, commandArguments)
	
	script FinalHandler
		on call(currentRequest)
			return app's router's dispatch(currentRequest, app's container)
		end call
	end script
	
	set middlewareList to {LoggingMiddleware}
	set response to runMiddleware(request, middlewareList, FinalHandler)
	
	display (body of response)
end run
```

The flow is now:

```text
argv
 ↓
Request
 ↓
Application bootstrap
 ↓
Service providers
 ↓
Middleware pipeline
 ↓
Router
 ↓
Controller
 ↓
Response
 ↓
Printed output
```

Run it conceptually like this:

```bash
osascript app.scpt greet Alice
```

Expected output:

```text
Hello, Alice!
```

## 9. Add configuration

Start with a simple configuration script:

```applescript
script Config
	property values : {appName:"StudyKit", environment:"local", debug:true}
	
	on get(keyName)
		return values's keyName
	end get
end script
```

A more practical version can load a property list:

```applescript
property configPath : "~/StudyKit/config/app.plist"
```

You can use AppleScriptObjC or shell commands to read property lists. Keep configuration separate from application logic, just as Laravel separates configuration from services and controllers.

Your container could expose configuration as a service:

```applescript
container's singleton("config", Config)
```

Then a controller can retrieve it:

```applescript
set config to app's container's make("config")
set appName to config's get("appName")
```

## 10. Add a facade-like interface last

Laravel facades provide convenient static-looking access to services in the container. Do not implement this first; it hides the mechanics you are trying to learn.

Once the container works, create a small facade:

```applescript
script Log
	property app : missing value
	
	on configure(application)
		set app to application
	end configure
	
	on info(messageText)
		log messageText
	end info
end script
```

Bootstrap it:

```applescript
Log's configure(app)
```

Then elsewhere:

```applescript
Log's info("Application started")
```

The educational point is to compare:

```text
Explicit dependency:
controller → receives logger

Facade:
controller → calls global Log facade
```

The explicit version is usually easier to test. The facade version is more convenient but more implicit.

## 11. Build in this order

A sensible progression is:

1. **Command-line entry point**
   - Read `argv`
   - Print a response

2. **Request and response records**
   - Stop passing loose strings around

3. **Router**
   - Map command names to handlers

4. **Controllers**
   - Move application logic out of the router

5. **Application object**
   - Centralize framework state

6. **Service container**
   - Register and resolve services

7. **Service providers**
   - Move registration out of the bootstrap file

8. **Middleware**
   - Add logging, validation, authorization, timing

9. **Configuration**
   - Load settings from records or plist files

10. **Events**
   - Implement event listeners and dispatching

11. **Testing helpers**
   - Fake requests
   - Fake services
   - Assert responses

12. **Optional HTTP adapter**
   - Use a local web server or another process later
   - Keep HTTP outside the core framework

## 12. A good first milestone

Your first finished milestone should support:

```bash
osascript app.scpt greet Alice
osascript app.scpt time
osascript app.scpt config
```

With:

- Three routes
- One controller
- One service
- One provider
- One logging middleware
- One missing-route response
- A small test script

For example:

```text
app.scpt
 ├── Application
 ├── Container
 ├── Router
 ├── Request
 ├── Response
 ├── AppProvider
 ├── GreetingController
 └── LoggingMiddleware
```

Once that works, add a `Clock` service:

```applescript
script Clock
	on now()
		return current date
	end now
end script
```

Register it:

```applescript
container's singleton("clock", Clock)
```

Use it in a controller:

```applescript
script TimeController
	property app : missing value
	
	on configure(application)
		set app to application
	end configure
	
	on |action|(request)
		set clock to app's container's make("clock")
		return {status:200, body:(clock's now as text), headers:{}}
	end |action|
end script
```

That gives you a small but meaningful demonstration of dependency resolution.

The key principle is to reproduce Laravel’s **separation of concerns**, not its syntax. AppleScript is well suited for experimenting with scripts, handlers, records, and automation, but it is not naturally suited to large, dynamically discovered class hierarchies. Keep the framework explicit, small, and inspectable so that every abstraction remains understandable.