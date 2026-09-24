---
title: app.scpt
---

Below is the updated full AppleScript with detailed comments in both English and Chinese.

```applescript
(*
    StudyKit
    --------
    A small Laravel-inspired application structure implemented in AppleScript.

    English:
    This example demonstrates several common framework concepts:
    - Application bootstrapping
    - Service containers
    - Service providers
    - Routing
    - Controllers
    - Middleware
    - Requests and responses
    - Configuration services

    中文：
    这个示例使用 AppleScript 模拟了一些常见的框架概念：
    - 应用程序启动流程
    - 服务容器
    - 服务提供者
    - 路由
    - 控制器
    - 中间件
    - 请求与响应
    - 配置服务

    Run examples / 运行示例：

        osascript app.scpt greet Alice
        osascript app.scpt time
        osascript app.scpt config
*)

--------------------------------------------------------------------------------
-- Global application configuration
-- 全局应用程序配置
--------------------------------------------------------------------------------

(*
    English:
    This property stores basic application-level settings.

    中文：
    这个属性保存应用程序级别的基本设置。
*)
property appConfig : {appName:"StudyKit", environment:"local", debug:true}


--------------------------------------------------------------------------------
-- Request object
-- 请求对象
--------------------------------------------------------------------------------

script Request

	(*
        English:
        Create a request record.

        A request contains:
        - command: the command or route name
        - arguments: command-line arguments
        - attributes: additional request data

        中文：
        创建一个请求记录。

        请求包含：
        - command：命令或路由名称
        - arguments：命令行参数
        - attributes：额外的请求数据
    *)
	on makeRequest(commandName, commandArguments)
		return {¬
			command:commandName, ¬
			arguments:commandArguments, ¬
			attributes:{}¬
			}
	end makeRequest

end script


--------------------------------------------------------------------------------
-- Response object
-- 响应对象
--------------------------------------------------------------------------------

script Response

	(*
        English:
        Create a response record.

        statusCode is normally an HTTP-style status code:
        - 200 means success
        - 400 means bad input
        - 404 means the command was not found

        中文：
        创建一个响应记录。

        statusCode 通常使用类似 HTTP 的状态码：
        - 200 表示成功
        - 400 表示输入错误
        - 404 表示找不到命令
    *)
	on makeResponse(statusCode, responseBody)
		return {¬
			status:statusCode, ¬
			body:responseBody, ¬
			headers:{}¬
			}
	end makeResponse

end script


--------------------------------------------------------------------------------
-- Service container
-- 服务容器
--------------------------------------------------------------------------------

script Container

	(*
        English:
        bindings stores the definitions of registered services.

        Example:
            "clock" -> Clock
            "config" -> Config

        中文：
        bindings 保存已经注册的服务定义。

        例如：
            "clock" -> Clock
            "config" -> Config
    *)
	property bindings : {}

	(*
        English:
        instances stores already-created singleton instances.

        A singleton is created once and reused whenever it is requested.

        中文：
        instances 保存已经创建的单例服务。

        单例服务只创建一次，之后每次请求都会重复使用同一个对象。
    *)
	property instances : {}


	(*
        English:
        Register a normal service binding.

        A normal binding does not promise that the service will be reused
        as a singleton.

        中文：
        注册一个普通服务绑定。

        普通绑定不保证服务对象会作为单例重复使用。
    *)
	on bind(serviceName, serviceObject)

		set end of bindings to {¬
			serviceName:serviceName, ¬
			serviceObject:serviceObject, ¬
			singleton:false¬
			}

	end bind


	(*
        English:
        Register a singleton service.

        singleton:true tells the container to cache the service after the
        first resolution.

        中文：
        注册一个单例服务。

        singleton:true 告诉容器：服务第一次解析后应该被缓存。
    *)
	on singleton(serviceName, serviceObject)

		set end of bindings to {¬
			serviceName:serviceName, ¬
			serviceObject:serviceObject, ¬
			singleton:true¬
			}

	end singleton


	(*
        English:
        Resolve a service by name.

        The method first checks whether a singleton instance was already
        created. If so, it returns the cached instance.

        Otherwise, it searches the registered bindings.

        中文：
        根据名称解析服务。

        这个方法首先检查单例服务是否已经创建。
        如果已经创建，就返回缓存中的对象。

        否则，它会搜索已经注册的服务绑定。
    *)
	on resolve(serviceName)

		------------------------------------------------------------
		-- Step 1: Search the singleton instance cache
		-- 第一步：搜索单例实例缓存
		------------------------------------------------------------

		repeat with storedInstance in instances

			set instanceRecord to contents of storedInstance

			if (instanceRecord's serviceName) is serviceName then
				return instanceRecord's serviceObject
			end if

		end repeat


		------------------------------------------------------------
		-- Step 2: Search registered service bindings
		-- 第二步：搜索已经注册的服务绑定
		------------------------------------------------------------

		repeat with binding in bindings

			set bindingRecord to contents of binding

			if bindingRecord's serviceName is serviceName then

				set resolvedObject to bindingRecord's serviceObject


				--------------------------------------------------------
				-- Cache singleton services for future requests
				-- 缓存单例服务，以便后续请求重复使用
				--------------------------------------------------------

				if bindingRecord's singleton is true then
					set end of instances to {¬
						serviceName:serviceName, ¬
						serviceObject:resolvedObject¬
						}
				end if

				return resolvedObject
			end if

		end repeat


		------------------------------------------------------------
		-- The service was not registered
		-- 服务没有被注册
		------------------------------------------------------------

		error "Service is not registered: " & serviceName

	end resolve

end script


--------------------------------------------------------------------------------
-- Router
-- 路由器
--------------------------------------------------------------------------------

script Router

	(*
        English:
        routes contains command-to-controller mappings.

        Example:
            "greet" -> "greeting.controller"
            "time"  -> "time.controller"

        中文：
        routes 保存命令到控制器的映射关系。

        例如：
            "greet" -> "greeting.controller"
            "time"  -> "time.controller"
    *)
	property routes : {}


	(*
        English:
        Register a route.

        commandName:
            The command entered by the user.

        serviceName:
            The controller's service-container name.

        actionName:
            The handler method to execute.

        中文：
        注册一个路由。

        commandName：
            用户输入的命令。

        serviceName：
            控制器在服务容器中的名称。

        actionName：
            要执行的处理方法名称。
    *)
	on register(commandName, serviceName, actionName)

		set end of routes to {¬
			commandName:commandName, ¬
			serviceName:serviceName, ¬
			actionName:actionName¬
			}

	end register


	(*
        English:
        Dispatch a request to the matching controller.

        中文：
        将请求分发给匹配的控制器。
    *)
	on dispatch(requestRecord, serviceContainer)

		set requestedCommand to requestRecord's command


		------------------------------------------------------------
		-- Search for a matching route
		-- 查找匹配的路由
		------------------------------------------------------------

		repeat with route in routes

			set routeRecord to contents of route

			if routeRecord's commandName is requestedCommand then

				----------------------------------------------------
				-- Resolve the controller from the service container
				-- 从服务容器中解析控制器
				----------------------------------------------------

				set controller to serviceContainer's ¬
					resolve(routeRecord's serviceName)


				----------------------------------------------------
				-- Execute the controller action
				-- 执行控制器动作
				----------------------------------------------------

				return controller's execute(requestRecord)

			end if

		end repeat


		------------------------------------------------------------
		-- No route matched the request
		-- 没有路由匹配这个请求
		------------------------------------------------------------

		return Response's makeResponse(404, ¬
			"Command not found: " & requestedCommand)

	end dispatch

end script


--------------------------------------------------------------------------------
-- Clock service
-- 时钟服务
--------------------------------------------------------------------------------

script Clock

	(*
        English:
        Return the current date and time.

        中文：
        返回当前日期和时间。
    *)
	on now()
		return current date
	end now

end script


--------------------------------------------------------------------------------
-- Configuration service
-- 配置服务
--------------------------------------------------------------------------------

script Config

	(*
        English:
        Configuration values are stored in a record.

        中文：
        配置值保存在一个 AppleScript record 中。
    *)
	property values : {¬
		appName:"StudyKit", ¬
		environment:"local", ¬
		debug:true¬
		}


	(*
        English:
        Return one configuration value by key.

        中文：
        根据配置键返回一个配置值。
    *)
	on getValue(keyName)

		if keyName is "appName" then
			return values's appName
		end if

		if keyName is "environment" then
			return values's environment
		end if

		if keyName is "debug" then
			return values's debug
		end if


		------------------------------------------------------------
		-- Protect against unknown configuration keys
		-- 防止访问未知的配置键
		------------------------------------------------------------

		error "Unknown configuration key: " & keyName

	end getValue

end script


--------------------------------------------------------------------------------
-- Greeting controller
-- 问候控制器
--------------------------------------------------------------------------------

script GreetingController

	(*
        English:
        The application property allows the controller to access the
        application and its service container.

        中文：
        application 属性允许控制器访问应用对象及其服务容器。
    *)
	property application : missing value


	(*
        English:
        Store a reference to the main application object.

        中文：
        保存主应用对象的引用。
    *)
	on configure(applicationObject)
		set application to applicationObject
	end configure


	(*
        English:
        Handle the "greet" command.

        中文：
        处理 "greet" 命令。
    *)
	on execute(requestRecord)

		set requestArguments to requestRecord's arguments


		------------------------------------------------------------
		-- Validate the command arguments
		-- 验证命令参数
		------------------------------------------------------------

		if (count of requestArguments) is 0 then

			return Response's makeResponse(400, ¬
				"Usage: greet NAME")

		end if


		------------------------------------------------------------
		-- Read the first argument as the person's name
		-- 将第一个参数作为姓名
		------------------------------------------------------------

		set personName to requestArguments's item 1


		------------------------------------------------------------
		-- Build a successful response
		-- 创建成功响应
		------------------------------------------------------------

		set greeting to "Hello, " & personName & "!"

		return Response's makeResponse(200, greeting)

	end execute

end script


--------------------------------------------------------------------------------
-- Time controller
-- 时间控制器
--------------------------------------------------------------------------------

script TimeController

	property application : missing value


	(*
        English:
        Save the application reference.

        中文：
        保存应用程序引用。
    *)
	on configure(applicationObject)
		set application to applicationObject
	end configure


	(*
        English:
        Handle the "time" command.

        The controller obtains the Clock service from the container
        instead of creating the Clock object directly.

        中文：
        处理 "time" 命令。

        控制器从服务容器中获取 Clock 服务，
        而不是直接创建 Clock 对象。
    *)
	on execute(requestRecord)

		set clockService to application's container's resolve("clock")

		set currentTime to clockService's now()

		return Response's makeResponse(200, currentTime as text)

	end execute

end script


--------------------------------------------------------------------------------
-- Configuration controller
-- 配置控制器
--------------------------------------------------------------------------------

script ConfigController

	property application : missing value


	(*
        English:
        Save the application reference.

        中文：
        保存应用程序引用。
    *)
	on configure(applicationObject)
		set application to applicationObject
	end configure


	(*
        English:
        Handle the "config" command.

        中文：
        处理 "config" 命令。
    *)
	on execute(requestRecord)

		------------------------------------------------------------
		-- Resolve the configuration service
		-- 解析配置服务
		------------------------------------------------------------

		set configService to application's container's resolve("config")


		------------------------------------------------------------
		-- Read configuration values
		-- 读取配置值
		------------------------------------------------------------

		set appName to configService's getValue("appName")
		set environmentName to configService's getValue("environment")
		set debugEnabled to configService's getValue("debug")


		------------------------------------------------------------
		-- Format the configuration output
		-- 格式化配置输出
		------------------------------------------------------------

		set outputText to "Application: " & appName & return
		set outputText to outputText & ¬
			"Environment: " & environmentName & return
		set outputText to outputText & ¬
			"Debug: " & debugEnabled


		return Response's makeResponse(200, outputText)

	end execute

end script


--------------------------------------------------------------------------------
-- Logging middleware
-- 日志中间件
--------------------------------------------------------------------------------

script LoggingMiddleware

	(*
        English:
        Middleware runs before and after the final request handler.

        It receives:
        - requestRecord: the current request
        - nextHandler: an object representing the next middleware or handler

        中文：
        中间件会在最终请求处理器之前和之后运行。

        它接收：
        - requestRecord：当前请求
        - nextHandler：表示下一个中间件或处理器的对象
    *)
	on handle(requestRecord, nextHandler)

		------------------------------------------------------------
		-- Code before the controller executes
		-- 控制器执行前的代码
		------------------------------------------------------------

		log "Starting command: " & requestRecord's command


		------------------------------------------------------------
		-- Continue through the middleware chain
		-- 继续执行中间件链
		------------------------------------------------------------

		set responseRecord to nextHandler's call(requestRecord)


		------------------------------------------------------------
		-- Code after the controller executes
		-- 控制器执行后的代码
		------------------------------------------------------------

		log "Finished with status: " & responseRecord's status

		return responseRecord

	end handle

end script


--------------------------------------------------------------------------------
-- Application service provider
-- 应用程序服务提供者
--------------------------------------------------------------------------------

script AppProvider

	(*
        English:
        Register application services.

        This method is used to tell the container which services exist.

        中文：
        注册应用程序服务。

        这个方法告诉服务容器有哪些服务可以使用。
    *)
	on register(applicationObject)

		set serviceContainer to applicationObject's container


		------------------------------------------------------------
		-- Register singleton services
		-- 注册单例服务
		------------------------------------------------------------

		serviceContainer's singleton("config", Config)
		serviceContainer's singleton("clock", Clock)

		serviceContainer's singleton(¬
			"greeting.controller", GreetingController)

		serviceContainer's singleton(¬
			"time.controller", TimeController)

		serviceContainer's singleton(¬
			"config.controller", ConfigController)

	end register


	(*
        English:
        Boot the application after all services have been registered.

        This method registers routes and configures controllers.

        中文：
        在所有服务注册完成后启动应用程序。

        这个方法负责注册路由并配置控制器。
    *)
	on boot(applicationObject)

		set commandRouter to applicationObject's router


		------------------------------------------------------------
		-- Register command routes
		-- 注册命令路由
		------------------------------------------------------------

		commandRouter's register(¬
			"greet", ¬
			"greeting.controller", ¬
			"execute"¬
			)

		commandRouter's register(¬
			"time", ¬
			"time.controller", ¬
			"execute"¬
			)

		commandRouter's register(¬
			"config", ¬
			"config.controller", ¬
			"execute"¬
			)


		------------------------------------------------------------
		-- Configure the greeting controller
		-- 配置问候控制器
		------------------------------------------------------------

		set greetingController to applicationObject's container's ¬
			resolve("greeting.controller")

		greetingController's configure(applicationObject)


		------------------------------------------------------------
		-- Configure the time controller
		-- 配置时间控制器
		------------------------------------------------------------

		set timeController to applicationObject's container's ¬
			resolve("time.controller")

		timeController's configure(applicationObject)


		------------------------------------------------------------
		-- Configure the configuration controller
		-- 配置配置控制器
		------------------------------------------------------------

		set configController to applicationObject's container's ¬
			resolve("config.controller")

		configController's configure(applicationObject)

	end boot

end script


--------------------------------------------------------------------------------
-- Main application object
-- 主应用程序对象
--------------------------------------------------------------------------------

script Application

	(*
        English:
        The application owns the service container and router.

        中文：
        应用程序对象拥有服务容器和路由器。
    *)
	property container : missing value
	property router : missing value
	property providers : {}


	(*
        English:
        Create and initialize an application instance.

        中文：
        创建并初始化一个应用程序实例。
    *)
	on create()

		set container to Container
		set router to Router
		set providers to {}

		return me

	end create


	(*
        English:
        Register a service provider.

        The provider immediately registers its services.

        中文：
        注册一个服务提供者。

        服务提供者会立即注册自己的服务。
    *)
	on registerProvider(providerObject)

		set end of providers to providerObject

		providerObject's register(me)

	end registerProvider


	(*
        English:
        Boot all registered service providers.

        中文：
        启动所有已经注册的服务提供者。
    *)
	on boot()

		repeat with providerObject in providers

			providerObject's boot(me)

		end repeat

	end boot

end script


--------------------------------------------------------------------------------
-- Middleware pipeline
-- 中间件执行管线
--------------------------------------------------------------------------------

(*
    English:
    Run middleware recursively.

    Each middleware receives a "next" object. Calling next.call() continues
    execution with the remaining middleware.

    When no middleware remains, finalHandler is called.

    中文：
    递归执行中间件。

    每个中间件都会收到一个 "next" 对象。
    调用 next.call() 会继续执行剩余的中间件。

    当没有更多中间件时，就调用 finalHandler。
*)
on runMiddleware(requestRecord, middlewareList, finalHandler)

	------------------------------------------------------------
	-- Base case: no middleware remains
	-- 基础情况：没有剩余中间件
	------------------------------------------------------------

	if (count of middlewareList) is 0 then
		return finalHandler's call(requestRecord)
	end if


	------------------------------------------------------------
	-- Select the first middleware
	-- 取出第一个中间件
	------------------------------------------------------------

	set currentMiddleware to middlewareList's item 1


	------------------------------------------------------------
	-- Create a list containing the remaining middleware
	-- 创建包含剩余中间件的新列表
	------------------------------------------------------------

	if (count of middlewareList) is 1 then
		set remainingMiddleware to {}
	else
		set remainingMiddleware to items 2 thru -1 of middlewareList
	end if


	------------------------------------------------------------
	-- Create the "next" handler
	-- 创建 "next" 处理器
	------------------------------------------------------------

	script NextMiddleware

		on call(nextRequest)

			return runMiddleware(¬
				nextRequest, ¬
				remainingMiddleware, ¬
				finalHandler¬
				)

		end call

	end script


	------------------------------------------------------------
	-- Run the current middleware
	-- 执行当前中间件
	------------------------------------------------------------

	return currentMiddleware's handle(¬
		requestRecord, ¬
		NextMiddleware¬
		)

end runMiddleware


--------------------------------------------------------------------------------
-- Program entry point
-- 程序入口
--------------------------------------------------------------------------------

on run argv

	(*
        English:
        The run handler is the entry point when this script is executed
        using osascript.

        中文：
        当使用 osascript 执行此脚本时，
        run 处理器就是程序入口。
    *)


	------------------------------------------------------------
	-- Check whether a command was supplied
	-- 检查用户是否提供了命令
	------------------------------------------------------------

	if (count of argv) is 0 then

		display dialog ¬
			"Usage: osascript app.scpt COMMAND [ARGUMENTS]" ¬
			with title "StudyKit"

		return

	end if


	------------------------------------------------------------
	-- Create the application
	-- 创建应用程序
	------------------------------------------------------------

	set applicationObject to Application's create()


	------------------------------------------------------------
	-- Register the application provider
	-- 注册应用程序服务提供者
	------------------------------------------------------------

	applicationObject's registerProvider(AppProvider)


	------------------------------------------------------------
	-- Boot the application
	-- 启动应用程序
	------------------------------------------------------------

	applicationObject's boot()


	------------------------------------------------------------
	-- Read the command name
	-- 读取命令名称
	------------------------------------------------------------

	set commandName to argv's item 1


	------------------------------------------------------------
	-- Read command arguments
	-- 读取命令参数
	------------------------------------------------------------

	if (count of argv) is 1 then
		set commandArguments to {}
	else
		set commandArguments to items 2 thru -1 of argv
	end if


	------------------------------------------------------------
	-- Create a request record
	-- 创建请求记录
	------------------------------------------------------------

	set requestRecord to Request's makeRequest(¬
		commandName, ¬
		commandArguments¬
		)


	------------------------------------------------------------
	-- Define the final request handler
	-- 定义最终请求处理器
	------------------------------------------------------------

	script FinalHandler

		on call(requestToHandle)

			return applicationObject's router's dispatch(¬
				requestToHandle, ¬
				applicationObject's container¬
				)

		end call

	end script


	------------------------------------------------------------
	-- Configure the middleware list
	-- 配置中间件列表
	------------------------------------------------------------

	set middlewareList to {LoggingMiddleware}


	------------------------------------------------------------
	-- Run the request through the middleware pipeline
	-- 让请求经过中间件管线
	------------------------------------------------------------

	set responseRecord to runMiddleware(¬
		requestRecord, ¬
		middlewareList, ¬
		FinalHandler¬
		)


	------------------------------------------------------------
	-- Print the response body
	-- 输出响应正文
	------------------------------------------------------------

	display responseRecord's body

end run
```

Save the file as `app.scpt`, then run:

```bash
osascript app.scpt greet Alice
```

```text
Hello, Alice!
```

```bash
osascript app.scpt time
```

```text
Thursday, September 25, 2026 at ...
```

```bash
osascript app.scpt config
```

```text
Application: StudyKit
Environment: local
Debug: true
```

```bash
osascript app.scpt unknown
```

```text
Command not found: unknown
```

The `log` statements from `LoggingMiddleware` may appear in the AppleScript or Console logs rather than directly in the Terminal output.
