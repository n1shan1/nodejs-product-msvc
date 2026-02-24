# Repository Comprehensive Summary

## 1) High-level purpose
This repository is a small Node.js microservices example for an e-commerce-style flow. It is composed of three services (auth, product, order), plus MongoDB and RabbitMQ orchestrated via Docker Compose.

At a system level:
- `auth-service` handles user registration/login and returns JWTs.
- `product-service` creates products and publishes order requests to RabbitMQ.
- `order-service` consumes order requests, calculates totals, saves orders, and emits a completion message.

## 2) Runtime architecture and orchestration
### `docker-compose.yml`
Defines five containers:
- `mongodb` (`mongo:latest`, port `27017`)
- `rabbitmq` (`rabbitmq:3-management`, ports `5672`, `15672`, guest/guest credentials)
- `auth-service` on `7020`
- `product-service` on `8080`
- `order-service` on `9090`

Each app service is built from its own folder and receives environment variables for Mongo URI and/or RabbitMQ URI.

### Service images (`*/Dockerfile`)
All three services use a very similar Dockerfile pattern:
1. `FROM node:14`
2. copy package manifest(s)
3. `npm install`
4. copy source
5. expose service port
6. run `node index.js`

## 3) Root-level files
### `README.md`
A short landing page that states the architecture topic and links to a diagram image.

### `MicroservicesProjectGuide.md`
A long-form educational guide that documents:
- microservices concepts
- architecture overview
- API examples for auth/product endpoints
- RabbitMQ message flow
- setup and troubleshooting steps

This file acts as tutorial documentation rather than executable code.

### `package.json` (root)
Root package is named `microservice` and currently depends only on `jsonwebtoken`. It has no meaningful scripts beyond a placeholder failing `test` script.

### `authenticator.js` (root)
Contains a reusable JWT middleware (`isAuthenticated`) that:
- reads bearer token from `authorization` header
- verifies with hardcoded secret `this_is_secret`
- puts decoded payload in `req.user` if valid

Equivalent copies of this middleware also exist in each service directory.

## 4) Auth service (`auth-service/`)
### `auth-service/index.js`
Main behaviors:
- starts Express server on port `7020` (or `PORT` env)
- connects Mongoose to `mongodb://localhost/auth-service` fallback
- uses JSON body parsing
- `POST /auth/register`
  - expects `email`, `password`, `name`
  - prevents duplicate email (returns HTTP 402 with message)
  - creates/saves user and returns created object
- `POST /auth/login`
  - finds user by email
  - compares raw password directly (no hashing)
  - signs JWT with payload `{ email, name }` and same hardcoded secret

Notes:
- error messages contain typos (e.g., `User doesn't exits`, `Passwod icorrect`).
- password storage/comparison is plaintext.

### `auth-service/models/user.js`
Mongoose schema:
- `name: String`
- `email: String`
- `password: String`
- `created_at: Date` with default `Date.now()`

Model name is `user`.

### `auth-service/authenticator.js`
JWT middleware copy matching root behavior.

### `auth-service/package.json`
Dependencies:
- `express`
- `mongoose`
- `jsonwebtoken`
- `amqplib` (not used by auth runtime code)

## 5) Product service (`product-service/`)
### `product-service/index.js`
Main behaviors:
- starts Express server on port `8080` (or `PORT` env)
- connects Mongoose to `mongodb://localhost/product-service` fallback
- creates RabbitMQ connection/channel and asserts queue `PRODUCT`
- protected test route `POST /test`
- protected `POST /product/create`
  - should create and save a new product from `name`, `description`, `price`
- protected `POST /product/buy`
  - receives product `ids`
  - finds matching products from Mongo
  - publishes message to `ORDER` queue with `{ products, userEmail }`
  - returns `Order is being processed`

Important implementation observations:
- file imports `const product = require("./models/product");` but later uses `new Product(...)` and `Product.find(...)`.
  - `Product` is not defined in this file as written, so these routes will throw at runtime unless corrected.
- RabbitMQ channel asserts `PRODUCT` queue but this service only explicitly sends to `ORDER`; likely intended for receiving completion messages but no consumer is implemented.

### `product-service/models/product.js`
Mongoose schema:
- `name: String`
- `description: String`
- `price: Number`
- `created_at: Date` default now

Model name is `product`.

### `product-service/authenticator.js`
JWT middleware copy matching root/auth-service behavior.

### `product-service/package.json`
Dependencies:
- `express`
- `mongoose`
- `jsonwebtoken`
- `amqplib`

## 6) Order service (`order-service/`)
### `order-service/index.js`
Main behaviors:
- starts Express server on port `9090` (or `PORT` env)
- connects Mongoose to `mongodb://localhost/order-service` fallback
- creates RabbitMQ connection/channel and asserts queue `ORDER`
- defines helper `createOrder(products, userEmail)`:
  - sums `price` across `products`
  - creates/saves order with product payload, user email, total price
- on startup, consumes `ORDER` queue:
  - parses message payload `{ products, userEmail }`
  - creates order
  - acknowledges message
  - publishes `{ newOrder }` to queue `PRODUCT`

Observations:
- imports `jsonwebtoken` and authenticator middleware but does not expose protected HTTP endpoints; these imports are currently unused.
- consumes from `ORDER`, aligning with product-service publisher behavior.

### `order-service/models/order.js`
Mongoose schema:
- `products`: array of objects with `product_id: String`
- `user: String`
- `total_price: Number`
- `created_at: Date` default now

Model name is `order`.

Schema mismatch note:
- service writes full product objects into `products`, while schema suggests objects containing only `product_id`. Mongoose may still store extra fields depending on strict mode and object shape, but the declared intent and write payload are not aligned.

### `order-service/authenticator.js`
JWT middleware copy matching other directories.

### `order-service/package.json`
Contains only a `dependencies` object with the same four libraries as other services; lacks standard metadata/scripts fields present in other package manifests.

## 7) Messaging/data flow summary
1. Client authenticates against `auth-service`, receives JWT.
2. Client calls `product-service` endpoints with bearer token.
3. On `/product/buy`, product-service queries products and sends an `ORDER` queue message.
4. order-service consumes `ORDER`, creates Mongo order document, emits `PRODUCT` queue message.
5. product-service currently does not consume `PRODUCT`, so completion is effectively fire-and-forget from user perspective.

## 8) Security and reliability characteristics (as implemented)
- JWT secret is hardcoded in source (`this_is_secret`) across multiple files.
- Passwords are stored unhashed.
- Minimal error handling around DB/message broker connections and async route operations.
- No centralized config validation.
- No automated tests in repository.

## 9) Duplicate/shared patterns
- JWT middleware is duplicated in 4 locations (root + each service).
- Service setup boilerplate is similar (Express + Mongoose + startup logging).

## 10) Practical status snapshot
This repository is best characterized as a learning/demo implementation:
- clear demonstration of service decomposition and RabbitMQ integration
- working compose scaffold for local experimentation
- several production-readiness gaps and at least one product-service runtime naming bug (`product` vs `Product`)

