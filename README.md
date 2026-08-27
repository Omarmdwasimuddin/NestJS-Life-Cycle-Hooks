# Life Cycle Hooks

### Create service & controller
```bash
nest g service database
```
```bash
nest g controller database
```
---


> Note: life cycle method start hoi on diye jemon onModuleInit, onApplicationShutdown etc.
### `database.service.ts`
```bash
import { Injectable, OnModuleInit, OnApplicationShutdown, } from '@nestjs/common';

@Injectable()
export class DatabaseService
  implements OnModuleInit, OnApplicationShutdown
{
  private isConnected = false;

  onModuleInit() {
    this.isConnected = true;
    console.log('Database Connected!');
  }

  onApplicationShutdown(signal: string) {
    this.isConnected = false;
    console.log(`Database Disconnected! signal: ${signal}`);
  }

  getStatus() {
    return this.isConnected ? 'Connected' : 'Disconnected';
  }
}
```
---


>Note: main.ts e add koro `app.enableShutdownHooks();`
### `main.ts`
```bash
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableShutdownHooks();
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```
---


### `database.controller.ts`
```bash
import { Controller, Get } from '@nestjs/common';
import { DatabaseService } from './database.service';

@Controller('database')
export class DatabaseController {
    constructor(private readonly databaseService: DatabaseService) {}

    @Get('status')
    getStatus() {
        return this.databaseService.getStatus();
    }

}
```
---



> run koro
> ```bash
>npm run start:dev
> ```
> <img width="830" height="276" alt="image" src="https://github.com/user-attachments/assets/bfbb2958-a73d-4b6f-b6b8-3cf0cea2d6f2" />

>```bash
>Ctrl + c
>```
><img width="787" height="293" alt="image" src="https://github.com/user-attachments/assets/3b895edc-9064-443d-827b-5994e32dc2c4" />

---
