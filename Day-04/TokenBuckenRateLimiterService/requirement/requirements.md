# Project 01

# Token Bucket Rate Limiter Service

## Overview

Build a standalone Rate Limiter Service that exposes REST APIs for other applications to consume.

Unlike embedding a rate limiting library inside an application, this project implements rate limiting as an independent network service responsible for maintaining shared state across multiple clients.

The service should support configurable rate limiting policies while ensuring correctness under heavy concurrent traffic.

---

# Core Requirements

## 1. Rate Limit Endpoint

Expose an endpoint that accepts a client identifier and determines whether the request should be allowed or denied using the Token Bucket algorithm.

Expected response:

- ALLOW
- DENY

---

## 2. Configurable Policies

Support per-client configuration including:

- Requests per second
- Bucket capacity
- Burst size (optional)
- Algorithm selection

Configurations should be manageable through Admin APIs.

---

## 3. Persistent State

Bucket state must survive service restarts.

The implementation must not rely solely on in-memory state.

---

## 4. Concurrency Safety

Multiple simultaneous requests for the same client must never consume the same token twice.

The service must remain race-condition free under high concurrency.

---

## 5. Multiple Algorithms

Support:

- Token Bucket
- Sliding Window

Algorithm selection should be configurable per client.

---

## 6. Standard Rate Limit Headers

Every response should include:

- X-RateLimit-Limit
- X-RateLimit-Remaining
- X-RateLimit-Reset

---

## 7. Load Testing

Demonstrate correctness under:

- 500+ concurrent requests/sec

The load test should validate:

- No double token consumption
- Correct remaining tokens
- Stable latency

---

# Stretch Goals

## Distributed Deployment

Support multiple service instances sharing bucket state correctly.

---

## Monitoring Dashboard

Provide a simple dashboard displaying:

- Allowed Requests
- Denied Requests
- Requests/sec
- Active Clients

---

# Expected Skills

- Distributed Systems
- API Design
- Concurrency
- Synchronization
- Atomic Operations
- Persistence
- Rate Limiting Algorithms
- Load Testing
- REST API Design
