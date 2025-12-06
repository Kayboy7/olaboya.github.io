# CORA Transit Architecture Blueprint

## 1. System Context
- **Primary external systems**: 
  - **Wiatag (driver mobile app)** pushes GPS/telemetry to **Wialon Hosting**.
  - **Wialon Hosting** exposes APIs and webhooks consumed by CORA Transit Integration Layer; maintains trackers/geofences during pilot.
- **Organizations & users** interact via CORA Transit web dashboards/APIs:
  - **C.O.R.A Technologies** (platform owner/admin) manages tenants, onboarding, SLA/ops.
  - **ECOWAS** (Directors of Transportation, Free Trade, Customs) consume read-only analytics and corridor KPIs; future SSO with ECOWAS identity.
  - **OCAL** monitors Abidjan–Lagos corridor performance, border posts, incidents.
  - **UCRAO** monitors member drivers and supports incident resolution.
  - **Fleet owners/transport companies** monitor their trucks/trips, receive alerts, manage operations.
  - **Drivers** (via Wiatag now, future CORA driver app) start/stop trips, send events, location.
  - **National agencies** (customs, transport, trade) receive read-only views or API feeds for country-specific monitoring.
- **Data flows**:
  - GPS/telemetry: Driver (Wiatag) → Wialon → CORA Transit Integration → Core Transit Platform → Dashboards/APIs.
  - Operational input (trip plans, incidents): Fleet ops / C.O.R.A / future driver app → Core Transit Platform.
  - Outputs: dashboards (web), alerts/notifications (email/SMS/push), APIs for future system-to-system exchange.

## 2. Stakeholders & Roles
- **C.O.R.A Technologies**: super-admin, platform ops/DevOps, data steward, support.
- **ECOWAS**: read-only executives (3 dashboards), analytics roles (future), policy/compliance roles.
- **OCAL**: corridor operator (Abidjan–Lagos), read-only pilot dashboard; future operator role for interventions.
- **UCRAO**: union operator/analyst, focused on member drivers; read-only in pilot.
- **Fleet owners/transport companies**: tenant admins, dispatchers/operators, analysts; CRUD trips/assets (MVP limited to their fleets).
- **Truck drivers**: tracked via Wiatag (pilot) with trip start/stop signals and incident reporting; future CORA driver app.
- **National agencies** (future): customs/transport/trade observers, risk/insights users.

## 3. Logical Architecture
### 3.1 Integration Layer
- **Wialon Connector**: ingest units/telemetry via API/webhooks; translate to CORA LocationPoint & Trip context; geofence/border crossing detection using Wialon geofences initially.
- **Wiatag Connector**: leverage Wialon as hub; future direct SDK/Webhook ingestion for redundancy.
- **External Connectors (future)**: customs single windows, port community systems, weighbridge/ANPR feeds, road safety/traffic APIs.
- **Data ingestion patterns**: streaming collector (WebSocket/webhook), polling fallback, buffering/retry for intermittent connectivity, idempotent processing with message IDs.

### 3.2 Core Transit Platform
- **Trip & Corridor Management**: define corridors, segments, checkpoints/border posts; create trips with planned route and schedule; detect segment entry/exit and border crossings; compute ETA.
- **Fleet & Asset Management**: trucks, trailers, ownership, driver assignments; compliance attributes (licenses, permits, insurance validity dates).
- **Events & Incidents**: stop events, dwell, delays, breakdowns, congestion/border queues; severity and category taxonomy.
- **Notifications & Alerts**: rules engine (e.g., ETA deviation, long dwell, off-route, device offline); multi-channel delivery (email/SMS/push/webhooks).
- **Geospatial Services**: map tiles, geofencing, route snapping; offline-aware caching of geofences.
- **Data Processing**: streaming enrichment (join telemetry with trips), analytics aggregation jobs (hourly/daily), data lake for history.

### 3.3 User & Organization Management
- **Multi-tenant hierarchy**: root tenant (C.O.R.A), regional tenant (ECOWAS), corridor orgs (OCAL), associations (UCRAO), companies, national agencies.
- **RBAC**: roles per tenant with scopes (corridor, fleet, driver set); inheritance for sub-roles; read-only vs operational roles.
- **Identity**: password/OIDC auth; SSO-ready for ECOWAS/large partners.

### 3.4 Analytics & Reporting Layer
- **Dashboards**: 
  - ECOWAS: macro corridor KPIs (transit time, border dwell, reliability, trip volumes, heatmaps of stops/incidents).
  - OCAL: corridor operational performance, segment-level dwell, incident watchlist.
  - UCRAO: member driver activity, safety/compliance incidents.
  - Fleet owners: trip status board, SLA/ETA adherence, driver-level performance.
