# Petistaan-ConfigServer

Centralized configuration server for the [Petistaan-MS](https://github.com/AbhishekSharma-99/Petistaan-MS)
pet management system. Built on **Spring Cloud Config Server**, it serves environment-specific
properties to every downstream service by pulling from a private, Git-backed configuration
repository at startup.

> **To run the full system** (all services + MySQL + Eureka + API Gateway), see the
> [Petistaan-MS](https://github.com/AbhishekSharma-99/Petistaan-MS) hub repo.
> The `docker-compose.yml` lives there.

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [API Reference](#api-reference)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Running Locally](#running-locally)
- [Branch Strategy](#branch-strategy)
- [Part of Petistaan-MS](#part-of-petistaan-ms)

---

## Overview

ConfigServer is a standalone Spring Boot service responsible for:

- Cloning the [Petistaan-Configuration](https://github.com/AbhishekSharma-99/Petistaan-Configuration)
  private Git repository on startup (`main` branch) and exposing its contents as resolvable properties
- Serving per-service, per-profile configuration to every other Petistaan-MS microservice over HTTP,
  so none of them need environment-specific `.properties`/`.yml` baked into their own repos
- Authenticating against the private config repo via a **read-only, fine-grained GitHub PAT**,
  supplied through environment variables — never committed to source
- Acting as the first service that must be up and healthy before EurekaServer, APIGateway,
  OwnerMS, PetMS, or MailMS can reliably fetch their configuration

ConfigServer holds no domain logic of its own — it is pure infrastructure, and `@EnableConfigServer`
is the entire feature surface. Its only "database" is the Git repository it clones.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 25 |
| Framework | Spring Boot 4.1.0 |
| Config Backend | Spring Cloud Config Server 2025.1.2 |
| Monitoring | Spring Boot Actuator |
| Build | Maven 3.9.x (Maven Wrapper) |
| Config Source | Git (private repo, clone-on-start) |

---

## API Reference

Base URL: `http://localhost:8888`

Spring Cloud Config Server exposes configuration via a convention-based URL pattern rather
than custom endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/{application}/{profile}` | Resolved config for a service + profile (e.g. `/OwnerMS/dev`) |
| `GET` | `/{application}/{profile}/{label}` | Same, pinned to a specific Git branch/tag (e.g. `/OwnerMS/dev/main`) |
| `GET` | `/{application}-{profile}.properties` | Config as a flat `.properties` file |
| `GET` | `/{application}-{profile}.yml` | Config as YAML |
| `GET` | `/actuator/health` | Service health check |

### Sample request

```
GET /OwnerMS/dev
```

Returns the merged property sources for OwnerMS's `dev` profile, sourced from the
corresponding file(s) in the `Petistaan-Configuration` repo — e.g. `OwnerMS-dev.properties`.

---

## Project Structure

```
src/main/java/com/abhishek/
└── PetistaanConfigServerApplication.java   # @EnableConfigServer entry point

src/main/resources/
└── application.properties                 # Git source, credentials, port

.env.example                                # Template for required GitHub credentials
.mvn/wrapper/                               # Maven Wrapper distribution config
.gitattributes                              # LF/CRLF enforcement for mvnw/mvnw.cmd
.gitignore                                  # IDE artifacts, build output, .env
pom.xml
```

There is no `controller`, `service`, or `repository` layer here by design — Spring Cloud
Config Server's autoconfiguration handles routing, resolution, and Git polling internally
once `@EnableConfigServer` is present.

---

## Configuration

All sensitive values are externalized. Copy `.env.example` to `.env` and fill in your values:

```dotenv
GitHub_Username=your_github_username
GitHub_Personal_Access_Token=your_fine_grained_pat_here
```

> The token must be a **fine-grained, read-only PAT** scoped to the private
> `Petistaan-Configuration` repo — it is only ever used to clone config files, never to write.

`application.properties` resolves these at runtime via `${GitHub_Username}` and
`${GitHub_Personal_Access_Token}`. The `.env` file is gitignored and never committed.

Key properties:

| Property | Value |
|---|---|
| `server.port` | `8888` |
| `spring.cloud.config.server.git.uri` | `https://github.com/AbhishekSharma-99/Petistaan-Configuration` |
| `spring.cloud.config.server.git.clone-on-start` | `true` |
| `spring.cloud.config.server.default-label` | `main` |

If credentials are missing or the token has expired, the service will fail to clone the
config repo on startup — check logs for a Git authentication error first if the service
won't boot.

---

## Running Locally

**Prerequisites:** Java 25, network access to GitHub, a valid read-only PAT for the
private `Petistaan-Configuration` repo.

> To spin up the full environment in one command, use the `docker-compose.yml` in the
> [Petistaan-MS](https://github.com/AbhishekSharma-99/Petistaan-MS) hub repo.

To run this service in isolation:

```bash
# 1. Clone the repo
git clone https://github.com/AbhishekSharma-99/Petistaan-ConfigServer.git
cd Petistaan-ConfigServer

# 2. Set up environment
cp .env.example .env
# Edit .env with your GitHub credentials

# 3. Run
./mvnw spring-boot:run
```

Service starts on `http://localhost:8888`. Verify it's serving config with:

```bash
curl http://localhost:8888/OwnerMS/dev
```

---

## Branch Strategy

```
main  ●──────────────────────────────────────────● (merge via PR)
       \                                        /
dev     ●──●──●──●──●──●──●──●──●──●──●──●─────
```

- `main` — stable, production-ready commits only; no direct pushes
- `dev` — integration branch; all feature work merges here first
- `feature/*` — short-lived branches off `dev` for individual concerns

---

## Part of Petistaan-MS

Petistaan-ConfigServer is one service in the broader Petistaan microservices ecosystem:

| Service | Port | Responsibility |
|---|---|---|
| [Petistaan-EurekaServer](https://github.com/AbhishekSharma-99/Petistaan-EurekaServer) | 8761 | Service discovery |
| **Petistaan-ConfigServer** | **8888** | **Centralized config** |
| [Petistaan-APIGateway](https://github.com/AbhishekSharma-99/Petistaan-APIGateway) | 8080 | Single entry point |
| [Petistaan-OwnerMS](https://github.com/AbhishekSharma-99/Petistaan-OwnerMS) | 8081 | Owner management |
| [Petistaan-PetMS](https://github.com/AbhishekSharma-99/Petistaan-PetMS) | 8082 | Pet management |
| [Petistaan-MailMS](https://github.com/AbhishekSharma-99/Petistaan-MailMS) | 8083 | Email dispatch |

See [Petistaan-MS](https://github.com/AbhishekSharma-99/Petistaan-MS) for system
architecture, build order, and Docker Compose setup.
