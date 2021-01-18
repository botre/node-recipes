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
  "watch": [
    "src"
  ],
  "ext": "js,json,ts",
  "exec": "ts-node --transpile-only -r ./src/application.ts"
}
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