- **Analytics store**: columnar warehouse (e.g., ClickHouse/BigQuery/Snowflake depending on hosting); semantic layer defining KPIs.
- **Reporting APIs**: for external consumption and embedding.

### 3.5 API Layer
- **External APIs**: REST/GraphQL for trip creation, asset management, telemetry queries, and analytics; webhooks for alerts.
- **Internal APIs**: service-to-service via gRPC/REST with service mesh for policy.

### 3.6 Presentation Layer
- **Web consoles**: responsive dashboards for ECOWAS/OCAL/UCRAO (read-only pilot), operations console for C.O.R.A/fleet operators.
- **Driver interface**: Wiatag during pilot; future CORA driver mobile app (offline-first, multilingual).

## 4. Data Model (Conceptual)
- **Country**: code, name, language(s), timezone.
- **Corridor**: name, countries covered; relations to CorridorSegment.
- **CorridorSegment**: start/end locations, country, road ID, typical speed, risk score.
- **BorderPost**: country pair, location, operating hours, typical dwell stats.
- **Checkpoint/Geofence**: polygon/point, type (border, toll, rest stop), linked to corridor segment.
- **Organization**: type (CORA, ECOWAS, CorridorOrg, Union, Company, Agency), parent relationships, allowed corridors.
- **User**: identity, language, org membership, roles.
- **Role/Permission**: CRUD scopes on entities; read-only flag; corridor/fleet scoping.
- **Driver**: name, license, contact, union membership, assigned organization.
- **Truck**: VIN/plate, capacity, type, owner (Organization), linked devices (Wialon unit ID), fuel type.
- **Trailer**: plate, capacity, owner; linked to truck for a trip.
- **Trip**: truck, driver, corridor, planned route (segments/checkpoints), schedule, cargo summary, status, ETA, start/end timestamps.
- **TripLeg**: segment-level plan/actual times.
- **LocationPoint**: timestamp, lat/long, speed, heading, unit ID, source (Wialon/Wiatag), accuracy.
- **StopEvent**: detected stop with location, duration, classification (planned/unplanned, border, rest, queue).
- **DelayEvent/Incident**: category (breakdown, customs, traffic, safety), severity, description, attachments.
- **Notification**: rule ID, target users/webhooks, delivery status.
- **Document (future)**: type (manifest, permit), issuer, validity, attachments.

## 5. Multi-Tenant & Access Control Design
- **Tenant isolation**: each Organization is a tenant with data partition keys (org_id) and corridor scopes; row-level security in DB/warehouse; per-tenant encryption keys where possible.
- **Visibility rules**:
  - **ECOWAS**: regional tenant with cross-corridor scope; sees aggregated data across countries/corridors; drill-down allowed but no asset ownership changes.
  - **OCAL**: corridor-scoped to Abidjan–Lagos; sees all trips/assets on that corridor regardless of owner, but only within corridor geography.
  - **UCRAO**: union-scoped to member drivers/fleets; sees trips for drivers tagged to union; corridor-limited in pilot.
  - **Fleet owners**: organization-scoped; only own trucks/drivers/trips; optional sharing with OCAL/ECOWAS via data-sharing policies.
  - **C.O.R.A**: super-admin across tenants; ops and support.
- **RBAC modeling**: roles (Admin, Operator, Analyst, ReadOnly) with scopes (corridor, fleet, driver group); permissions mapped to resources (Trip, Asset, Analytics, Alerts). Read-only dashboards use roles with view-only permissions and preconfigured layouts.
- **Provisioning read-only dashboards**: create tenant-specific dashboard bundles; assign users to ReadOnly role; limit API tokens to GET/analytics endpoints; enforce via UI components and API gateway policies.

## 6. Integrations (Wiatag / Wialon and Future Systems)
- **Telemetry ingestion**: Wiatag → Wialon (unit IDs) → CORA webhook/API polling; store raw points, enrich with trip context; detect geofence crossings (border posts, checkpoints) via Wialon geofences initially.
- **Entity mapping**:
  - Wialon Unit → CORA Truck (with device_id mapping).
  - Wialon Geofence → CORA Checkpoint/BorderPost (sync geofence definitions).
  - Wialon Routes (if used) → CORA CorridorSegment/TripLeg templates.
- **Division of responsibilities**:
  - **Remain in Wialon during pilot**: device management, SIM/APN config, baseline tracking reliability, geofence storage, Wiatag device provisioning.
  - **Handled by CORA**: trip orchestration, RBAC, analytics, dashboarding, alert rules, corridor modeling, data sharing policies.
