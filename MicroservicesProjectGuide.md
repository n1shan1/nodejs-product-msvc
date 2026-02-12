# Node.js Microservices E-Commerce Application

## Abstract

This document provides a comprehensive guide to a Node.js-based microservices e-commerce application. The project demonstrates the implementation of a distributed system using three independent services: authentication, product management, and order processing. It incorporates asynchronous communication via RabbitMQ, data persistence with MongoDB, and containerization using Docker and Docker Compose. This guide is designed for beginners, including those new to microservices, RabbitMQ, Docker, and Docker Compose, offering step-by-step explanations and practical examples.

## Table of Contents

1. [Introduction](#introduction)
2. [System Architecture](#system-architecture)
3. [Services Overview](#services-overview)
   - [Authentication Service](#authentication-service)
   - [Product Service](#product-service)
   - [Order Service](#order-service)
4. [Technologies Used](#technologies-used)
5. [Setup and Installation](#setup-and-installation)
6. [API Documentation](#api-documentation)
7. [Message Flow](#message-flow)
8. [Code Examples](#code-examples)
9. [Database Schema](#database-schema)
10. [Conclusion and Further Reading](#conclusion-and-further-reading)

## Introduction

Microservices architecture involves breaking down a monolithic application into smaller, independent services that communicate with each other. This approach enhances scalability, maintainability, and fault isolation. In this project, we build a simple e-commerce system where users can register, log in, create products, and place orders.

Key concepts introduced:
- **Microservices**: Modular components that run independently but collaborate to fulfill business logic.
- **RabbitMQ**: A message broker that enables asynchronous communication between services, allowing them to send and receive messages without direct coupling.
- **Docker**: A platform for containerizing applications, ensuring they run consistently across different environments by packaging code, dependencies, and runtime into isolated containers.
- **Docker Compose**: A tool for defining and running multi-container Docker applications, simplifying the orchestration of interconnected services.

This project serves as an educational example, illustrating how these technologies work together in a real-world scenario.

## System Architecture

The application consists of three microservices, a database, and a message broker, all orchestrated via Docker Compose.

```
+----------------+     +----------------+     +----------------+
| Auth Service   |     | Product Service|     | Order Service  |
| (Port 7020)    |     | (Port 8080)    |     | (Port 9090)    |
+----------------+     +----------------+     +----------------+
         |                      |                      |
         +----------------------+----------------------+
                        |
                +----------------+
                |   RabbitMQ     |
                |   (Port 5672)  |
                +----------------+
                        |
                +----------------+
                |    MongoDB     |
                |   (Port 27017) |
                +----------------+
```

- **Services**: Each service is a separate Node.js application running in its own Docker container.
- **Communication**: Services use HTTP APIs for external interactions and RabbitMQ for internal messaging.
- **Data Storage**: MongoDB stores user, product, and order data in separate databases.
- **Containerization**: Docker ensures each component has its own environment, preventing conflicts.

## Services Overview

### Authentication Service

**Purpose**: Manages user registration and authentication, issuing JSON Web Tokens (JWT) for secure access to other services.

**Key Features**:
- User registration with email, password, and name.
- User login with JWT token generation.
- Password validation (note: in production, passwords should be hashed for security).

**Endpoints**:
- `POST /auth/register`: Creates a new user.
- `POST /auth/login`: Authenticates a user and returns a JWT token.

**Dependencies**: Express.js, Mongoose (for MongoDB), jsonwebtoken.

### Product Service

**Purpose**: Handles product creation and purchasing, integrating with the authentication service for security and the order service for processing.

**Key Features**:
- Authenticated product creation.
- Product purchasing, which triggers an asynchronous order via RabbitMQ.

**Endpoints**:
- `POST /product/create`: Adds a new product (requires JWT).
- `POST /product/buy`: Initiates a purchase (requires JWT).

**Dependencies**: Express.js, Mongoose, amqplib (for RabbitMQ), jsonwebtoken.

### Order Service

**Purpose**: Processes orders received via RabbitMQ, calculates totals, and saves order data.

**Key Features**:
- Listens for order messages from the Product Service.
- Computes order totals and persists data to MongoDB.
- Sends confirmation back via RabbitMQ.

**Endpoints**: None (operates via message queues).

**Dependencies**: Express.js, Mongoose, amqplib.

## Technologies Used

- **Node.js**: A JavaScript runtime for server-side development, enabling fast, scalable applications.
- **Express.js**: A web framework for building RESTful APIs in Node.js.
- **MongoDB**: A NoSQL database for flexible, document-based data storage.
- **RabbitMQ**: An open-source message broker that implements the Advanced Message Queuing Protocol (AMQP), facilitating decoupled communication.
- **Docker**: Containerization technology that packages applications and their dependencies into portable containers.
- **Docker Compose**: A tool for defining and managing multi-container applications using a YAML file.
- **JWT (JSON Web Tokens)**: A standard for securely transmitting information between parties as a JSON object.

## Setup and Installation

### Prerequisites

- Docker and Docker Compose installed on your system.
- Basic knowledge of command-line interfaces.

### Steps

1. **Clone or Navigate to the Project Directory**:
   ```
   cd /mnt/c/Developer/nodejs-product-msvc
   ```

2. **Build and Start the Application**:
   ```
   docker compose up -d --build
   ```
   - This command builds Docker images for each service and starts the containers in detached mode.
   - `--build` ensures images are rebuilt if code changes.
   - `-d` runs containers in the background.

3. **Verify Services**:
   ```
   docker compose ps
   ```
   - Check that all services (auth-service, product-service, order-service, mongodb, rabbitmq) are running.

4. **View Logs (if needed)**:
   ```
   docker compose logs <service-name>
   ```
   - Replace `<service-name>` with the service (e.g., `auth-service`).

5. **Stop the Application**:
   ```
   docker compose down
   ```

### Troubleshooting

- If a service fails to start, check logs for errors (e.g., connection issues with MongoDB or RabbitMQ).
- Ensure ports 7020, 8080, 9090, 27017, 5672, and 15672 are available.
- RabbitMQ management interface is accessible at `http://localhost:15672` (default credentials: guest/guest).

## API Documentation

All APIs expect JSON payloads and return JSON responses. Use tools like `curl`, Postman, or a web browser for testing.

### Authentication Service (localhost:7020)

#### Register User
- **Method**: POST
- **Endpoint**: `/auth/register`
- **Headers**: `Content-Type: application/json`
- **Body**:
  ```json
  {
    "email": "user@example.com",
    "password": "password123",
    "name": "User Name"
  }
  ```
- **Response** (Success):
  ```json
  {
    "_id": "user_id",
    "name": "User Name",
    "email": "user@example.com",
    "password": "password123",
    "created_at": "2026-02-12T10:00:00.000Z"
  }
  ```

#### Login User
- **Method**: POST
- **Endpoint**: `/auth/login`
- **Headers**: `Content-Type: application/json`
- **Body**:
  ```json
  {
    "email": "user@example.com",
    "password": "password123"
  }
  ```
- **Response** (Success):
  ```json
  {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
  ```

### Product Service (localhost:8080)

#### Create Product
- **Method**: POST
- **Endpoint**: `/product/create`
- **Headers**:
  - `Content-Type: application/json`
  - `authorization: Bearer <jwt_token>`
- **Body**:
  ```json
  {
    "name": "Sample Product",
    "description": "A description",
    "price": 29.99
  }
  ```
- **Response** (Success):
  ```json
  {
    "_id": "product_id",
    "name": "Sample Product",
    "description": "A description",
    "price": 29.99,
    "created_at": "2026-02-12T10:05:00.000Z"
  }
  ```

#### Buy Product
- **Method**: POST
- **Endpoint**: `/product/buy`
- **Headers**:
  - `Content-Type: application/json`
  - `authorization: Bearer <jwt_token>`
- **Body**:
  ```json
  {
    "ids": ["product_id"]
  }
  ```
- **Response** (Success):
  ```json
  {
    "message": "Order is being processed"
  }
  ```

## Message Flow

RabbitMQ enables asynchronous communication:

1. **Product Purchase**: Product Service sends a message to the "ORDER" queue with product details and user email.
2. **Order Processing**: Order Service consumes the message, calculates the total, saves the order to MongoDB, and sends a confirmation to the "PRODUCT" queue.
3. **Confirmation**: Product Service can consume the confirmation (though not implemented in the current version).

This decoupling ensures services remain independent and scalable.

## Code Examples

### Authentication Service: Login Endpoint
```javascript
app.post("/auth/login", async (req, res) => {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user || password !== user.password) {
        return res.json({ message: "Invalid credentials" });
    }
    const token = jwt.sign({ email, name: user.name }, "this_is_secret");
    res.json({ token });
});
```

### Product Service: Buy Endpoint
```javascript
app.post("/product/buy", isAuthenticated, async (req, res) => {
    const { ids } = req.body;
    const products = await Product.find({ _id: { $in: ids } });
    channel.sendToQueue("ORDER", Buffer.from(JSON.stringify({
        products,
        userEmail: req.user.email
    })));
    res.json({ message: "Order is being processed" });
});
```

### Order Service: Message Consumer
```javascript
connect().then(() => {
    channel.consume("ORDER", data => {
        const { products, userEmail } = JSON.parse(data.content);
        let total = 0;
        products.forEach(p => total += p.price);
        const newOrder = new Order({
            products,
            user: userEmail,
            total_price: total
        });
        newOrder.save();
        channel.ack(data);
        channel.sendToQueue("PRODUCT", Buffer.from(JSON.stringify({ newOrder })));
    });
});
```

## Database Schema

- **Users** (auth-service database):
  ```json
  {
    "name": "String",
    "email": "String",
    "password": "String",
    "created_at": "Date"
  }
  ```

- **Products** (product-service database):
  ```json
  {
    "name": "String",
    "description": "String",
    "price": "Number",
    "created_at": "Date"
  }
  ```

- **Orders** (order-service database):
  ```json
  {
    "products": ["ObjectId"],
    "user": "String",
    "total_price": "Number",
    "created_at": "Date"
  }
  ```

## Conclusion and Further Reading

This project illustrates the fundamentals of microservices, asynchronous messaging, and containerization. By understanding these concepts, you can build scalable, maintainable applications.

### Further Reading
- **Microservices**: "Building Microservices" by Sam Newman.
- **RabbitMQ**: Official documentation at rabbitmq.com.
- **Docker**: Docker's getting started guide at docs.docker.com.
- **Node.js**: nodejs.org for tutorials.

For questions or improvements, refer to the project code or consult online resources. This setup provides a solid foundation for expanding into more complex systems.