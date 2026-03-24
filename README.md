# VizDSGraph


/codebook
   codebook_v1.csv
   codebook_v2.csv
   codebook_change_log.md

/protocol
   coding_protocol.pdf
   sampling_strategy.pdf

/examples
   anonymized_excerpt_01.pdf
   coded_example_matrix.csv

/kg_mapping
   code_to_domain_parameter_mapping.csv
   neo4j_schema_snapshot.cypher


## Project dashboard
Bulit using NeoDash.

Fork this repository, use NeoDash or purchase (https://neo4j.com/docs/neodash-commercial/current/#_getting_access_to_neodash_commercial) a NeoDash commercial license together with a Neo4j Enterprise license.  

# Architecture Summary

## Overview

The dashboard uses NeoDash as a client-side dashboard builder for Neo4j. The main application is a React 17 + Redux SPA that lets users connect to a Neo4j database, compose dashboards from reusable report cards, run Cypher queries, and render the results through a chart/report registry.

At a high level, the system is organized around four layers:

1. Application shell: startup, configuration loading, connection lifecycle, modal orchestration, and global UI state.
2. Dashboard model: dashboard metadata, pages, cards, report settings, and persisted user/session state.
3. Query/report runtime: Cypher execution, result parsing, schema extraction, refresh behavior, and chart rendering.
4. Extension surface: optional report types and behaviors such as advanced charts, forms, rule-based styling, report actions, and text-to-Cypher.

## Repository Layout

### Main app

- [`src/`](./src) contains the production NeoDash application.
- [`src/application/`](./src/application) owns app bootstrapping, connection state, modal state, and app-level thunks/actions/selectors.
- [`src/dashboard/`](./src/dashboard) owns the main dashboard frame, header, sidebar, and dashboard-level state transitions.
- [`src/page/`](./src/page) models dashboard pages and page-level actions.
- [`src/card/`](./src/card) models grid cards, their settings panes, and card-level behavior.
- [`src/report/`](./src/report) executes queries and bridges report configuration to chart components.
- [`src/chart/`](./src/chart) contains built-in visualizations such as table, graph, line, bar, pie, map, markdown, iframe, JSON, and parameter selection.
- [`src/extensions/`](./src/extensions) adds optional report types and cross-cutting features.
- [`src/config/`](./src/config) defines report metadata, card settings, style configuration, example content, and other static configuration.
- [`src/modal/`](./src/modal) contains connection, onboarding, about, import/export, share, and notification dialogs.
- [`src/utils/`](./src/utils) contains shared helpers, including proxy-aware Cypher execution utilities.

### Supporting subprojects

- [`server/`](./server) is a small Express-based Cypher proxy for deployments that should not expose direct browser-to-Neo4j connectivity.
- [`docs/`](./docs) is an Antora documentation site containing the user and developer guides.
- [`gallery/`](./gallery) is a separate React app that showcases example dashboards.
- [`cypress/`](./cypress) contains end-to-end tests.

## Runtime Architecture

### 1. Application startup

The entrypoint in [`src/index.tsx`](./src/index.tsx) initializes:

- the Redux store from [`src/store.ts`](./src/store.ts)
- `redux-persist` for browser-side state persistence
- style configuration and global CSS
- Sentry only for the hosted `neodash.graphapp.io` deployment

`Application` is the top-level runtime container. It loads application configuration, manages connection and onboarding state, and decides whether to show the dashboard or placeholder UI.

### 2. State management

The app uses Redux with thunk middleware and persisted browser storage.

The top-level state is split into:

- `dashboard`: the active dashboard definition, including title, settings, pages, reports, and dashboard parameters
- `application`: connection details, modal visibility, standalone mode settings, SSO flags, notifications, and other app-level UI state
- `sessionStorage`: transient session-only state used for runtime helpers and extension workflows

This split is important architecturally:

- `application` answers "what state is the app shell in?"
- `dashboard` answers "what dashboard is currently loaded and how is it configured?"
- `sessionStorage` answers "what temporary runtime values should not become part of the saved dashboard?"

### 3. Connection and query execution

`Dashboard` creates the Neo4j access layer and exposes it through `use-neo4j`.

There are two connection modes:

- Direct driver mode: the browser connects to Neo4j using `use-neo4j` and a live Neo4j driver.
- Proxy mode: the browser talks to the local proxy service, which executes read-only Cypher server-side and returns serialized results.

The proxy path is implemented by:

- [`server/index.js`](./server/index.js) on the server side
- [`src/utils/CypherProxy.ts`](./src/utils/CypherProxy.ts) on the client side

The proxy deliberately blocks write-oriented or unsafe query patterns by default and serializes Neo4j values into JSON-safe payloads that the client rehydrates back into driver-compatible values.

### 4. Dashboard composition

The UI hierarchy follows this shape:

`Application` -> `Dashboard` -> `Page` -> `Card` -> `Report` -> `Chart`

Responsibilities are separated as follows:

- `Dashboard` owns navigation, connection updates, page chrome, and the left sidebar.
- `Page` renders the current page and its report-card layout.
- `Card` owns placement, titles, settings, fullscreen/download controls, and report container behavior.
- `Report` owns query execution, loading/error/no-data states, field detection, and refresh timers.
- `Chart` components only render a specific visualization given normalized data and settings.

This separation is one of the project’s main strengths: layout logic, query logic, and visualization logic are decoupled enough to support many chart types without duplicating the dashboard shell.

### 5. Report and chart registry

Built-in report types are registered centrally in [`src/config/ReportConfig.tsx`](./src/config/ReportConfig.tsx). Each report type defines:

- label and helper text
- the React component used to render it
- selection requirements
- supported settings
- record limits and rendering behavior

`Report` uses this registry to determine:

- whether a report is text-only or query-backed
- which fields should be inferred from the result set
- which chart component should receive the processed data

This registry-driven design makes new visualizations relatively straightforward to add without rewriting the report runtime.

## Extension Model

Extensions are registered in [`src/extensions/ExtensionConfig.tsx`](./src/extensions/ExtensionConfig.tsx) and activated dynamically.

Current extensions include:

- advanced visualizations
- forms
- rule-based styling
- report actions
- text-to-Cypher / query translator

Extensions can contribute one or more of:

- additional report types
- custom reducers
- settings UI
- card settings components
- drawer/menu buttons
- pre-population logic that modifies query generation before a report runs

This gives NeoDash a plugin-like architecture without a separate plugin runtime. The extension surface is compile-time integrated, but runtime-configurable.

## Deployment Model

The main app is built with Webpack and emitted as a static bundle. The root [`Dockerfile`](./Dockerfile) uses a multi-stage build:

- build the frontend with Node
- serve the generated static assets from Nginx

[`compose.yaml`](./compose.yaml) exposes the containerized app on port `5005`.

For production setups that need a backend middle tier, the `server` subproject can be deployed alongside the frontend to provide a read-oriented Cypher proxy.

## Testing and Docs

- Cypress end-to-end tests live in [`cypress/`](./cypress).
- Developer and user documentation are maintained in [`docs/`](./docs) using Antora.
- The gallery app in [`gallery/`](./gallery) acts as a showcase for sample dashboards and example content.

## Key Architectural Takeaways

- NeoDash is primarily a frontend application with optional backend support through a Cypher proxy.
- The core domain model is dashboard -> page -> card -> report -> chart.
- Redux cleanly separates saved dashboard state from application shell state and transient session state.
- Query execution is centralized in the report layer, which keeps chart components focused on presentation.
- Report types and extensions are registry-driven, which is the main mechanism that makes the system extensible.
- Docs, gallery, proxy, and main UI are kept as separate subprojects, which reduces coupling between product, documentation, and deployment concerns.


