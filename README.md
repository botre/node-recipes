# Node recipes

## Directories

```bash
mkdir scripts
mkdir src
```

## execute.sh

```
#!/bin/bash

set -e

DIRECTORIES=("." "backend" "frontend/dashboard" "frontend/landing")

if [ $# -eq 0 ]; then
  echo Enter a command to execute:
  read -r COMMAND
else
  COMMAND=$*
fi

for i in "${DIRECTORIES[@]}"; do
  echo "Executing '$COMMAND' in $i"
  (cd "$i" && eval "$COMMAND")
done
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
    "jsx": "react",
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

application.use(
  helmet({
    contentSecurityPolicy:
      process.env.NODE_ENV === "production" ? undefined : false,
  })
);

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

router
  .get("/", (ctx) => (ctx.body = "🌊"))
  .get("/env", (ctx) => (ctx.body = process.env.NODE_ENV))
  .get("/ping", (ctx) => (ctx.body = "OK"));

application.use(router.routes()).use(router.allowedMethods());
```

```typescript
const port = 3000;

application.listen(port, () => {
  console.log(`Application listening at http://localhost:${port}`);
});
```

### Wrapping a Koa application for Lambda

```bash
npm install --save-dev aws-lambda serverless-http
```

```typescript
import { APIGatewayProxyEvent, Context } from "aws-lambda";
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
#!/bin/bash

set -e

node_modules/.bin/prettier --write . --ignore-path .gitignore
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

## ESLint

### Setup

```bash
npm install --save-dev @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint eslint-plugin-jsx-a11y eslint-plugin-no-only-tests eslint-plugin-react typescript
```

.eslintrc.js

```javascript
module.exports = {
  env: {
    browser: true,
    es2020: true,
    node: true,
  },
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaFeatures: {
      jsx: true,
    },
    ecmaVersion: 11,
    sourceType: "module",
  },
  plugins: ["@typescript-eslint", "jsx-a11y", "no-only-tests", "react"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:jsx-a11y/recommended",
    "plugin:react/recommended",
  ],
  rules: {
    "@typescript-eslint/explicit-module-boundary-types": "off",
    "@typescript-eslint/ban-ts-comment": "off",
    "@typescript-eslint/no-empty-function": "off",
    "@typescript-eslint/no-explicit-any": "off",
    "@typescript-eslint/no-non-null-assertion": "off",
    "@typescript-eslint/prefer-ts-expect-error": "error",
    "jsx-a11y/anchor-is-valid": "off",
    "no-console": "warn",
    "no-only-tests/no-only-tests": [
      "error",
      {
        block: ["assert", "context", "it", "test"],
        focus: ["focus", "only"],
      },
    ],
    "react/display-name": "off",
    "react/no-unescaped-entities": "off",
    "react/prop-types": "off",
    "react/react-in-jsx-scope": "off",
  },
  settings: {
    react: {
      version: "17.0.0",
    },
  },
};
```

lint.sh

```bash
#!/bin/bash

set -e

node_modules/.bin/eslint --fix --ignore-path .gitignore "**/*.+(js|jsx|ts|tsx)"
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
  "watch": ["src"],
  "ext": "js,jsx,ts,tsx,json",
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

## React

```bash
npm install react react-dom
```

```bash
npm install --save-dev @types/react @types/react-dom
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
#!/bin/bash

docker-compose down
docker-compose up -d
```

## TypeORM

### Dependencies

```bash
npm install typeorm
```

#### PostgreSQL

```bash
npm install typeorm-naming-strategies
```

#### SQLite

```bash
npm install sqlite3
```

### ormconfig.ts

#### PostgreSQL

```typescript
require("dotenv").config({ path: `.env.${process.env.NODE_ENV}` });

import { SnakeNamingStrategy } from "typeorm-naming-strategies";
import entities from "./src/database/entities";

export = {
  cli: {
    migrationsDir: "./src/database/migrations",
  },
  database: process.env.DB_DATABASE,
  entities: entities,
  host: process.env.DB_HOST,
  migrations: ["./src/database/migrations/*.ts"],
  namingStrategy: new SnakeNamingStrategy(),
  password: process.env.DB_PASSWORD,
  port: process.env.DB_PORT,
  type: process.env.DB_TYPE,
  username: process.env.DB_USERNAME,
};
```

#### SQLite

```typescript
import entities from "./src/database/entities";

export = {
  cli: {
    migrationsDir: "./src/database/migrations",
  },
  database: `database.sqlite`,
  entities: entities,
  migrations: ["./src/database/migrations/*.ts"],
  type: "sqlite",
};
```

### DatabaseService.ts

```typescript
import { Singleton } from "typescript-ioc";
import {
  Connection,
  createConnection,
  getConnection,
  getConnectionManager,
  getManager,
} from "typeorm";
import { SnakeNamingStrategy } from "typeorm-naming-strategies";
import entities from "../database/entities";

