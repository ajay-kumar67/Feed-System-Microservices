# Linkiden-Microservices
Linkedin Clone 

# 🔗 LinkedIn System Design Clone

A distributed social networking platform built with **Spring Boot Microservices**, demonstrating **event-driven architecture**, **fan-out-on-write feed generation**, and **centralized JWT authentication** at scale.

> ⚠️ **Disclaimer:** This is an educational implementation inspired by system design walkthroughs. It prioritizes architectural correctness and scalability patterns over production hardening (e.g., no circuit breakers, service mesh, or IaC).

## 🏗️ Architecture

![Architecture Diagram](./docs/architecture.png) <!-- Replace with your actual diagram -->

The system is decomposed into **6 independent microservices** communicating asynchronously via **Apache Kafka** to ensure loose coupling and independent scalability.

| Service | Port | Responsibility | Data Store |
| :--- | :--- | :--- | :--- |
| **API Gateway** | 8080 | Single entry point, JWT validation, Rate Limiting | Redis (Rate Limits) |
| **User Service** | 8081 | Auth, Profile CRUD, Connections | MySQL, AWS S3 |
| **Post Service** | 8082 | Posts, Likes, Comments, Media Upload | MySQL, AWS S3 |
| **Feed Service** | 8083 | Fan-out-on-write, Paginated Feed Delivery | Redis |
| **Search Service** | 8084 | Full-text search indexing & querying | Elasticsearch |
| **Notification Service** | 8085 | Async alerts for social events | Stateless |

## 🧠 Key Architectural Decisions

### 1. Fan-Out-On-Write Feed Generation
When a user creates a post, the `post.created` event triggers the Feed Service to push the post ID into the Redis list of **every connection**. 
- **Trade-off:** Expensive writes, but **O(1) reads** when fetching a feed.
- **Why:** Social feeds are read-heavy. Users expect instant feed loads even if post creation takes slightly longer.
- **Bounded Storage:** Redis lists are trimmed to `MAX_FEED_SIZE` per user to prevent unbounded memory growth.

### 2. Centralized JWT Validation at Gateway
Instead of validating tokens in every microservice, the **Spring Cloud Gateway** performs HMAC-SHA256 verification once using a reactive `GlobalFilter`.
- On success: Injects `X-User-Id` and `X-User-Email` headers into the request.
- Downstream services **trust these headers** implicitly, eliminating redundant crypto overhead and keeping services stateless.
- `/api/v1/auth/*` routes bypass this filter entirely.

### 3. Eventual Consistency for Search
Profile updates and new posts do **not** synchronously update Elasticsearch. Instead:
1. User/Post services publish events to Kafka topics (`user.updated`, `post.created`).
2. Search Service consumes these events asynchronously to index documents.
3. **Resilience:** If Elasticsearch is down, Kafka retains messages; consumers replay them upon recovery without data loss.

### 4. Decoupled Media Storage
Binary media never touches application servers. The `S3Service` generates UUID-prefixed keys and streams files directly to **AWS S3**, returning only the public URL to be stored in relational databases.

## 🛠️ Tech Stack

- **Languages:** Java 17
- **Framework:** Spring Boot 3, Spring Cloud Gateway, Spring Data JPA/Redis/Elasticsearch
- **Messaging:** Apache Kafka (KRaft mode)
- **Data Stores:** MySQL 8, Redis 7, Elasticsearch 8, AWS S3
- **Security:** JWT (jjwt), BCrypt, Spring Security Crypto
- **Infrastructure:** Docker Compose, Maven

## 🚀 Quick Start

### Prerequisites
- Java 17+, Maven 3.9+
- Docker Desktop
- AWS Account (for S3 credentials)

### 1. Start Infrastructure
```bash
cd linkedin-system
docker compose up -d redis elasticsearch kafka
