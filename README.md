# my-boilerplate-go

This repository serves as a collection of Go language template repositories for my personal use. All template repositories are public and free to use. You'll find the architectural references that inspired these templates listed below. If you find these templates helpful for your own projects, I'd appreciate a GitHub Star!

[English](./README.md) | [日本語](./README.jp.md)

## Software Architecture Patterns

### Data-Centric

1. [**CQRS Architecture**](docs/en/cqrs-architecture.md):  
separates read and write operations for a data store. It enables independent scaling of read and write workloads and optimizes them separately.

### Layered

1. [**Layered(n-tier) Architecture**](docs/en/layered-architecture.md):  
separates software into logical layers.
    - [i7s7-ymp/go-layered](https://github.com/i7s7-ymp/go-layered).

### Component-Based

1. [**Microkernel**](docs/en/microkernel-architecture.md):  
separates a minimal functional core from extended functionality and customer-specific parts.
    - WIP

### Service-Oriented

1. [**Microservice**](docs/en/microservices-architecture.md):  
This architecture designs a software application as a suite of independently deployable, small, modular services.
   - [i7s7-ymp/go-microservice](https://github.com/i7s7-ymp/go-microservice.git)

### Distributed System

1. [**Space-Based**](docs/en/space-based-architecture.md):
This resolves the issues of data consistency, reliable performance, and scalability for large-scale distributed systems.
   - WIP

### Domain-Driven

1. [**DDD**](docs/en/domain-driven-design.md):  
focuses on the domain logic and complexity rather than the technology used.
   - WIP

### Event-Driven

1. [**Event-Driven**](docs/en/event-driven-architecture.md):  
promotes the production, detection, consumption of, and reaction to events.
   - WIP

### Separation Of Concern

1. [**MVP**](docs/en/mvp-architecture.md):  
derivative of the Model-View-Controller(MVC) pattern, which aims to separate the concerns of data management, user interface, and control flow.
   - WIP

### Interpreter

1. [**Interpreter**](docs/en/interpreter-pattern.md):  
written in a high-level language, which the interpreter translates into executable code.
   - WIP

### Concurrency

1. [**Orchestration**](docs/en/orchestration-pattern.md): 
a central coordinator (often called an orchestrator) that directs the interaction between services. The orchestrator is responsible for managing the control flow and data flow between services.
   - WIP

## Reference

- [ByteByteGo - Top 5 Software Architectural Patterns](https://bytebytego.com/guides/top-5-software-architectural-patterns/)