@Singleton
export default class DatabaseService {
  private _connection: Connection;

  async createConnection(overrides?: { database?: string }) {
    try {
      const options = {
        database: overrides?.database ?? process.env.DB_DATABASE,
        entities,
        host: process.env.DB_HOST,
        migrations: ["../migrations/*.ts"],
        namingStrategy: new SnakeNamingStrategy(),
        password: process.env.DB_PASSWORD,
        port: parseInt(process.env.DB_PORT!),
        type: process.env.DB_TYPE as any,
        username: process.env.DB_USERNAME,
      };
      const connection = await createConnection(options);
      if (process.env.DB_SYNCHRONIZE === "true") {
        await connection.synchronize(process.env.DB_DROP === "true");
      }
      this._connection = connection;
    } catch (e) {
      if (e.name === "AlreadyHasActiveConnectionError") {
        const existingConnection = getConnectionManager().get("default");
        this._connection = existingConnection;
      } else {
        throw e;
      }
    }
    return this._connection;
  }

  async closeConnection() {
    await getConnection().close();
  }

  async flushDatabase() {
    if (process.env.NODE_ENV !== "test") {
      throw new Error("Illegal NODE_ENV");
    }
    const entityNames = getConnection()
      .entityMetadatas.map(({ tableName }) => tableName)
      .filter((tableName) => tableName !== "migrations");
    await getManager().query(
      `TRUNCATE TABLE ${entityNames
        .map((name) => `"${name}"`)
        .join(", ")} CASCADE;`
    );
  }
}
```

### generate-migration.sh

```bash
#!/bin/bash

set -e

if [ $# -eq 0 ]; then
  echo Enter a name for your migration:
  read -r NAME
else
  NAME=$1
fi

NODE_ENV="production" ./node_modules/.bin/ts-node ./node_modules/typeorm/cli.js migration:generate -n "$NAME"
```

### run-migration.sh

```bash
#!/bin/bash

set -e

NODE_ENV="production" ./node_modules/.bin/ts-node ./node_modules/typeorm/cli.js migration:run
```

## GraphQL

```bash
npm install graphql graphql-type-json
```

### Apollo with Koa

```bash
npm install apollo-server apollo-server-koa
```

```typescript
import { ApolloServer } from "apollo-server-koa";

const apolloServer = new ApolloServer({
  typeDefs: [],
  resolvers: [],
});

apolloServer.applyMiddleware({ app: application, path: "/graphql" });
```

### TypeGraphQL

```bash
npm install type-graphql
```

```typescript
import { buildTypeDefsAndResolvers, registerEnumType } from "type-graphql";
import enums from "../gql/enums";
import resolvers from "../gql/resolvers";

const typeDefsAndResolvers = await (async () => {
  Object.entries(enums).forEach(([name, object]) =>
    registerEnumType(object, {
      name,
    })
  );
  return await buildTypeDefsAndResolvers({
    resolvers,
  });
})();
```

resolvers.ts

```typescript
import { NonEmptyArray } from "type-graphql";
import { SomeResolver } from "./resolvers/SomeResolver";

export default [SomeResolver] as NonEmptyArray<Function>;
```

#### Schema generation

buildAndEmitSchema.ts

```typescript
import "reflect-metadata";

import path from "path";

import {
  buildSchema,
  emitSchemaDefinitionFile,
  registerEnumType,
} from "type-graphql";

import enums from "./enums";
import resolvers from "./resolvers";

Object.entries(enums).forEach(([name, object]) =>
  registerEnumType(object, {
    name,
  })
);

buildSchema({
  resolvers,
}).then((schema) => {
  emitSchemaDefinitionFile(
    path.resolve(__dirname, "../../schema.graphql"),
    schema
  )
    .then(() => process.exit(0))
    .catch((error) => {
      console.log(error);
    });
});
```

```bash
#!/bin/bash

NODE_ENV="production" node_modules/.bin/ts-node src/gql/buildAndEmitSchema.ts
```

.graphqlconfig

```json
{
  "name": "schema",
  "schemaPath": "schema.graphql"
}
```

### Depth limit

```bash
npm install graphql-depth-limit
```

```bash
npm install --save-dev @types/graphql-depth-limit
```

```typescript
import { ApolloServer } from "apollo-server-koa";
import depthLimit from "graphql-depth-limit";

const DepthLimitRule = depthLimit(6);

const apolloServer = new ApolloServer({
  validationRules: [DepthLimitRule],
});
```

## IoC

### typescript-ioc

```bash
npm install typescript-ioc
```

## Stripe

### Subscription boilerplate

```bash
npm install stripe
```

```typescript
import Stripe from "stripe";
import { findUserForId, updateUser } from "./database";

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
  });
