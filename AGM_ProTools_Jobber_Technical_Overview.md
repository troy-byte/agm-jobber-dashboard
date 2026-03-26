# AGM Pro Tools — Jobber API Technical Overview

**Prepared for:** Jobber Marketplace Team
**Prepared by:** Troy Scott, AGM Pro Tools
**Date:** March 27, 2026

---

## Overview

AGM Pro Tools is an automation layer that connects to Jobber via OAuth 2.0 and the GraphQL API. It listens for Jobber webhook events and takes action on behalf of the contractor — follow-ups, pipeline management, review requests — without the end user logging into any additional software.

This document outlines our API usage, data handling, authentication model, and security posture for marketplace evaluation purposes.

---

## Authentication

### OAuth 2.0 Flow
- **Grant type:** Authorization Code
- **Authorization endpoint:** `https://api.getjobber.com/api/oauth/authorize`
- **Token endpoint:** `https://api.getjobber.com/api/oauth/token`
- **Redirect URI:** Localhost callback during onboarding (guided setup with contractor)
- **Token storage:** Encrypted at rest, per-client isolation (each contractor has their own token set)
- **Token refresh:** Automatic — access tokens refreshed before expiry with a 5-minute safety buffer
- **Refresh token rotation:** Handled atomically — new refresh tokens are saved before old ones are discarded

### OAuth Scopes Requested

| Scope | Purpose |
|-------|---------|
| `read_quotes` | Fetch quote details when webhook events fire |
| `write_quotes` | Reserved for future use (not currently exercised) |
| `read_clients` | Look up existing clients to prevent duplicates |
| `write_clients` | Create new clients and set custom fields |
| `read_jobs` | Fetch job details for post-job automations |
| `write_jobs` | Reserved for future use (not currently exercised) |
| `read_invoices` | Read invoice data for reporting |

We request only the scopes required for our workflows. Two scopes (`write_quotes`, `write_jobs`) are reserved but not currently exercised — we do not modify quotes or jobs in Jobber.

---

## Webhook Events Consumed

AGM Pro Tools registers webhook endpoints via the `webhookEndpointCreate` GraphQL mutation. All incoming webhooks are verified using HMAC-SHA256 signature validation against the app's client secret.

| Event | What We Do |
|-------|------------|
| `QUOTE_SENT` | Fetch quote details, update CRM pipeline stage, schedule follow-up sequence |
| `QUOTE_APPROVED` | Fetch quote details including line items, update CRM to won stage |
| `QUOTE_CREATE` | Track new quotes entering the system |
| `QUOTE_UPDATE` | Detect status changes for quotes in progress |
| `JOB_CLOSED` | Fetch job details, trigger post-job review request workflow |

All events are deduplicated — each event is processed exactly once, tracked by event ID with a 30-day retention window.

---

## GraphQL Operations

### Queries (Read Operations)

| Operation | Entity | Fields Read | Purpose |
|-----------|--------|------------|---------|
| GetQuote | Quote | ID, number, status, title, amounts, client info, line items | Process quote events |
| GetJob | Job | ID, number, title, status, total, client info | Process job completion events |
| SearchClients | Client | ID, name, emails, company | Duplicate prevention before creating clients |
| GetClientCustomFields | Client | Custom field IDs and labels | Map custom field configurations |

### Mutations (Write Operations)

| Operation | Entity | Fields Written | Purpose |
|-----------|--------|---------------|---------|
| clientCreate | Client | First name, last name, email, phone, company | Create client record from CRM lead |
| clientEdit | Client | Custom field values | Set custom field (e.g., lead source tracking) |
| propertyCreate | Property | Street address, city, state, zip | Create service location for new client |

**We do not modify quotes, jobs, or invoices.** Jobber remains the source of truth for all job management data. Our writes are limited to client creation and custom field tagging.

---

## Data Flow Summary

```
Jobber (source of truth)
  │
  ├── Webhook events ──→ AGM Pro Tools ──→ CRM pipeline updates
  │                                    ──→ Follow-up sequences
  │                                    ──→ Review request triggers
  │
  └── GraphQL reads ───→ Quote details, job details, client lookup

CRM (lead source)
  │
  └── New lead tag ────→ AGM Pro Tools ──→ Jobber client creation
```

**Jobber → AGM direction:** Read-only. We fetch quote, job, and client data to inform downstream automations.

**AGM → Jobber direction:** Limited writes. We create clients, set custom fields, and create properties. We never modify existing quotes, jobs, or invoices.

---

## Rate Limit Compliance

- **Retry strategy:** Exponential backoff (1s, 2s, 4s, 8s) on transient errors (429, 500, 502, 503, 504)
- **429 handling:** Respects `Retry-After` header from Jobber's API response
- **Connection errors:** Automatic retry with backoff, up to 4 attempts
- **No bulk operations:** All API calls are event-driven (triggered by webhooks), not batch/polling. This keeps API usage proportional to actual business activity.

---

## Multi-Tenant Architecture

Each contractor operates in full isolation:

- **Separate OAuth tokens** — each contractor authorizes independently
- **Separate configuration** — pipeline stages, custom fields, and webhook secrets are per-client
- **Separate event tracking** — deduplication is scoped per contractor
- **No cross-client data access** — one contractor's token cannot access another contractor's Jobber account

---

## Security Posture

| Layer | Implementation |
|-------|---------------|
| **Webhook verification** | HMAC-SHA256 signature validation on every incoming Jobber webhook |
| **Token storage** | Per-client isolation, encrypted at rest, atomic writes to prevent corruption |
| **Credential management** | No credentials stored in source code; all secrets managed via secure environment |
| **Pre-commit scanning** | Automated hooks prevent accidental credential commits |
| **Transport** | All API communication over HTTPS/TLS |
| **Error handling** | Failed events are logged and retried — no data is silently dropped |

---

## Current Scale

| Metric | Value |
|--------|-------|
| Active Jobber integrations | 1 (Texas Turf Company — live production) |
| Quotes processed | 3,744+ |
| Jobs tracked | 1,803+ |
| Webhook events handled | 4,200+ |
| Uptime | Zero unplanned downtime since deployment |

---

## Infrastructure

- **Hosting:** Modal (serverless Python, auto-scaling)
- **Webhook endpoints:** HTTPS with TLS, publicly routable
- **Token persistence:** Encrypted volume storage
- **Monitoring:** Real-time alerting on webhook failures, automated error classification

---

## Contact

**Troy Scott** — Troy@agmprotools.com
**Andrew Croot** — Andrew@artificialgrassmarketing.com
