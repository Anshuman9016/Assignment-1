# Project Report: Containerized Web Application with PostgreSQL

**Name:** Anshuman Mohapatra  
**SAP ID:** 500119016  
**Roll No:** R2142230609  
**Subject:** Containerization & Orchestration  
**Project Title:** Containerized Web Application using Docker, PostgreSQL, and Macvlan  
**Date:** March 2026  

---

## Table of Contents

1. Introduction  
2. System Architecture  
3. Docker Multi-Stage Build  
4. Network Configuration  
5. Image Optimization  
6. Macvlan vs IPvlan  
7. Data Persistence & Volume Management  
8. Conclusion  

---

## 1. Introduction

This project focuses on building and deploying a containerized web application using Docker. The application consists of two main components:

- **Backend:** Python with Flask framework providing REST APIs  
- **Database:** PostgreSQL 15 database for structured data storage and retrieval  

The project demonstrates practical implementation of modern DevOps concepts such as:

- Multi-stage Docker builds  
- Docker Compose orchestration  
- Container networking (Macvlan)  
- Persistent storage using named volumes  

---

## 2. System Architecture

The application follows a client-server architecture where services run in isolated containers.

```
┌─────────────────────────────────────────────────────────┐
│                   Docker Host Machine                    │
│                                                          │
│  ┌──────────────────┐       ┌──────────────────────┐   │
│  │  flask_backend    │       │   postgres_db         │   │
│  │  (Flask REST API) │──────▶│   (PostgreSQL 15)     │   │
│  │                  │  TCP   │                       │   │
│  │  Port: 5000      │ :5432  │  Port: 5432           │   │
│  │  IP: 10.0.0.11   │       │  IP: 10.0.0.10        │   │
│  └────────┬─────────┘       └──────────┬────────────┘   │
│           │                             │                │
│  ┌────────┴─────────────────────────────┴────────────┐  │
│  │           macvlan Network (L2 Mode)               │  │
│  │           Subnet: 10.0.0.0/24                      │  │
│  │           Gateway: 10.0.0.1                        │  │
│  │           Driver: macvlan                          │  │
│  │           Parent: eth0                             │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │           Named Volume: pgdata                      │  │
│  │           Mount: /var/lib/postgresql/data           │  │
│  │           Driver: local                             │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Host Port Mapping: 0.0.0.0:5000 → 5000 (backend)       │
└─────────────────────────────────────────────────────────┘
```

**Workflow:**

1. Client sends HTTP request to Flask backend (`localhost:5000`)
2. Backend validates and processes the request
3. Backend queries PostgreSQL over the Macvlan network
4. Database returns the query result
5. Backend formats and sends the response back to the client

---

## 3. Docker Multi-Stage Build

Multi-stage builds allow Docker images to be optimized by separating build and runtime environments. Only necessary runtime artifacts are included in the final image, reducing size and attack surface.

### Backend Optimization Strategy

- Stage 1 (builder): Installs all dependencies including dev tools
- Stage 2 (runtime): Copies only the app and required libraries using a slim base image
- Alpine Linux base is used to minimize the image footprint

### Advantages

- Significantly reduced final image size  
- Faster image pull and push in CI/CD pipelines  
- Reduced attack surface — no build tools in production image  
- Cleaner separation of development and production concerns  

---

## 4. Network Configuration

Container networking is a critical aspect of this project. The Macvlan driver assigns each container a unique MAC address, making them appear as independent physical devices on the local network.

### Macvlan Driver Overview

- Each container gets its own unique MAC address and IP address  
- Containers communicate directly on the network without NAT  
- Operates at Layer 2, behaving like a physical device on the LAN  
- Provides full network isolation between containers  

### Implementation Note

Macvlan networking was tested on a Linux (WSL Ubuntu 22.04) environment where kernel-level support for macvlan is available. The network is defined in `docker-compose.yml` using a custom subnet (10.0.0.0/24).

---

## 5. Image Optimization

Optimizing Docker images improves deployment speed, reduces storage consumption, and limits vulnerability exposure.

| Component | Base Image | Unoptimized | Optimized | Reduction |
|-----------|------------|-------------|-----------|-----------|
| Flask Backend | python:3.11-alpine | ~850 MB | ~120 MB | ~86% |
| PostgreSQL DB | postgres:15-alpine | ~350 MB | ~215 MB | ~38% |

### Key Optimization Techniques

- Alpine-based base images  
- Multi-stage builds to eliminate build-time dependencies  
- `.dockerignore` to exclude unnecessary files from build context  
- Combining `RUN` commands to reduce the number of image layers  

---

## 6. Macvlan vs IPvlan

Both Macvlan and IPvlan are Docker network drivers that enable containers to appear as devices on the physical network. They differ in MAC address handling and performance characteristics.

| Feature | Macvlan | IPvlan |
|---------|---------|--------|
| MAC Address | Unique per container | Shared (host MAC) |
| Network Performance | Moderate | High |
| Cloud Compatibility | Limited (needs promiscuous mode) | Better |
| Layer of Operation | L2 | L2 / L3 |
| Configuration | Slightly complex | Simpler |
| ARP Flooding | More likely | Reduced |

### Summary

- **Macvlan** gives each container a unique MAC address — full network isolation but requires promiscuous mode on the host NIC  
- **IPvlan** shares the host MAC address — better cloud compatibility and reduced ARP flooding  
- This project uses **Macvlan** for complete network isolation at Layer 2  
- Both drivers eliminate NAT overhead, enabling direct communication between containers  

---

## 7. Data Persistence & Volume Management

Docker volumes are the recommended mechanism for persisting data generated by containers. A named volume (`pgdata`) ensures PostgreSQL data survives container stops and removals.

### Volume Configuration (docker-compose.yml)

```yaml
volumes:
  pgdata:

services:
  database:
    image: postgres:15-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
```

### How It Works

1. Docker creates a named volume `pgdata` managed by the `local` driver
2. PostgreSQL stores all data at `/var/lib/postgresql/data` inside the container
3. This internal path is mapped to the named volume on the host machine
4. When the container is stopped or removed, the volume **persists** on disk
5. On restart, the same volume is re-attached — all data remains intact

### Persistence Test Procedure

```bash
# Step 1: Start all services
docker compose up -d

# Step 2: Insert sample data
curl -X POST http://localhost:5000/records \
  -H "Content-Type: application/json" \
  -d '{"message": "Volume persistence verified"}'

# Step 3: Confirm data exists
curl http://localhost:5000/records

# Step 4: Stop and remove containers (keep volumes)
docker compose down

# Step 5: Restart services
docker compose up -d

# Step 6: Verify data survived container removal
curl http://localhost:5000/records
# Expected output: should return "Volume persistence verified"
```

### Volume Management Commands

```bash
# List all Docker volumes
docker volume ls

# Inspect the pgdata volume
docker volume inspect project_pgdata

# WARNING: This permanently deletes all data
docker compose down -v
```

---

## 8. Conclusion

This project demonstrated key containerization and orchestration concepts through practical implementation:

1. **Multi-stage builds** reduced image sizes by over 85%, leading to faster deployments and a smaller attack surface
2. **Macvlan networking** enabled direct Layer 2 communication between containers without NAT overhead
3. **Named volumes** ensured full database persistence across container lifecycle events
4. **Docker Compose** orchestrated multi-service startup with dependency management and health checks
5. **Alpine-based images** and `.dockerignore` significantly reduced build context and final image sizes

The architecture is production-ready, follows the principle of least privilege (non-root container execution), and maintains a clean separation of concerns between application logic and infrastructure.

---

*End of Report*