```

```typescript
router.post("/webhooks/stripe", async (ctx) => {
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
});
```

## Jest

```bash
mkdir __tests__
```

```bash
npm install --save-dev @types/jest jest ts-jest
```

jest.config.ts

```javascript
export default {
  globals: {
    "ts-jest": {
      isolatedModules: true,
    },
  },
  preset: "ts-jest",
  verbose: true,
};
```

tsconfig.json

```json
{
  "include": ["src/**/*", "__tests__/**/*"]
}
```

### Global setup

globalSetup.ts

```typescript
const globalSetup = () => {
  if (process.env.NODE_ENV !== "test") {
    throw new Error("Illegal NODE_ENV");
  }
  require("dotenv").config({ path: `.env.${process.env.NODE_ENV}` });
};

export default globalSetup;
```

jest.config.ts

```typescript
{
  globalSetup: "./src/testing/globalSetup.ts";
}
```

### initializeBackendBeforeAll

```typescript
import { Container } from "typescript-ioc";
import http from "http";
import supertest from "supertest";
import { v4 as uuid } from "uuid";
import DatabaseService from "../services/DatabaseService";
import KoaServerService from "../services/KoaServerService";
import ApolloServerService from "../services/ApolloServerService";

export const initializeBackendBeforeAll = ({
  resetDatabaseBeforeEach: resetDatabaseBeforeEach = true,
} = {}) => {
  const $database = Container.get(DatabaseService);
  const $koaServer = Container.get(KoaServerService);
  const $apolloServer = Container.get(ApolloServerService);

  let request: supertest.SuperTest<supertest.Test>;

  beforeAll(async () => {
    const database = `db${uuid().replace(/-/g, "").slice(0, 32)}`;
    const connection = await $database.createConnection();
    await connection.query(`CREATE DATABASE ${database}`);
    await $database.closeConnection();
    await $database.createConnection({
      database,
    });
    const server = await $koaServer.create();
    await $apolloServer.create(server);
    request = supertest(http.createServer(server.callback()));
  });

  afterAll(async () => {
    await $database.closeConnection();
  });

  if (resetDatabaseBeforeEach) {
    beforeEach(async () => {
      await $database.flushDatabase();
    });
  }

  return () => ({ request });
};
```

### Webpack for AWS Lambda

```bash
npm install --save-dev @types/webpack ts-loader webpack webpack-cli
```

build.sh

```bash
#!/bin/bash

set -e

export TS_NODE_PROJECT="webpack.tsconfig.json"
export NODE_ENV="production"

node_modules/.bin/webpack
```

webpack.config.ts

```typescript
/* eslint-disable @typescript-eslint/no-var-requires */

import * as path from "path";
import * as webpack from "webpack";

require("dotenv").config({
  path: path.join(__dirname, `.env.${process.env.NODE_ENV}`),
});

const config: webpack.Configuration = {
  entry: {
    api: path.join(__dirname, "src", "serverless"),
  },
  output: {
    path: `${process.cwd()}/dist`,
    filename: "[name].js",
    libraryTarget: "umd",
  },
  target: "node",
  module: {
    rules: [
      {
        test: /\.m?js/,
        resolve: {
          fullySpecified: false,
        },
      },
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        use: {
          loader: "ts-loader",
        },
      },
    ],
  },
  plugins: [
    new webpack.EnvironmentPlugin(Object.keys(process.env)),
    new webpack.IgnorePlugin({
      resourceRegExp: /^pg-native$/,
    }),
  ],
  resolve: {
    extensions: [".ts", ".tsx", ".mjs", ".js", ".jsx"],
  },
  optimization: { minimize: false },
  externals: [{ "aws-sdk": "commonjs aws-sdk" }],
  devtool: "source-map",
};

export default config;
```

webpack.tsconfig.json

```json
{
  "compilerOptions": {
    "esModuleInterop": true,
    "module": "commonjs",
    "target": "es5"
  }
}
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
    "lib": ["dom", "ES2015"],
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
  "include": ["src"]
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
    rules: [{ test: /\.ts?$/, loader: "ts-loader", exclude: /node_modules/ }],
  },
};
```

package.json

```json
{
  "private": false,
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist", "src"],
  "engines": {
    "node": ">=10"
  },
  "scripts": {
    "build": "webpack --mode production --progress"
  }
}
```
