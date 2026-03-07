# ScaffoldOps — Project Handoff / Context Summary

## Overview

**ScaffoldOps** is a platform for automated generation and deployment of microservices on Kubernetes.

The core idea is similar to **Spring Initializr**, but extended so that the platform not only generates a starter project, but also prepares it for packaging and deployment, and tracks the lifecycle of that process.

ScaffoldOps should:
- accept a **microservice specification**
- generate a standardized microservice from a template
- package it
- deploy it to Kubernetes
- expose lifecycle status to the user

This is the **platform project**.

A separate future project called **Energyco / energyC&O** was discussed, but it is **not part of ScaffoldOps MVP**.

---

## Final Project Description

**ScaffoldOps** is a platform for generating, packaging, and deploying standardized microservices from a declarative specification. It automates project scaffolding, containerization, and Kubernetes deployment to reduce repetitive setup work and accelerate microservice development.

### Academic title
**ScaffoldOps: A Platform for Automated Generation and Deployment of Microservices on Kubernetes**

---

## Scope Decision

We explicitly separated two ideas:

### ScaffoldOps
An internal developer platform / generator platform.

### Energyco / energyC&O
A future separate business/domain project for energy price collection, analytics, alerts, and APIs.

ScaffoldOps must remain focused on:
- generation
- standardization
- deployment
- lifecycle tracking

It should **not** become a generic “everything platform.”

---

## MVP Architecture

### Core services
ScaffoldOps MVP should have **3 core microservices**:

#### 1. `generator-api`
The front door of the platform.

Responsibilities:
- receive the microservice specification
- validate the request
- persist the request
- expose status endpoints
- trigger the next workflow step

It should **not** do code generation itself and should **not** deploy directly.

#### 2. `generator-worker`
Responsibilities:
- consume generation requests
- generate the microservice from a template
- create the project structure and files
- generate Dockerfile / deployment assets
- prepare the project for build/deploy

#### 3. `deployment-worker`
Responsibilities:
- react after generation is complete
- deploy the generated service to Kubernetes
- update lifecycle status
- later maybe coordinate image build/push or GitOps steps

---

## Supporting platform components

### Keycloak
Used for:
- platform login
- token issuance
- protecting ScaffoldOps APIs

Keycloak is a **supporting platform component**, not the thesis core.

### Kafka
Optional for MVP, but discussed as a way to represent lifecycle events such as:
- request received
- generation started
- generation finished
- deployment started
- deployment failed / succeeded

Kafka should support the platform workflow, not dominate the project scope.

### API Gateway
Discussed, but **not a first-priority MVP component**.

---

## What `generator-api` is supposed to do

`generator-api` is the **entrypoint/orchestrator API** of ScaffoldOps.

### It should:
- accept a generation request
- validate it
- store it in PostgreSQL
- set/request lifecycle status
- expose status retrieval endpoints

### It should not:
- contain code generation logic
- build Docker images
- deploy to Kubernetes
- include Kafka initially
- include complex auth logic initially

### Initial endpoints
- `POST /generation-requests`
- `GET /generation-requests/{id}`
- optionally `GET /generation-requests`

---

## First implementation step

We agreed the first thing to build is the **generation request contract**, not the workers.

### Example request payload

```json
{
  "name": "billing-service",
  "template": "spring-boot-hexagonal",
  "database": true,
  "restApi": true,
  "security": true,
  "messaging": false,
  "deploymentTarget": "kubernetes"
}
