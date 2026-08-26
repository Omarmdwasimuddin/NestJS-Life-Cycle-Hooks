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
