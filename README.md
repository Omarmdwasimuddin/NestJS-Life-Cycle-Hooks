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


>Note: main.ts e add koro app.enableShutdownHooks();
### `main.ts`
```bash
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
  }));

  await app.listen(process.env.PORT ?? 3000);
  app.enableShutdownHooks();
}
bootstrap();
```
---
