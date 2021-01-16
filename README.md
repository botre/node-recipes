# Node recipes

## Directories

```bash
mkdir src
```

## TypeScript

### Setup

```bash
npm install --save-dev @types/node typescript
```

tsconfig.json

```json
{
  "compilerOptions": {
    "emitDecoratorMetadata": true,
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "lib": ["es2019", "esnext.asynciterable"],
    "module": "commonjs",
    "moduleResolution": "node",
    "noImplicitAny": true,
    "noImplicitThis": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "sourceMap": true,
    "strictBindCallApply": true,
    "strictNullChecks": true,
    "target": "es2019"
  },
  "include": ["src/**/*"]
}
```

## Koa

### Setup

```bash
npm install @koa/cors @koa/router koa koa-bodyparser koa-helmet koa-logger
```

```bash
npm install --save-dev @types/koa @types/koa-bodyparser @types/koa-helmet @types/koa-logger @types/koa__cors @types/koa__router
```

application.ts

```typescript
import Koa from "koa";
import logger from "koa-logger";
import cors from "@koa/cors";
import helmet from "koa-helmet";
import bodyParser from "koa-bodyparser";
import Router from "@koa/router";

const application = new Koa();

application.use(logger());

application.use(cors());

application.use(helmet());

application.use(bodyParser());

application.use(async (ctx, next) => {
    try {
        await next();
    } catch (error) {
        console.error(error);
        ctx.status = error.status || 500;
        ctx.body = error.message;
    }
});

const router = new Router();

router.get("/", (ctx) => (ctx.body = "🌊"));

application.use(router.routes()).use(router.allowedMethods());
```

## Nodemon

### Setup

```bash
npm install --save-dev nodemon ts-node
```

nodemon.json

```json
{
  "watch": ["src"],
  "ext": "js,json,ts",
  "exec": "ts-node --transpile-only -r ./src/application.ts"
}
```