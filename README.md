# Node recipes

## Directories

```bash
mkdir scripts
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
    "lib": [
      "es2019",
      "esnext.asynciterable"
    ],
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
  "include": [
    "src/**/*"
  ]
}
```

typecheck.sh

```bash
#!/bin/bash

set -e

node_modules/.bin/tsc --noEmit -p tsconfig.json
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

### Wrapping a Koa application for Lambda

```bash
npm install --save-dev aws-lambda serverless-http
```

```typescript
import {APIGatewayProxyEvent, Context} from "aws-lambda";
import serverless from "serverless-http";
import createApplication from "./application";

export const handler = async (
    event: APIGatewayProxyEvent,
    context: Context
) => {
    const application = await createApplication();
    const handler = serverless(application);
    const result = await handler(event, context);
    return result;
};
```

## Prettier

### Format all

```bash
prettier --write . --ignore-path .gitignore
```

### Quick setup

```bash
npm install --save-dev husky prettier pretty-quick
```

package.json

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "pretty-quick --staged"
    }
  }
}
```

## Nodemon

### Setup

```bash
npm install --save-dev nodemon ts-node
```

dev.sh

```bash
#!/bin/bash

set -e

export NODE_ENV="development"

node_modules/.bin/nodemon
```

nodemon.json

```json
{
  "watch": [
    "src"
  ],
  "ext": "js,json,ts",
  "exec": "ts-node --transpile-only -r ./src/application.ts"
}
```

## .env

```bash
npm install dotenv
```

```bash
require("dotenv").config({ path: `.env.${process.env.NODE_ENV}` });
```

## Securing .env files

```bash
npm install --save-dev senv
```

Add the following to your .bashrc

```bash
export DOTENV_SECRET=your-secret
```

encrypt.sh

```bash
#!/bin/bash

set -e

node_modules/.bin/senv encrypt .env.development -p "$DOTENV_SECRET" > .env.development.encrypted
node_modules/.bin/senv encrypt .env.production -p "$DOTENV_SECRET" > .env.production.encrypted
node_modules/.bin/senv encrypt .env.test -p "$DOTENV_SECRET" > .env.test.encrypted
```

decrypt.sh

```bash
#!/bin/bash

set -e

node_modules/.bin/senv decrypt .env.development.encrypted -p "$DOTENV_SECRET" > .env.development
node_modules/.bin/senv decrypt .env.production.encrypted -p "$DOTENV_SECRET" > .env.production
node_modules/.bin/senv decrypt .env.test.encrypted -p "$DOTENV_SECRET" > .env.test
```

## Database

### Setup

.env

```
DB_DATABASE=postgres
DB_HOST=localhost
DB_PASSWORD=postgres
DB_PORT=5432
DB_TYPE=postgres
```

docker-compose.yml

```yaml
version: "3"
services:
  postgres:
    image: "postgres"
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    ports:
      - 5432:5432
```

dev.sh

```bash
docker-compose down
docker-compose up -d
```

## TypeORM

### Setup

```bash
npm install pg typeorm typeorm-naming-strategies
```

ormconfig.ts

```typescript
require("dotenv").config({path: `.env.${process.env.NODE_ENV}`});

import {SnakeNamingStrategy} from "typeorm-naming-strategies";
import entities from "./src/entities";

export = {
    cli: {
        migrationsDir: "./src/migrations",
    },
    database: env.DB_DATABASE,
    entities: entities,
    host: env.DB_HOST,
    migrations: ["./src/migrations/*.ts"],
    namingStrategy: new SnakeNamingStrategy(),
    password: env.DB_PASSWORD,
    port: env.DB_PORT,
    type: env.DB_TYPE,
    username: env.DB_USERNAME,
};
```

database.ts

```typescript
import {
    createConnection,
    getConnection,
    getConnectionManager,
    getManager,
} from "typeorm";
import {SnakeNamingStrategy} from "typeorm-naming-strategies";
import entities from "../entities";

let _connection;

export const createDatabaseConnection = async () => {
    try {
        const connection = await createConnection({
            database: process.env.DB_DATABASE,
            entities,
            host: process.env.DB_HOST,
            migrations: ["../migrations/*.ts"],
            namingStrategy: new SnakeNamingStrategy(),
            password: process.env.DB_PASSWORD,
            port: parseInt(process.env.DB_PORT!),
            type: process.env.DB_TYPE as any,
            username: process.env.DB_USERNAME,
        });
        if (process.env.DB_SYNCHRONIZE) {
            await connection.synchronize(false);
        }
        _connection = connection;
    } catch (e) {
        if (e.name === "AlreadyHasActiveConnectionError") {
            const existingConnection = getConnectionManager().get("default");
            _connection = existingConnection;
        } else {
            throw e;
        }
    }
};

export const closeDatabaseConnection = async () => {
    await getConnection().close();
};

export const flushDatabase = async () => {
    if (process.env.NODE_ENV !== "test") {
        throw new Error("Illegal NODE_ENV");
    }
    const entityNames = getConnection()
        .entityMetadatas.map(({tableName}) => tableName)
        .filter((tableName) => tableName !== "migrations");
    await getManager().query(
        `TRUNCATE TABLE ${entityNames
            .map((name) => `"${name}"`)
            .join(", ")} CASCADE;`
    );
};
```

generate-migration.sh

