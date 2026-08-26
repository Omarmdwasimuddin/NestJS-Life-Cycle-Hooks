# NestJS Lifecycle Events 

## Lifecycle Event কী?

একটা Nest application এবং তার প্রতিটা element (module, provider, controller)-এর একটা **lifecycle** থাকে, যেটা Nest নিজে manage করে। Nest কিছু **lifecycle hook** দেয়, যেগুলো দিয়ে গুরুত্বপূর্ণ lifecycle event-এর সময় নিজের code (module/provider/controller-এর ভিতরে registered) চালানো যায়।

### Lifecycle-এর ৩টা ধাপ

Application bootstrap হওয়া থেকে node process বন্ধ হওয়া পর্যন্ত পুরো lifecycle-কে ৩টা ধাপে ভাগ করা যায়:

1. **Initializing** (শুরু হওয়া)
2. **Running** (চলমান)
3. **Terminating** (বন্ধ হওয়া)

এই lifecycle ব্যবহার করে module/service-এর সঠিক initialization পরিকল্পনা করা যায়, active connection manage করা যায়, আর termination signal পেলে gracefully shutdown করা যায়।

---

## Lifecycle Hook-এর তালিকা

নিচের table-এ প্রতিটা hook method কখন call হয় তা দেখানো হলো:

| Hook Method | কখন call হয় |
|---|---|
| `onModuleInit()` | Host module-এর dependency resolve হয়ে যাওয়ার পর একবার call হয় |
| `onApplicationBootstrap()` | সবগুলো module initialize হওয়ার পর, কিন্তু connection listen শুরু হওয়ার আগে call হয় |
| `onModuleDestroy()` * | Termination signal (যেমন `SIGTERM`) পাওয়ার পর call হয় |
| `beforeApplicationShutdown()` * | সব `onModuleDestroy()` handler শেষ হওয়ার পর (Promise resolve/reject হওয়া পর্যন্ত) call হয়; এরপর সবগুলো existing connection বন্ধ হয়ে যায় (`app.close()` call হয়) |
| `onApplicationShutdown()` * | Connection বন্ধ হওয়ার পর (`app.close()` resolve হওয়ার পর) call হয় |

> **গুরুত্বপূর্ণ:**
> - `onModuleInit()` আর `onApplicationBootstrap()` — এই দুইটা তখনই trigger হয় যখন `app.init()` বা `app.listen()` explicitly call করা হয়।
> - `*` চিহ্নিত তিনটা (`onModuleDestroy`, `beforeApplicationShutdown`, `onApplicationShutdown`) তখনই trigger হয় যখন `app.close()` explicitly call করা হয়, **অথবা** process কোনো system signal (যেমন `SIGTERM`) পায় এবং bootstrap-এর সময় সঠিকভাবে `enableShutdownHooks` call করা হয়েছে (নিচে বিস্তারিত)।

> **Warning:** এই সব lifecycle hook **request-scoped class**-এর জন্য trigger হয় না। Request-scoped class application lifecycle-এর সাথে যুক্ত না — প্রতিটা request-এর জন্য আলাদা করে তৈরি হয় এবং response পাঠানোর পর automatic garbage-collect হয়ে যায়।

> **Hint:** `onModuleInit()` আর `onApplicationBootstrap()`-এর execution order সরাসরি নির্ভর করে module import করার order-এর উপর — আগেরটা শেষ না হলে পরেরটা শুরু হয় না (await করে)।

---

## Lifecycle Hook ব্যবহার করা

প্রতিটা lifecycle hook একটা interface দিয়ে represent হয়। TypeScript compile হওয়ার পর interface আসলে থাকে না, তাই টেকনিক্যালি এগুলো optional — কিন্তু strong typing আর editor tooling-এর সুবিধার জন্য এগুলো ব্যবহার করাই ভালো practice।

Hook register করতে হলে, সংশ্লিষ্ট interface implement করতে হয়। যেমন — module initialization-এর সময় কোনো method call করাতে চাইলে (Controller/Provider/Module-এ), `OnModuleInit` interface implement করে `onModuleInit()` method দিতে হবে:

```ts
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
```

---

## Asynchronous Initialization

`OnModuleInit` আর `OnApplicationBootstrap` — দুইটা hook-ই application initialization process কে delay করতে দেয়। এর জন্য method-টা `async` বানিয়ে ভিতরে কোনো asynchronous কাজ `await` করা যায় (অথবা সরাসরি একটা Promise return করা যায়):

```ts
async onModuleInit(): Promise<void> {
  await this.fetch();
}
```

---

## Application Shutdown

`onModuleDestroy()`, `beforeApplicationShutdown()`, আর `onApplicationShutdown()` — এই তিনটা hook **terminating phase**-এ call হয় (হয় `app.close()` explicitly call করলে, নাহলে opt-in থাকলে `SIGTERM`-এর মতো system signal পেলে)। এই feature সাধারণত Kubernetes দিয়ে container lifecycle manage করতে, বা Heroku-এর মতো service-এ dyno manage করতে ব্যবহার হয়।

### কেন Enable করতে হয়

Shutdown hook listener-গুলো system resource ব্যবহার করে, তাই এগুলো **by default বন্ধ থাকে**। ব্যবহার করতে হলে `enableShutdownHooks()` call করে listener চালু করতে হবে:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Starts listening for shutdown hooks
  app.enableShutdownHooks();

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> **Warning (Windows নিয়ে):** Platform-এর নিজস্ব সীমাবদ্ধতার কারণে Windows-এ shutdown hook-এর support সীমিত। `SIGINT` কাজ করবে, `SIGBREAK`-ও কিছুটা কাজ করবে, `SIGHUP`-ও আংশিকভাবে। কিন্তু **`SIGTERM` Windows-এ কখনোই কাজ করবে না** — কারণ Task Manager থেকে process kill করা unconditional, application-এর সেটা detect বা আটকানোর কোনো উপায় নেই।

> **Info:** `enableShutdownHooks` listener চালু করে memory ব্যবহার করে। একই Node process-এ একাধিক Nest app চললে (যেমন Jest দিয়ে parallel test চালানোর সময়), Node "excessive listener" নিয়ে warning দিতে পারে। এই কারণেই এটা by default disabled রাখা হয়েছে — একই process-এ একাধিক instance চালানোর সময় এই বিষয়টা মাথায় রাখতে হবে।

### Signal পাওয়ার পর কী হয়

Application termination signal পেলে, নিচের ক্রম অনুযায়ী registered method-গুলো call হয়:

1. `onModuleDestroy()`
2. `beforeApplicationShutdown()`
3. `onApplicationShutdown()`

প্রতিটাতেই signal-টা প্রথম parameter হিসেবে পাঠানো হয়। কোনো registered function যদি asynchronous call `await` করে (Promise return করে), Nest সেই Promise resolve/reject না হওয়া পর্যন্ত sequence-এর পরের ধাপে যাবে না।

```ts
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // e.g. "SIGINT"
  }
}
```

> **Info:** `app.close()` call করলে Node process নিজে থেকে বন্ধ হয়ে যায় না — এটা শুধু `onModuleDestroy()` আর `onApplicationShutdown()` hook trigger করে। তাই যদি কোনো interval, long-running background task ইত্যাদি চলমান থাকে, তাহলে process automatic terminate হবে না।

---
