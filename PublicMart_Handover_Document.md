# Public Mart Application Handover Document

**Production Support Engineer Guide**

| Item | Value |
|------|-------|
| **Document Version** | 1.0 |
| **Date** | April 22, 2026 |
| **Classification** | Internal Use |

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Application Overview](#application-overview)
3. [Technical Architecture](#technical-architecture)
4. [Infrastructure & Deployment](#infrastructure--deployment)
5. [Key Components & Services](#key-components--services)
6. [Access & Connectivity](#access--connectivity)
7. [Common Production Issues](#common-production-issues)
8. [Monitoring & Alerting](#monitoring--alerting)
9. [Operational Procedures](#operational-procedures)
10. [Support & Escalation](#support--escalation)

---

## Executive Summary

Public Mart is a modern e-commerce marketplace application designed to manage product catalog, inventory, orders, and user transactions. The application utilizes a microservices architecture with containerized backend services running on Red Hat OpenShift Kubernetes (OCP) and frontend services deployed on IIS servers.

This document provides comprehensive technical documentation for production support engineers responsible for maintaining application availability, diagnosing issues, and ensuring optimal performance in the production environment.

### Key Application Information

| Attribute | Value |
|-----------|-------|
| **Application Name** | Public Mart |
| **Environment** | Production |
| **Backend Platform** | Red Hat OpenShift Kubernetes |
| **Frontend Platform** | IIS Servers (Windows) |
| **OCP Namespace** | apps-pbm-prod |
| **Frontend Servers** | 10.170.65.23 (Primary) / 10.170.65.24 (Secondary) |
| **Production Domain** | pbm |
| **Cache System** | Redis |
| **Backend Source Control** | Bitbucket |
| **Frontend Source Control** | Azure DevOps |
| **Backend CI/CD Pipeline** | CloudBees (Jenkins) |
| **File Repository** | FTP Server |

---

## Application Overview

### Purpose & Functionality

- Product catalog management and search
- Inventory tracking and stock management
- Order processing and fulfillment
- User account management and authentication
- Payment processing and transaction handling
- Reporting and analytics

### Business Context

- **Target Users:** Both customers and internal administrators
- **Availability Requirement:** 24/7 availability expected
- **Peak Traffic Hours:** 9 AM - 10 PM (local business time)
- **Critical Transactions:** Order placement, payment processing, inventory updates

### Support Expectations

The production support team should be prepared to:

- Monitor application health 24/7
- Respond to alerts within defined SLAs
- Diagnose and resolve issues quickly
- Maintain detailed logs of all incidents
- Escalate complex issues to development teams
- Perform scheduled maintenance and deployments

---

## Technical Architecture

### Architecture Overview

Public Mart follows a modern microservices architecture with separated frontend and backend tiers:

| Layer | Components |
|-------|-----------|
| **Presentation Layer** | IIS web servers (10.170.65.23, 10.170.65.24) |
| **Application Layer** | Microservices on OpenShift (apps-pbm-prod) |
| **Data Access Layer** | Database layer with primary/replica setup |
| **Caching Layer** | Redis distributed cache cluster |
| **Storage Layer** | FTP server for file uploads/downloads |

### Technology Stack

| Component | Technology |
|-----------|-----------|
| **Container Orchestration** | Red Hat OpenShift Kubernetes |
| **Backend Framework** | ASP.NET Core / Java Spring Boot (TBD) |
| **Frontend Framework** | ASP.NET MVC / Razor Pages |
| **Web Server** | Internet Information Services (IIS) 10.0+ |
| **Cache Store** | Redis |
| **Database** | SQL Server / PostgreSQL (TBD) |
| **File Storage** | FTP Server |
| **Message Queue** | RabbitMQ / Kafka (TBD) |
| **Logging & Monitoring** | ELK Stack / Splunk (TBD) |
| **CI/CD** | CloudBees (Backend), Azure DevOps (Frontend) |

### High-Level System Design

The application follows a classic three-tier architecture pattern:

**Presentation Tier:** IIS-hosted frontend web application serving user requests

**Application Tier:** RESTful backend APIs deployed on OpenShift microservices

**Data Tier:** Database, cache, and file storage services

---

## Infrastructure & Deployment

### Backend Infrastructure (OpenShift Kubernetes)

| Item | Details |
|------|---------|
| **Platform** | Red Hat OpenShift Kubernetes |
| **Production Namespace** | apps-pbm-prod |
| **Node Type** | Multiple worker nodes (high availability setup) |
| **CPU & Memory** | To be defined based on service requirements |
| **Storage** | PersistentVolumeClaims for stateful components |
| **Ingress** | OpenShift routes or Kubernetes ingress |
| **High Availability** | Minimum 2 replicas per microservice |

### Frontend Infrastructure (IIS Servers)

| Item | Details |
|------|---------|
| **Server 1 (Primary)** | 10.170.65.23 - Windows Server |
| **Server 2 (Secondary)** | 10.170.65.24 - Windows Server |
| **Web Server** | IIS 10.0 or later |
| **.NET Framework** | ASP.NET Core (version TBD) |
| **Application Pool** | Dedicated pool with recycling policy |
| **Load Balancing** | Network load balancer or hardware load balancer |
| **SSL/TLS** | HTTPS enabled, TLS 1.2 or higher |
| **Session State** | Distributed (Redis) for failover capability |

### Data Storage Infrastructure

- **Redis Cache Cluster:** Session storage and distributed caching
- **FTP Server:** File upload/download repository for product images and documents
- **Primary Database:** Production transactional data store
- **Backup Database:** Standby replica for high availability

### Deployment Pipeline

#### Backend (Bitbucket + CloudBees)

- **Source Code:** Bitbucket repository
- **CI/CD Pipeline:** CloudBees (Jenkins-based)
- **Deployment Target:** OpenShift Kubernetes cluster
- **Deployment Strategy:** Rolling updates with health checks

#### Frontend (Azure DevOps)

- **Source Code:** Azure DevOps repository
- **Build Pipeline:** Azure DevOps build
- **Deployment Target:** IIS Servers
- **Deployment Strategy:** Manual/automated to both servers

---

## Key Components & Services

### Backend Microservices

| Service Name | Purpose | Port | Replicas |
|-------------|---------|------|----------|
| **API Gateway** | Request routing, load balancing, and authentication | 8080 | 2+ |
| **Product Service** | Catalog management, product search, and details | 8081 | 2+ |
| **Order Service** | Order processing, fulfillment, and tracking | 8082 | 2+ |
| **User Service** | Authentication, user profiles, and management | 8083 | 2+ |
| **Payment Service** | Payment gateway integration and processing | 8084 | 2+ |
| **Notification Service** | Email and SMS alerts, user notifications | 8085 | 1+ |
| **Admin Service** | Internal management interface and operations | 8086 | 1+ |

### Frontend Components

- **Web UI:** ASP.NET Core web application serving customer portal
- **Admin Portal:** Internal administration interface for staff
- **Mobile Web:** Responsive design for mobile device access

### Supporting Services

- **Redis Cache:** Distributed session and cache store
- **Message Queue:** Asynchronous task processing and job queues
- **Email Service:** Transactional email notifications
- **Payment Gateway:** Third-party payment processor integration
- **Logging & Monitoring:** Centralized log aggregation and metrics

---

## Access & Connectivity

### Production URLs

| Application | URL | Protocol | Port | Notes |
|-------------|-----|----------|------|-------|
| **Customer Portal** | pbm.com/portal (or configured domain) | HTTPS | 443 | Public-facing application |
| **Admin Portal** | [Internal URL - TBD] | HTTPS | 443 | Internal network only |
| **API Gateway** | [Internal URL - TBD] | HTTPS | 8080 | Internal services communication |
| **Health Check Endpoint** | /api/health | HTTPS | 443 | Application status monitoring |

### OpenShift Kubernetes Access

**Cluster Information:**
- **Production Namespace:** apps-pbm-prod
- **OCP Console:** [To be provided by infrastructure team]
- **CLI Access:** oc login command (requires proper credentials)

**Common OpenShift Commands:**

```bash
oc login <cluster-url>                    # Authenticate to cluster
oc get pods -n apps-pbm-prod              # List all pods
oc describe pod <pod-name>                # Get pod details
oc logs -f <pod-name>                     # Stream pod logs
oc get svc                                # List services
oc get events                             # View cluster events
oc port-forward <pod> <local>:<remote>    # Port forwarding
oc scale deployment <name> --replicas=N   # Scale deployment
```

### IIS Server Access

- **Server 1 (Primary):** 10.170.65.23 - Windows Server (RDP Access)
- **Server 2 (Secondary):** 10.170.65.24 - Windows Server (RDP Access)
- **Access Method:** Remote Desktop Protocol (RDP)
- **Service Control:** IIS Manager
- **Web Root:** [To be provided by IIS administrator]

### Database & Cache Access

- **Database:** Connection string in application configuration (provide credentials securely)
- **Redis Cache:** Port 6379 (default), credentials stored in secrets management system
- **FTP Server:** Standard FTP/SFTP access for file repository (credentials in secrets)

---

## Common Production Issues

| Issue | Symptoms | Root Causes | Resolution Steps |
|-------|----------|------------|------------------|
| **Pod Crashes** | Services going offline, error messages in logs | Out of Memory (OOM), CPU limits exceeded, application crashes | Check pod logs, review resource limits, increase allocation, redeploy service |
| **High Latency** | Slow API responses, UI feels sluggish | Database bottleneck, cache misses, network issues, resource constraints | Optimize database queries, check cache hit rates, scale pods, investigate network |
| **Database Connection Errors** | 503 Service Unavailable, connection timeout errors | Connection pool exhaustion, database server down, network connectivity issue | Check database server status, restart connection pool, verify network connectivity |
| **Redis Cache Issues** | Session loss, data inconsistency, cache errors | Memory full, eviction policies, connection issues, cache server down | Flush cache (non-critical data), increase memory, restart cache, reconnect |
| **Frontend/IIS Down** | HTTP 500 errors, unresponsive website | Application pool crash, deployment failure, IIS service stopped | Check IIS event logs, restart application pool, redeploy application |
| **Authentication Failures** | Login errors, unexpected session timeouts | Token expiry, Redis session loss, auth service down | Clear sessions, restart authentication service, check Redis connectivity |
| **High Memory Usage** | System slowdown, pod restarts, OOM kills | Memory leak, cache not evicting, large data processing | Monitor memory trends, check for leaks, adjust cache policy, scale horizontally |
| **Payment Processing Failures** | Orders not completing, payment errors | Payment gateway timeout, API error, network connectivity | Check payment service logs, verify gateway connectivity, retry transactions |

> **Note:** Always check application logs first before making infrastructure changes. Log files often contain the root cause of issues.

---

## Monitoring & Alerting

### Key Metrics to Monitor

- **Pod Availability:** Ensure all pods are running and in Ready state
- **CPU & Memory Usage:** Monitor resource consumption per pod and node
- **Response Time:** Track API latency and frontend page load times
- **Error Rates:** Monitor HTTP 4xx and 5xx error percentages
- **Database Performance:** Query execution times, connection pool status
- **Redis Health:** Cache hit rates, eviction rates, memory usage
- **Disk Usage:** Available storage on servers and persistent volumes
- **Network I/O:** Bandwidth usage and packet loss

### Recommended Monitoring Tools

- **Prometheus:** Metrics collection and alerting framework
- **Grafana:** Visualization and custom dashboards
- **ELK Stack:** Elasticsearch, Logstash, Kibana for log aggregation
- **OpenShift Monitoring:** Built-in cluster monitoring and metrics
- **IIS Manager:** Windows server performance and application monitoring
- **New Relic / DataDog:** Optional APM for deeper application insights

### Alert Thresholds (Recommended)

| Metric | Warning Level | Critical Level | Duration |
|--------|--------------|-----------------|----------|
| **CPU Usage** | >70% | >90% | 5 min sustained |
| **Memory Usage** | >75% | >90% | 5 min sustained |
| **Error Rate** | >5% | >10% | 2 min window |
| **API Latency (p95)** | >500ms | >2000ms | 5 min average |
| **Pod Ready Status** | <100% | Pods offline | 1 min |
| **Disk Usage** | >70% | >85% | Immediate alert |
| **Database Connections** | >80% of pool | >95% of pool | 2 min |
| **Redis Hit Rate** | <50% | <30% | 10 min average |

### Dashboard Recommendations

Create dashboards for:
- Real-time cluster health (pods, nodes, resources)
- Application performance metrics (latency, errors, throughput)
- Business metrics (orders processed, transactions completed)
- Infrastructure metrics (disk space, network I/O, database connections)
- Alert status and active incidents

---

## Operational Procedures

### Application Startup Procedures

1. Verify OpenShift cluster is operational and accessible
2. Confirm primary database is running and accepting connections
3. Verify Redis cache cluster availability and connectivity
4. Check FTP server status for file repository access
5. Deploy backend microservices to OpenShift (apps-pbm-prod namespace)
6. Deploy frontend application to IIS servers (both primary and secondary)
7. Wait for all pods to reach Ready state (typically 5-10 minutes)
8. Perform smoke tests on all critical functions
9. Verify health check endpoint responds successfully
10. Monitor logs for any startup errors

### Graceful Shutdown Procedures

1. Stop accepting new requests at the load balancer level
2. Display maintenance page to users
3. Wait for in-flight requests to complete (max timeout 5 minutes)
4. Flush Redis cache (non-session critical data only)
5. Gracefully scale down backend services (allowing pods to drain)
6. Stop IIS service on frontend servers
7. Close database connections gracefully
8. Verify application is fully offline
9. Document shutdown time for auditing

### Deployment Procedures

1. Obtain approval from change management process
2. Create backup of current database
3. Notify users of upcoming maintenance window (if required)
4. Deploy backend services to OpenShift using rolling update strategy
5. Monitor backend health for 10 minutes post-deployment
6. Run smoke tests for backend API functionality
7. Deploy frontend application to primary IIS server (10.170.65.23)
8. Verify frontend functionality and test critical user flows
9. Deploy frontend application to secondary IIS server (10.170.65.24)
10. Perform end-to-end functional testing across both servers
11. Verify all monitoring and alerts are functioning
12. Update deployment log with version numbers and timestamps

### Rollback Procedures

1. Identify the issue or failure triggering rollback requirement
2. Assess severity and user impact
3. Determine rollback scope (backend only, frontend only, or both)
4. Obtain previous stable version/image from repository
5. Revert deployments to previous stable version
6. Monitor application health during rollback
7. Verify critical functions are working correctly
8. Investigate root cause of failed deployment
9. Notify all stakeholders of rollback and status
10. Document incident for post-mortem analysis

### Database Backup & Recovery

- **Backup Frequency:** Daily automated backups (schedule TBD)
- **Backup Location:** Off-site storage or secondary data center
- **Recovery Testing:** Monthly restore drills
- **RTO (Recovery Time Objective):** < 1 hour
- **RPO (Recovery Point Objective):** < 15 minutes

### Maintenance Windows

- **Scheduled Maintenance:** [To be defined by business]
- **Typical Duration:** 2-4 hours (includes testing)
- **Communication:** Notify users 48 hours in advance
- **Rollback Plan:** Always have a rollback plan ready

---

## Support & Escalation

### Support Contact Matrix

| Team/Role | Contact Person | Email | Phone | Availability |
|-----------|---|---|---|---|
| **L1 Support Lead** | [Team Lead Name] | pbm-support@company.com | [Phone Number] | 24/7 |
| **Backend Team Lead** | [Developer Lead] | pbm-backend@company.com | [Phone Number] | Business hours + On-call rotation |
| **Frontend Team Lead** | [Developer Lead] | pbm-frontend@company.com | [Phone Number] | Business hours + On-call rotation |
| **Infrastructure Team** | [Infrastructure Lead] | infra-support@company.com | [Phone Number] | 24/7 On-call |
| **Database Team (DBA)** | [DBA Lead] | dba-support@company.com | [Phone Number] | 24/7 On-call |
| **Business Owner** | [Product Owner] | [Email] | [Phone] | Business hours |

### Incident Response Procedure

1. **Alert Received:** Monitoring system triggers alert or incident reported
2. **Initial Triage:** Assess severity, scope, and immediate impact to business
3. **Acknowledgment:** Notify relevant teams immediately for P1/P2 issues
4. **Investigation:** Check application logs, metrics, and system status
5. **Root Cause Identification:** Determine underlying cause of the issue
6. **Mitigation:** Apply immediate fix or temporary workaround if needed
7. **Communication:** Update stakeholders on status and ETA every 15-30 minutes
8. **Resolution:** Implement permanent fix or escalate to dev team
9. **Verification:** Test that issue is fully resolved
10. **Documentation:** Log all incident details for post-mortem analysis

### Escalation Matrix by Severity

| Severity | Business Impact | Response Time | Resolution Target | Escalation Path |
|----------|---|---|---|---|
| **P1 - Critical** | Complete outage, 0% uptime, revenue impact | 15 minutes | 4 hours | L1 → Team Leads → Product Owner → CTO |
| **P2 - High** | Major functionality affected, >50% users | 30 minutes | 12 hours | L1 → Relevant Team Lead → Product Owner |
| **P3 - Medium** | Partial degradation, <50% users affected | 1 hour | 24 hours | L1 → Relevant Team Lead |
| **P4 - Low** | Minor issues, cosmetic problems, no business impact | 4 hours | 5 business days | L1 → Ticket queue |

### Known Limitations & Workarounds

- **Large File Uploads:** Recommend maximum file size of 500MB for optimal performance
- **Concurrent User Sessions:** System is designed to support up to 10,000 concurrent users
- **Redis Memory Limits:** Cache eviction policy activates at 80% capacity
- **Database Transaction Timeout:** Default 30 seconds (configurable in application settings)
- **API Rate Limiting:** Currently [TBD requests per second per API key]

### Quick Reference Commands

#### OpenShift Commands

```bash
# Authentication
oc login <cluster-url>

# Pod management
oc get pods -n apps-pbm-prod                    # List all pods
oc describe pod <pod-name> -n apps-pbm-prod     # Get pod details
oc logs -f <pod-name> -n apps-pbm-prod          # Stream pod logs
oc delete pod <pod-name> -n apps-pbm-prod       # Delete a pod

# Service management
oc get svc -n apps-pbm-prod                     # List services
oc get routes -n apps-pbm-prod                  # List routes

# Scaling
oc scale deployment <name> --replicas=N -n apps-pbm-prod

# Port forwarding
oc port-forward <pod> <local-port>:<remote-port> -n apps-pbm-prod

# View events
oc get events -n apps-pbm-prod --sort-by='.lastTimestamp'
```

#### IIS Commands (Windows Server)

```bash
# Service management
iisreset                              # Restart IIS
iisreset /start                       # Start IIS
iisreset /stop                        # Stop IIS

# Application pool commands
Get-IISAppPool -Name "<pool-name>"
Start-IISAppPool -Name "<pool-name>"
Stop-IISAppPool -Name "<pool-name>"
Restart-IISAppPool -Name "<pool-name>"

# View event logs
Get-EventLog -LogName "Application" -Source "IIS" -Newest 100
```

### Communication During Incidents

- **Slack Channel:** #pbm-incidents (real-time updates)
- **Status Page:** [Company status page URL]
- **Email Notifications:** Critical incidents email distribution list
- **Update Frequency:** Every 15-30 minutes for P1/P2 issues

---

## Document Information

| Item | Value |
|------|-------|
| **Document Title** | Public Mart Application Handover Document |
| **Version** | 1.0 |
| **Date Created** | April 22, 2026 |
| **Last Updated** | April 22, 2026 |
| **Prepared for** | Production Support Engineering Team |
| **Classification** | Internal Use Only |
| **Distribution** | Limited to authorized support staff |

### Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | April 22, 2026 | Claude Assistant | Initial handover document created |

### Next Steps

To complete this handover document, please provide:

1. Specific database names and connection details
2. Backend framework and version details
3. Message queue system details (RabbitMQ/Kafka)
4. Logging platform (ELK Stack/Splunk) setup information
5. Monitoring tool endpoints and dashboards
6. Team contact information and escalation contacts
7. Specific maintenance windows and blackout periods
8. Network topology and firewall rules
9. Disaster recovery and business continuity plans
10. SLA definitions and support metrics

---

**This document is confidential and intended for authorized personnel only. Unauthorized distribution is prohibited. Please handle with appropriate security measures.**