```bash
set -e

echo You are about to generate a migration from schema changes
echo Pick a name:
read -r NAME
NODE_ENV="production" ./node_modules/.bin/ts-node ./node_modules/typeorm/cli.js migration:generate -n "$NAME"
```

run-migration.sh

```bash
set -e

NODE_ENV="production" ./node_modules/.bin/ts-node ./node_modules/typeorm/cli.js migration:run
```

## Stripe

### Subscription boilerplate

```bash
npm install stripe
```

```typescript
import Stripe from "stripe";
import {findUserForId, updateUser} from "./database";

const STRIPE_SECRET_KEY = "TODO";
const STRIPE_SUBSCRIPTION_PRICE_ID = "TODO";
const STRIPE_WEBSITE_URL = "TODO";

export const stripe = new Stripe(STRIPE_SECRET_KEY, {
    apiVersion: "2020-08-27",
});

export const createCustomer = async (userId: string) => {
    const user = await findUserForId(userId);
    if (user) {
        if (user.stripeCustomerId) {
            return user.stripeCustomerId;
        } else {
            const customer = await stripe.customers.create({
                email: user.email,
            });
            await updateUser(user.id, {
                stripeCustomerId: customer.id,
            });
            return customer.id;
        }
    }
    return null;
};

export const createCheckoutSession = async (userId: string) => {
    const user = await findUserForId(userId);
    if (!user) {
        throw new Error("Missing user");
    }
    let customerId = user.stripeCustomerId;
    if (!customerId) {
        customerId = await createCustomer(userId);
    }
    if (!customerId) {
        throw new Error("Error creating Stripe Customer");
    }
    const session = await stripe.checkout.sessions.create({
        customer: customerId,
        mode: "subscription",
        payment_method_types: ["card"],
        line_items: [
            {
                price: STRIPE_SUBSCRIPTION_PRICE_ID,
                quantity: 1,
            },
        ],
        success_url: STRIPE_WEBSITE_URL,
        cancel_url: STRIPE_WEBSITE_URL,
    });
    return session;
};

export const createBillingPortalSession = async (userId: string) => {
    const user = await findUserForId(userId);
    if (!user) {
        throw new Error("Missing user");
    }
    if (!user.stripeCustomerId) {
        throw new Error("Missing Stripe Customer");
    }
    const session = await stripe.billingPortal.sessions.create({
        customer: user.stripeCustomerId,
        return_url: STRIPE_WEBSITE_URL,
    });
    return session;
};
```

```typescript
router
    .get("/checkout-session", async (ctx) => {
        const checkoutSession = await createCheckoutSession("user-id");
        ctx.body = checkoutSession.id;
    })
    .get("/billing-portal", async (ctx) => {
        const billingPortalSession = await createBillingPortalSession("user-id");
        ctx.redirect(billingPortalSession.url);
    })
```

```typescript
router
    .post("/webhooks/stripe", async (ctx) => {
        const event = stripe.webhooks.constructEvent(
            ctx.request.rawBody,
            ctx.request.headers["stripe-signature"],
            "STRIPE_SIGNING_SECRET" // TODO
        );

        console.log(`Event type: ${event.type}`);

        if (event.type === "checkout.session.completed") {
            console.log("Stripe: checkout session completed");
            const session = event.data.object as Stripe.Checkout.Session;
            const customerId = session.customer;
            if (!customerId) {
                throw new Error(`Missing customer ${customerId}`);
            }
            const user = await findUserForStripeCustomerId(customerId as string);
            if (!user) {
                throw new Error("Missing user");
            }
            await updateUser(user.id, {
                subscribedAt: new Date(),
            });
        }

        if (event.type === "customer.subscription.deleted") {
            console.log("Stripe: customer subscription deleted");
            const subscription = event.data.object as Stripe.Subscription;
            const customerId = subscription.customer;
            if (!customerId) {
                throw new Error(`Missing customer ${customerId}`);
            }
            const user = await findUserForStripeCustomerId(customerId as string);
            if (!user) {
                throw new Error("Missing user");
            }
            await updateUser(user.id, {
                unsubscribedAt: new Date(),
            });
        }

        ctx.status = 200;
    })
```

## NPM browser library (Webpack)

```bash
npm install --save-dev ts-loader tslib typescript webpack webpack-cli
```

tsconfig.json

```json
{
  "compilerOptions": {
    "declaration": true,
    "esModuleInterop": true,
    "importHelpers": true,
    "lib": [
      "dom",
      "ES2015"
    ],
    "module": "CommonJS",
    "moduleResolution": "node",
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "outDir": "dist",
    "resolveJsonModule": true,
    "rootDir": "src",
    "sourceMap": true,
    "strict": true,
    "target": "ES5"
  },
  "include": [
    "src"
  ]
}
```

webpack.config.js

```javascript
module.exports = {
    devtool: "inline-source-map",
    output: {
        filename: "index.js",
        library: "LibraryName",
        libraryTarget: "umd",
        libraryExport: "default",
        umdNamedDefine: true,
        globalObject: "typeof self !== 'undefined' ? self : this",
    },
    resolve: {
        extensions: [".ts", ".js"],
    },
    module: {
        rules: [{test: /\.ts?$/, loader: "ts-loader", exclude: /node_modules/}],
    },
};
```

package.json

```json
{
  "private": false,
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist",
    "src"
  ],
  "engines": {
    "node": ">=10"
  },
  "scripts": {
    "build": "webpack --mode production --progress"
  }
}
```