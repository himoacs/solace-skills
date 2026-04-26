# Solace Skills

A comprehensive collection of AI agent skills for developing, operating, and auditing Solace PubSub+ event broker solutions.

## Overview

This repository contains skills that help AI agents understand Solace PubSub+ concepts, write better Solace applications, and follow best practices. Each skill provides domain-specific guidance, code examples, and automation patterns.

## Skills Catalog

### Core Platform Skills (P0)

| Skill | Description |
|-------|-------------|
| [solace-agent-mesh](./skills/solace-agent-mesh) | Solace Agent Mesh (SAM) development for agentic AI workflows |
| [solace-event-portal](./skills/solace-event-portal) | Event Portal design, governance, and REST API usage |

### Platform & Development (P1)

| Skill | Description |
|-------|-------------|
| [solace-cloud-platform](./skills/solace-cloud-platform) | Solace Cloud Console, Mission Control, and cloud APIs |
| [solace-micro-integrations](./skills/solace-micro-integrations) | Connectors, Kafka bridge, and integration patterns |
| [solace-java-development](./skills/solace-java-development) | JCSMP, JMS, and JavaRTO development patterns |
| [solace-spring-development](./skills/solace-spring-development) | Spring Cloud Stream Binder and Spring Boot integration |
| [solace-python-development](./skills/solace-python-development) | Python API, asyncio patterns, and Pandas integration |
| [solace-javascript-development](./skills/solace-javascript-development) | Browser, Node.js, TypeScript, and React patterns |
| [solace-go-development](./skills/solace-go-development) | Go API, goroutines, context handling, and Kubernetes |
| [solace-dotnet-development](./skills/solace-dotnet-development) | .NET API, async patterns, and IHostedService |
| [solace-c-development](./skills/solace-c-development) | Native C API, embedded systems, and low latency |
| [solace-mqtt-development](./skills/solace-mqtt-development) | MQTT 3.1.1/5.0, IoT patterns, QoS, and LWT |
| [solace-rest-messaging](./skills/solace-rest-messaging) | REST Delivery Points, webhooks, and MicroGateway |

### Operations & Architecture (P2)

| Skill | Description |
|-------|-------------|
| [solace-operations](./skills/solace-operations) | SEMP API, CLI, monitoring, and broker administration |
| [solace-security](./skills/solace-security) | TLS, OAuth, LDAP, ACLs, and security hardening |
| [solace-troubleshooting](./skills/solace-troubleshooting) | Debugging connections, delivery, and performance |
| [solace-topic-architecture](./skills/solace-topic-architecture) | Topic hierarchy design, naming conventions, wildcards |
| [solace-messaging-patterns](./skills/solace-messaging-patterns) | Request/reply, saga, DLQ, fan-out, event sourcing |
| [solace-event-mesh](./skills/solace-event-mesh) | DMR clusters, multi-site distribution, hybrid cloud |

### Best Practices Audit Skills

| Skill | Description |
|-------|-------------|
| [solace-health-check](./skills/solace-health-check) | Comprehensive health audit with scoring framework |
| [solace-app-review](./skills/solace-app-review) | Application code review against best practices |
| [solace-topic-review](./skills/solace-topic-review) | Topic naming and hierarchy audit |
| [solace-security-review](./skills/solace-security-review) | Security configuration audit |
| [solace-queue-review](./skills/solace-queue-review) | Queue configuration and spool audit |
| [solace-performance-review](./skills/solace-performance-review) | Performance analysis and optimization |
| [solace-event-design-review](./skills/solace-event-design-review) | Event schema and AsyncAPI audit |
| [solace-semp-review](./skills/solace-semp-review) | SEMP API usage and automation audit |

## Usage

### With GitHub Copilot

Place skills in your Copilot skills directory:

```bash
# Global skills
~/.copilot/skills/

# Repository-specific
.github/skills/
```

### Skill Structure

Each skill follows this structure:

```
skill-name/
├── SKILL.md           # Main skill file with guidance
├── references/        # Supporting documentation
└── examples/          # Code examples
```

### Skill File Format

```yaml
---
name: skill-name
description: 'Keywords for skill discovery'
argument-hint: 'options for /skill command'
---

# Skill Title

Markdown content with guidance, examples, and best practices.
```

## Contributing

1. Follow the existing skill structure
2. Include practical code examples
3. Add reference links to official documentation
4. Test with AI agents before submitting

## License

[Add your license here]

## Resources

- [Solace Documentation](https://docs.solace.com)
- [Solace Tutorials](https://tutorials.solace.dev)
- [Solace Developer Portal](https://solace.dev)
- [Solace GitHub](https://github.com/SolaceSamples)
