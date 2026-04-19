# Roof Segmentation System — Project Timeline Estimate
> **Method:** Progressive Estimation · PERT Three-Point (O = Optimistic, M = Most Likely, P = Pessimistic)
> **Formula:** `E = (O + 4M + P) / 6` · **Std Dev:** `σ = (P − O) / 6`
> **Scope:** Approach 1 — Google Solar API + React Canvas Editor (v1)
> **Team mode:** Hybrid — 1–2 developers with AI-assisted coding (~60% of implementation via agent)
> **Date of estimate:** 2026-04-18
> **All values in working days (1 day = 8 hours)**

---

## Estimation Assumptions

| Parameter | Value |
|---|---|
| Team size | 1 senior fullstack developer + 1 AI coding agent |
| Agent uplift multiplier | ×0.55 on implementation tasks (research-backed: AI agents reduce hands-on coding ~40–55%) |
| Working days / week | 5 |
| Confidence target for commitments | **P75** (not P50 — per PERT best practice) |
| India geography v1 | Yes — affects geocoding error rate assumption |
| Auth complexity | Minimal — internal tool, API key / cookie auth only |

---

## Work Breakdown Structure + PERT Table

### Phase 0 — Project Setup & Infrastructure

| Task | O | M | P | **E** | σ | Notes |
|---|---|---|---|---|---|---|
| 0.1 Repo scaffold (monorepo, CI skeleton, Docker-compose) | 0.5 | 1.0 | 1.5 | **1.0** | 0.17 | Agent generates boilerplate |
| 0.2 Google API key provisioning + secret management setup | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Depends on GCP project access |
| 0.3 Redis / in-memory cache setup | 0.5 | 0.5 | 1.0 | **0.58** | 0.08 | Simple for MVP (dict cache) |
| 0.4 Postgres or S3 correction log schema + migrations | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | |
| **Phase 0 Total** | | | | **3.75** | **0.40** | |

---

### Phase 1 — Backend BFF (FastAPI)

| Task | O | M | P | **E** | σ | Notes |
|---|---|---|---|---|---|---|
| 1.1 Address geocoding module (Google Maps Geocoding API, error handling) | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Includes ambiguous/failed geocode path |
| 1.2 Solar API `buildingInsights` client + 404 / no-building error path | 0.5 | 1.5 | 3.0 | **1.58** | 0.42 | API schema parsing + edge cases |
| 1.3 Solar API `dataLayers` GeoTIFF fetch | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | HTTP streaming of large GeoTIFF |
| 1.4 GeoTIFF → PNG conversion (`rasterio` / `rio-cogeo`) | 0.5 | 1.5 | 3.5 | **1.67** | 0.50 | Rasterio setup has quirks on different OS |
| 1.5 TTL cache layer (24h, keyed on rounded lat/lng) | 0.5 | 1.0 | 1.5 | **1.0** | 0.17 | Redis or in-memory |
| 1.6 Initial area calculation from API polygons (Python/Shapely) | 0.5 | 1.0 | 1.5 | **1.0** | 0.17 | |
| 1.7 `POST /analyse` endpoint (orchestrates 1.1–1.6, full error paths) | 1.0 | 2.0 | 4.0 | **2.17** | 0.50 | Integration of all modules |
| 1.8 `POST /corrections` endpoint + correction log persistence | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Append-only write |
| 1.9 API unit + integration tests | 1.0 | 2.0 | 3.5 | **2.08** | 0.42 | Mocked Google API responses |
| **Phase 1 Total** | | | | **12.75** | **0.97** | |

---

### Phase 2 — Frontend (React)

