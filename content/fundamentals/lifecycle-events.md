### Lifecycle Events

A Nest application, as well as every application element, has a lifecycle managed by Nest. Nest provides **lifecycle hooks** that give visibility into key lifecycle events, and the ability to act (run registered code on your `module`, `injectable` or `controller`) when they occur.

#### Lifecycle sequence

The following diagram depicts the sequence of key application lifecycle events, from the time the application is bootstrapped until the node process exits. We can divide the overall lifecycle into three phases: **initializing**, **running** and **terminating**. Using this lifecycle, you can plan for appropriate initialization of modules and services, manage active connections, and gracefully shutdown your application when it receives a termination signal.

<figure><img src="/assets/lifecycle-events.png" /></figure>

<p style="clear: both;"></p>

#### Lifecycle events

Lifecycle events happen during application bootstrapping and shutdown. Nest calls registered lifecycle hook methods on `modules`, `injectables` and `controllers` at each of the following lifecycle events (**shutdown hooks** need to be enabled first, as described [below](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)). As shown in the diagram above, Nest also calls the appropriate underlying methods to begin listening for connections, and to stop listening for connections.

| Lifecycle hook method         | Lifecycle event triggering the hook method call                                                                                                                                                                   |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `onModuleInit()`              | Called once the host module's dependencies have been resolved.                                                                                                                                                    |
| `onApplicationBootstrap()`    | Called once all modules have been initialized, but before listening for connections.                                                                                                                              |
| `onModuleDestroy()`           | Called after a termination signal (e.g., `SIGTERM`) has been received.                                                                                                                                            |
| `beforeApplicationShutdown()` | Called after all `onModuleDestroy()` handlers have completed (Promises resolved or rejected);<br />once complete (Promises resolved or rejected), all existing connections will be closed (`app.close()` called). |
| `onApplicationShutdown()`     | Called after connections close (`app.close()` resolves).                                                                                                                                                          |

#### Usage

Each lifecycle hook is represented by an interface. Interfaces are technically optional because they do not exist after TypeScript compilation. Nonetheless, it's good practice to use them in order to benefit from strong typing and editor tooling. To register a lifecycle hook, implement the appropriate interface. For example, to register a method to be called during module initialization on a particular class (e.g., Controller, Provider or Module), implement the `OnModuleInit` interface by supplying an `onModuleInit()` method, as shown below:

```typescript
@@filename()
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
```

#### Asynchronous initialization

Both the `OnModuleInit` and `OnApplicationBootstrap` hooks allow you to defer the application initialization process (return a `Promise` or mark the method as `async` and `await` an asynchronous method completion in the method body).

```typescript
@@filename()
async onModuleInit(): Promise<void> {
  await this.fetch();
}
@@switch
async onModuleInit() {
  await this.fetch();
}
```

#### Application shutdown

The `beforeApplicationShutdown()` and `onApplicationShutdown()` hooks are called in the **terminating** phase (in response to system signals such as `SIGTERM`). This feature is often used with [Kubernetes](https://kubernetes.io/), [Heroku](https://www.heroku.com/) or similar services.

> warning **warning** Due to inherent platform limitations, NestJS has limited support for application shutdown hooks on Windows. You can expect `SIGINT` to work, as well as `SIGBREAK` and to some extent `SIGHUP` - [read more](https://nodejs.org/api/process.html#process_signal_events). However `SIGTERM` will never work on Windows because killing a process in the task manager is unconditional, "i.e., there's no way for an application to detect or prevent it". Here's some [relevant documentation](http://docs.libuv.org/en/v1.x/signal.html) from libuv to learn more about how `SIGINT`, `SIGBREAK` and others are handled on Windows. Also, see Node.js documentation of [Process Signal Events](https://nodejs.org/api/process.html#process_signal_events)

To use these hooks you must activate a listener which listens to shutdown signals.

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  // Starts listening for shutdown hooks
  app.enableShutdownHooks();
  await app.listen(3000);
}
bootstrap();
```

> info **Info** `enableShutdownHooks` consumes memory by starting listeners. In cases where you are running multiple Nest apps in a single Node process (e.g., when running parallel tests with Jest), Node may complain about excessive listener processes. For this reason, `enableShutdownHooks` is not enabled by default. Be aware of this condition when you are running multiple instances in a single Node process.

When the application receives a termination signal it will call any registered `beforeApplicationShutdown()`, then `onApplicationShutdown()` methods (in the sequence described above) with the corresponding signal as the first parameter. If a registered function awaits an asynchronous call (returns a promise), Nest will not continue in the sequence until the promise is resolved or rejected.

```typescript
@@filename()
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // e.g. "SIGINT"
  }
}
@@switch
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal) {
    console.log(signal); // e.g. "SIGINT"
  }
}
```
