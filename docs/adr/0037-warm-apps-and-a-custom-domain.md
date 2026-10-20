# ADR 0037 — Prod keeps its apps warm and serves its own domain

- **Status:** Proposed
- **Date:** 2026-10-20
- **Deciders:** Luca Martinelli

## Context

[ADR-0034](0034-azure-hosting-with-terraform.md) chose Container Apps so that both apps scale to
zero, and [ADR-0035](0035-single-lean-prod-and-github-actions.md) kept them at zero replicas when
idle. In use, every visit after a quiet spell waits twice: once for the UI to start, then, after
signing in, for the API to start and check the schema (`MigrateAsync` at startup) before the
story list arrives. For an app shown to someone once, that first impression is most of the visit.

The app is also now reachable at its own domain, `storywalker.life` (and `www.`), bound to the UI
from the portal. The API trusts tokens only from the origins in `Clerk__AuthorizedParties`, which
held the Container Apps address alone, so every signed-in request from the new domain was refused
with `401`.

## Decision

We will keep **one replica of the API and one of the UI running at all times** in prod, and
**manage the UI's custom domains in Terraform**, deriving the API's authorized parties from them.

- `api_min_replicas` and `ui_min_replicas` default to 0, so the configuration still scales to zero
  unless an environment opts out; prod sets both to 1. The maximum stays 1 (ADR-0035), so
  migrating at startup remains safe.
- `ui_custom_domains` lists the UI's hostnames. Each gets an Azure-managed certificate and a
  binding, and `https://<hostname>` joins `Clerk__AuthorizedParties`. The apex binding made in the
  portal is imported, not recreated.
- The monthly budget is raised to cover replicas that no longer stop.

## Consequences

### Positive
- No cold start: the first page and the first story list answer as fast as later ones.
- The domain and its certificates are in the configuration: a rebuilt environment gets them back,
  and adding a hostname authorizes it for the API in the same change.

### Negative / trade-offs accepted
- Both apps are billed around the clock, at the idle rate when nobody uses them, instead of only
  while serving.
- PostgreSQL still sits in North Europe (ADR-0035's capacity workaround) while the apps are in
  Sweden Central; each query crosses regions. Warm replicas do not change that.

### Things this decision obliges us to build or watch
- DNS at the registrar: for each hostname the record that points it at the UI and the `asuid` TXT
  record with the environment's verification id (the `custom_domain_verification_id` output),
  before the apply that binds it.
- The budget alerts, now that the fixed part of the bill is larger.

## Alternatives considered

### Keep scaling to zero
Free while idle, and the original choice. Rejected: the wait on every first visit is what users
noticed.

### Warm the API only
Removes the slower of the two waits, but the first page still waits for the UI.

### App Service for the UI
Its free tier sleeps too, and an always-on plan costs more than a warm Container App. A UI outside
the Container Apps environment also cannot reach the API's internal address without a VNet, or
without making the API public.

### Authorize the domain by hand in `extra_authorized_parties`
Fixes the `401` with one line, but leaves the binding and its certificate outside Terraform and
the list of hostnames in two places.