| Task | O | M | P | **E** | σ | Notes |
|---|---|---|---|---|---|---|
| 2.1 Project scaffold (Vite + React + TypeScript + routing) | 0.5 | 0.5 | 1.0 | **0.58** | 0.08 | Agent-generated |
| 2.2 Address input form + validation + BFF call | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | |
| 2.3 Satellite PNG image viewer component | 0.5 | 1.0 | 1.5 | **1.0** | 0.17 | Plain `<img>` tag from BFF URL |
| 2.4 Konva.js canvas setup + polygon overlay rendering | 1.0 | 2.0 | 4.0 | **2.17** | 0.50 | Konva coordinate mapping is fiddly |
| 2.5 Vertex drag-edit interaction (move, add, delete vertex) | 1.0 | 2.5 | 5.0 | **2.67** | 0.67 | Core UX complexity of the product |
| 2.6 Turf.js live area recalculation on edit | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | |
| 2.7 Manual draw mode (fallback for no-API-data) | 1.0 | 2.0 | 4.0 | **2.17** | 0.50 | Required per ADR-10 |
| 2.8 Error state UI (geocoding fail, no building, low-confidence, timeout) | 0.5 | 1.5 | 3.0 | **1.58** | 0.42 | 4 distinct error paths (ADR-09) |
| 2.9 Corrections POST on save | 0.5 | 0.5 | 1.0 | **0.58** | 0.08 | Simple API call |
| 2.10 Area display + imagery date display | 0.5 | 0.5 | 1.0 | **0.58** | 0.08 | |
| 2.11 Basic responsive layout + internal-tool polish | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Not a consumer app |
| 2.12 Frontend unit tests (canvas interaction, Turf.js area) | 1.0 | 1.5 | 3.0 | **1.67** | 0.33 | |
| **Phase 2 Total** | | | | **16.25** | **1.09** | |

---

### Phase 3 — Integration, QA & Hardening

| Task | O | M | P | **E** | σ | Notes |
|---|---|---|---|---|---|---|
| 3.1 End-to-end integration testing (real addresses, India) | 1.0 | 2.0 | 4.0 | **2.17** | 0.50 | Real API calls may have surprises |
| 3.2 API cost measurement + cache hit-rate validation | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Verify TTL cache working |
| 3.3 Security review — API key secret management, no browser exposure | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Quick audit vs ADR-06 |
| 3.4 Error path QA (all 4 error states, edge addresses) | 0.5 | 1.5 | 3.0 | **1.58** | 0.42 | |
| 3.5 Performance check (p95 response < 30s) | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | GeoTIFF conversion is the likely bottleneck |
| 3.6 Bug fixes from QA round | 1.0 | 2.0 | 4.0 | **2.17** | 0.50 | Always underestimated |
| **Phase 3 Total** | | | | **9.17** | **0.83** | |

---

### Phase 4 — Deployment & Launch

| Task | O | M | P | **E** | σ | Notes |
|---|---|---|---|---|---|---|
| 4.1 Dockerfile (backend + frontend) | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | |
| 4.2 Cloud deployment (AWS ECS / GCP Cloud Run) + env var config | 0.5 | 1.5 | 3.0 | **1.58** | 0.42 | First cloud deploy always has surprises |
| 4.3 Basic CI/CD pipeline (GitHub Actions: lint → test → deploy) | 0.5 | 1.5 | 3.0 | **1.58** | 0.42 | |
| 4.4 Internal user acceptance testing with 2–3 solar agents | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Feedback-driven iteration |
| 4.5 Documentation (API README + user guide for agents) | 0.5 | 1.0 | 2.0 | **1.08** | 0.25 | Internal tool — minimal docs |
| **Phase 4 Total** | | | | **6.42** | **0.66** | |

---

## Project-Level Summary

| Phase | E (days) | σ | P50 | P75 | P90 |
|---|---|---|---|---|---|
| Phase 0 — Setup | 3.75 | 0.40 | 3.75 | 4.02 | 4.26 |
| Phase 1 — Backend | 12.75 | 0.97 | 12.75 | 13.40 | 13.99 |
| Phase 2 — Frontend | 16.25 | 1.09 | 16.25 | 16.98 | 17.65 |
| Phase 3 — QA & Hardening | 9.17 | 0.83 | 9.17 | 9.73 | 10.23 |
| Phase 4 — Deployment | 6.42 | 0.66 | 6.42 | 6.86 | 7.27 |
| **Project Total** | **48.33** | **1.72** | **48.33** | **49.49** | **50.53** |

