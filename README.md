# my-boilerplate-go

This repository serves as a collection of Go language template repositories for my personal use. All template repositories are public and free to use. You'll find the architectural references that inspired these templates listed below. If you find these templates helpful for your own projects, I'd appreciate a GitHub Star!

[English](README.md) | [日本語](README.jp.md)

## Software Architecture Patterns

### Data-Centric

1. **CQRS Architecture**:  
separates read and write operations for a data store. It enables independent scaling of read and write workloads and optimizes them separately.

### Layered

1. **Layered(n-tier) Architecture**:  
separates software into logical layers. [show detail](docs/en/layered-architecture.md)
    - [i7s7-ymp/go-layered](https://github.com/i7s7-ymp/go-layered).

### Component-Based

1. **Microkernel**:  
separates a minimal functional core from extended functionality and customer-specific parts. [show detail](docs/en/microkernel-architecture.md).
    - WIP

### Service-Oriented

1. **Microservice**:  
This architecture designs a software application as a suite of independently deployable, small, modular services. [show detail](docs/en/microservices-architecture.md).
   - WIP

### Distributed System

1. **Space-Based**:
This resolves the issues of data consistency, reliable performance, and scalability for large-scale distributed systems. [show detail](docs/en/space-based-architecture.md).
   - WIP

### Domain-Driven

1. **DDD**:  
focuses on the domain logic and complexity rather than the technology used. [show detail](docs/en/domain-driven-design.md).
   - WIP

### Event-Driven

1. **Event-Driven**:  
promotes the production, detection, consumption of, and reaction to events. [show detail](docs/en/event-driven-architecture.md).
   - WIP

### Separation Of Concern

1. **MVP**:  
derivative of the Model-View-Controller(MVC) pattern, which aims to separate the concerns of data management, user interface, and control flow. [show detail](docs/en/mvp-architecture.md).
   - WIP

### Interpreter

1. **Interpreter**:  
written in a high-level language, which the interpreter translates into executable code. [show detail](docs/en/interpreter-pattern.md).
   - WIP

### Concurrency

1. **Orchestration**: 
a central coordinator (often called an orchestrator) that directs the interaction between services. The orchestrator is responsible for managing the control flow and data flow between services. [show detail](docs/en/orchestration-pattern.md). 
   - WIP

## Reference

- [ByteByteGo - Top 5 Software Architectural Patterns](https://bytebytego.com/guides/top-5-software-architectural-patterns/)