- **Evolution path**: progressively replicate geofences/routes into CORA geospatial service; add direct Wiatag/CORA driver app ingestion; migrate device management to CORA-compatible telematics hub if needed.

## 7. User Journeys (Pilot)
- **Driver (Wiatag)**: receives trip assignment; starts tracking in Wiatag; travels along corridor; border crossing detected via geofence; unplanned stop triggers incident report (message in Wiatag or dispatcher call); arrival auto-detected or manually confirmed; app works offline with buffered points.
- **Fleet Owner/Dispatcher**: logs into CORA ops console; views live map of own fleet; monitors trip list with status/ETA; receives alerts for off-route/long dwell/device offline; reviews daily KPIs (on-time %, average transit time) and incident log.
- **ECOWAS Director of Transportation (Read-only)**: logs in; selects Abidjan–Lagos corridor; sees live map with trucks; views KPIs (trip counts, average transit time, reliability, incident heatmap); drills into border performance.
- **ECOWAS Director of Free Trade (Read-only)**: accesses trade facilitation dashboard showing average transit time, variability, delays by border post, top bottlenecks, historical trends over pilot period.
- **ECOWAS Director of Customs (Read-only)**: views border dwell times, queue indicators, risk flags (frequent prolonged stops near borders), per-border post rankings.
- **OCAL**: corridor operations dashboard focusing on segment performance, active incidents, border queues; can download weekly reports for the corridor authority.
- **UCRAO**: union dashboard showing member driver locations, active trips, incident reports, and safety/compliance alerts affecting union members.

## 8. MVP Scope (April 2026)
- Abidjan–Lagos corridor support with 11+ pilot trucks; onboarding additional fleets.
- Wialon/Wiatag integration for telemetry; mapping of units to trucks and corridor geofences.
- Core trip management with segment/border detection; ETA and dwell calculation.
- Multi-tenant RBAC with read-only dashboards for ECOWAS (3), OCAL (1), UCRAO (1); ops console for CORA and fleet owners.
- Basic analytics: transit time, border dwell, stop counts, incident summaries, live map.
- Notifications: off-route, long dwell/border queue, device offline, ETA deviation.
- Bilingual UI (EN/FR), responsive web; offline-tolerant ingestion pipeline.

## 9. Future Roadmap
- Add additional ECOWAS corridors and countries; corridor templates and geofence libraries.
- Direct CORA driver app with richer event capture (photos/docs), chat, and offline workflows.
- Deep integrations: customs single windows, port/terminal systems, weighbridges/ANPR; document exchange (e-permits, manifests).
- Advanced analytics/AI: predictive border wait times, risk scoring, route optimization, CO2 estimation.
- Partner SSO and data residency controls per country; per-tenant encryption keys/HSM.
- Self-service onboarding for fleets and unions; marketplace for telematics connectors beyond Wialon.

## 10. Security, Compliance & Data Governance
- **AuthN/AuthZ**: OAuth2/OIDC with JWT access tokens; identity providers per tenant; MFA for admins; API gateway enforcing scopes and rate limits.
- **Encryption**: TLS 1.2+ in transit; at-rest encryption (disk-level + column/field for sensitive data like driver PII); secrets in vault (e.g., HashiCorp Vault/KMS).
- **Auditability**: audit logs for authentication, data access, configuration changes; immutable storage and retention policies aligned with ECOWAS data rules.
- **Data residency/sovereignty**: deploy in regional cloud (e.g., Azure West Africa/North Europe as DR); partition data by country/tenant; configurable retention and data export for regulators.
- **Monitoring & logging**: centralized logs/metrics/traces (ELK/Opensearch + Prometheus/Grafana); alerting for ingestion lag, API errors, security events.

## 11. Non-Functional Requirements
- **Intermittent connectivity**: ingestion buffering, retry with backoff; UI shows last-updated timestamps; offline caching for driver app (future).
- **Scalability**: design for 10k+ trucks, 100k+ trips/month, 500+ concurrent users; autoscaling ingestion and analytics; partitioned telemetry storage (time-series DB or data lake).
- **Performance**: sub-2s dashboard loads for common queries with caching and pre-aggregations; real-time map updates within 15–30s latency.
- **Availability & DR**: 99.5%+ target for pilot/MVP; multi-AZ deployment, backups, RPO ≤ 1h, RTO ≤ 4h.
- **Maintainability/Extensibility**: modular services (integration, core, analytics, UI); config-driven corridor/role definitions; API-first with versioning.
- **I18n**: English/French baseline; locale-specific formats (time, units); extensible to Portuguese/Arabic as needed.
- **Compliance**: adhere to regional data protection norms; configurable consent for driver data where required.