> **Combined σ (independence assumed):** `√(0.40² + 0.97² + 1.09² + 0.83² + 0.66²) = 1.72 days`

---

## Calendar Duration (Working Weeks)

| Confidence | Working Days | Working Weeks | Calendar Weeks |
|---|---|---|---|
| **P50** (internal target) | 48.3 | 9.7 | ~11 weeks |
| **P75** (team commitment) | 49.5 | 9.9 | ~11.5 weeks |
| **P90** (stakeholder promise) | 50.5 | 10.1 | **~12 weeks** |

> Use **P75 (~10 weeks / 2.5 months)** as your committed delivery date.
> Use **P90 (~12 weeks / 3 months)** when communicating to stakeholders.

---

## Gantt — Indicative Schedule

```
Week  1   2   3   4   5   6   7   8   9   10  11  12
      |---|---|---|---|---|---|---|---|---|---|---|---|
P0    [===]
P1        [==========]
P2             [==================]
P3 (parallel with late P2)       [========]
P4                                         [======]
Buffer                                              [=]
```

> Phases 2 and 3 overlap intentionally — QA starts once core canvas is functional (~Week 6).

---

## Risk Register

| Risk | Likelihood | Impact | Mitigated E Δ | Mitigation |
|---|---|---|---|---|
| Rasterio / GeoTIFF conversion breaks on cloud Linux | Medium | High | +2 days | Test in Docker container from day 1 |
| Google Solar API unavailable for Indian addresses (rural / new builds) | High | Medium | +1 day | Manual draw fallback (ADR-10) already in scope |
| Konva.js vertex editing coordinate mismatch with lat/lng overlay | Medium | High | +2 days | Spike on Konva + Solar API polygon projection in Week 1 |
| GCP billing overrun on Solar API before cache is implemented | Low | Medium | +0.5 day | Implement cache (Phase 1.5) before enabling real API keys |
| Cloud deployment environment mismatch | Medium | Low | +1 day | Dockerize from day 1 (Phase 0.1) |
| **Worst-case total risk buffer** | | | **+6.5 days** | |

> **Absolute worst-case P90 + all risks triggered:** 50.5 + 6.5 = **57 days (~11.5 weeks / ~3 months)**

---

## Velocity Calibration Notes

These estimates assume:
- **AI agent uplift**: ~40–55% reduction on boilerplate, API client code, test scaffolding
- **AI agent NO uplift** on: Konva.js coordinate math, GeoTIFF raster conversion, first-time cloud deploy
- **Senior developer assumption**: Already familiar with React, FastAPI, Docker; new to Solar API

**Calibration feedback:** After Phase 1 completion, compare actuals vs E values. If Phase 1 actual > 16 days (E=12.75, P90=14), re-estimate Phase 2 upward by the same ratio. This is the calibration feedback loop the progressive estimation method requires.

---

## v2 Sizing (Indicative — Approach 2 Custom CV)

| v2 Track | Rough Order of Magnitude |
|---|---|
| GPU infra setup (AWS p3 / GCP T4) | 1–2 weeks |
| SAM inference pipeline (zero-shot, no training) | 3–4 weeks |
| SegFormer fine-tuning on SpaceNet dataset | 6–10 weeks |
| Custom dataset labelling (1000 Indian roofs) | 4–6 weeks (parallel) |
| Model → API integration (replacing Solar API) | 1–2 weeks |
| **v2 total (rough)** | **~4–6 months** (parallel to v1 production use) |

> v2 should begin after v1 ships and correction data accumulates (~1–2 months of usage).
