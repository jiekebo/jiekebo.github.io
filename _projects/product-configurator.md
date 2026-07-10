---
title: Product Configurator
eyebrow: Java EE · 3D Web · Full-Stack
subtitle: >-
  A configurable-product platform — a graph-driven engine resolves which parts
  fit where, a Three.js WebGL viewer renders the assembled product in 3D, and
  the whole thing ships as a Java EE EAR on JBoss with MySQL + MongoDB.
summary: >-
  Java EE 6 (EJB 3.1 + JAX-RS/RESTEasy + JPA/Hibernate) backend deployed as an
  EAR to JBoss AS 7, backed by MySQL (configuration graph) and MongoDB GridFS
  (3D model assets), serving a Backbone + Three.js editor and an embeddable
  configurator widget.
order: 4
cover: /assets/projects/product-configurator/ui.png
tags: [Java EE, EJB, JAX-RS, Hibernate, Three.js, Backbone, MySQL, MongoDB, WebGL]
stack: [JBoss AS 7, EJB 3.1, JAX-RS, Hibernate, MySQL, MongoDB, Three.js, Backbone.js]
---

> Pick a part, see it snap into place in 3D. **Product Configurator** is a
> graph-driven configuration engine: an assembly is a directed graph of nodes
> and edges, each node holds parts, each part references a 3D model asset, and
> connectors on parts encode how pieces fit together. A Backbone + Three.js
> single-page app drives the UX, a Java EE backend walks the graph to resolve
> valid choices, and the result is rendered live in WebGL — all packaged as one
> EAR on JBoss.

## The idea

Let a customer configure a product visually — a table, a Lego figure, a piece
of furniture — and see it assemble in 3D as they choose.

A configurable product is modelled as a **directed graph**: nodes are
configurable decision points, edges define which nodes can follow which, and
each node carries one or more **parts** (the actual selectable items). A part
references a **3D asset** (a Three.js geometry stored in MongoDB GridFS) and
exposes **connectors** — 3D positions and rotations that say where it joins
neighbouring parts. When a user picks a part, the backend walks the graph,
adds the adjacent nodes' parts, aligns them in 3D via the connectors, and
returns both the assembled product and the next set of available choices.

The work splits into three layers:

1. **Model** — an assembly graph editor (d3 force layout) and a part editor
   (Three.js) let an author build the configuration graph, place 3D models and
   define connectors.
2. **Resolve** — a JAX-RS endpoint runs the graph engine: load the in-memory
   graph, apply the user's choices, walk adjacent nodes, align parts in 3D and
   return the assembled product plus the next available choices.
3. **View** — a Three.js WebGL viewer renders the assembled product, and an
   embeddable widget lets the configurator be dropped onto any third-party page
   with a single `<script>` tag.

## System architecture

The diagram shows the full stack as it ships: the JBoss application server
runs a single `Configurator.ear` made of an EJB module (domain, DAOs, JAX-RS
resources, the graph/product engine) and two WAR frontends — the Backbone
editor at `/editor` and the embeddable viewer at `/embed`. MySQL holds the
configuration graph (users, assemblies, nodes, parts, edges, connectors,
assets, settings); MongoDB GridFS holds the 3D model files and textures. The
flagship endpoint `POST /rest/configurator/product` is where a client sends
its choices and gets back the assembled 3D product.

<div class="diagram">
<div class="mermaid">
flowchart TD
  subgraph client["Clients"]
    EDITOR["Author / admin<br/>Backbone SPA · /editor"]
    EMBED["Consumer widget<br/>Backbone + Three.js · /embed<br/>drop-in &lt;script&gt; tag"]
  end

  subgraph jboss["JBoss AS 7.1.1 · Configurator.ear"]
    direction TB
    subgraph fe["WAR frontends"]
      WEDITOR["configurator-web-editor<br/>RequireJS / r.js · Bower<br/>Three.js + d3"]
      WEMBED["configurator-web-embed<br/>Browserify / Gulp<br/>Three.js"]
    end
    subgraph be["configurator-ejb · EJB 3.1"]
      direction TB
      REST["JAX-RS / RESTEasy<br/>/rest/* · Basic-auth + CORS"]
      SEC["SecurityInterceptor<br/>HTTP Basic · PBKDF2<br/>roles: user / admin"]
      DAO["DAO layer · @Stateless<br/>Assembly · Node · Part · Edge<br/>Connector · Asset · User · Setting"]
      GRAPH["GraphLoader<br/>@Startup @Singleton<br/>in-memory graph cache"]
      ENGINE["ProductBuilder + PartAligner<br/>walk graph · align parts in 3D"]
      MONGO["MongoClientProvider<br/>GridFS 3D models + textures"]
    end
  end

  MYSQL[("MySQL 5.6<br/>users · assemblies · nodes<br/>parts · edges · connectors<br/>assets · settings")]
  MONGODB[("MongoDB · GridFS<br/>Three.js model JSON<br/>texture images")]

  EDITOR --> WEDITOR
  EMBED --> WEMBED
  WEDITOR -->|"static SPA"| EDITOR
  WEMBED -->|"static bundle"| EMBED

  EDITOR -->|"/rest/* · Basic auth"| SEC
  EMBED -->|"https://nordlytics.se/editor/rest/*"| SEC
  SEC --> REST
  REST --> DAO
  REST -->|"POST /configurator/product"| ENGINE
  GRAPH --> ENGINE
  DAO <--> MYSQL
  DAO --> MONGO
  MONGO --> MONGO
  ENGINE -->|"ProductResponse<br/>added parts + choices"| REST
  REST -->|"JSON"| EDITOR
  REST -->|"JSON"| EMBED

  subgraph ops["Build & deploy"]
    MVN["Maven multi-module<br/>ejb · web-editor · web-embed · ear"]
    DOCKER["Docker · MySQL + MongoDB"]
  end
  MVN -->|"deploy Configurator.ear"| jboss
  DOCKER --> MYSQL
  DOCKER --> MONGODB
</div>
</div>

## The configuration graph

The whole product model is a directed graph. The diagram below shows how the
entities relate: a user owns assemblies; an assembly has nodes and directed
edges (node → node); each node holds parts; each part references a 3D asset and
exposes connectors that sit on an edge; connectors carry the 3D position and
rotation that the `PartAligner` uses to snap parts together.

<div class="diagram">
<div class="mermaid">
flowchart TD
  USER["UserEntity<br/>PBKDF2 · roles"]
  ASM["AssemblyEntity<br/>uuid"]
  NODE["NodeEntity"]
  EDGE["EdgeEntity<br/>composite PK: source → target"]
  PART["PartEntity"]
  CONN["ConnectorEntity<br/>Position + Rotation"]
  ASSET["AssetEntity<br/>scale · Position · Rotation"]
  GRIDFS[("MongoDB GridFS<br/>asset_store_id (ObjectId)")]
  SETTING["SettingEntity<br/>key / value"]

  USER -->|"1..* user_id"| ASM
  ASM -->|"1..* assembly_id"| NODE
  NODE -->|"source"| EDGE
  NODE -->|"target"| EDGE
  NODE -->|"1..* node_id"| PART
  PART -->|"1..* part_id"| CONN
  EDGE -->|"edge_source/target"| CONN
  ASSET -->|"1..* asset_id"| PART
  ASSET -->|"asset_store_id"| GRIDFS
  USER -.->|"1..* user_id"| ASSET
  SETTING -.->|"mongo_db · mongo_collection"| GRIDFS
</div>
</div>

The full entity-relationship diagram of the relational schema is below.

<figure class="cover-figure">
  <img src="{{ '/assets/projects/product-configurator/database.png' | relative_url }}" alt="Entity-relationship diagram of the Product Configurator MySQL schema — users, assemblies, nodes, parts, edges, connectors, assets and settings" loading="lazy" />
  <figcaption>The MySQL schema — users own assemblies; an assembly's nodes are linked by directed edges; parts live on nodes and reference 3D assets; connectors join parts across edges.</figcaption>
</figure>

## Architecture by component

### 1. Backend (Java EE 6)

A classic Java EE stack — **EJB 3.1 + CDI + JPA 2.0 (Hibernate 3.6) + JAX-RS
1.1 (RESTEasy) + Hibernate Search** — packaged as a single EAR and deployed to
**JBoss AS 7.1.1**. The Maven reactor builds four modules:

- **`configurator-ejb`** (packaging `ejb`) — all domain logic: JPA entities,
  `@Stateless` DAOs, JAX-RS resources, the graph/product engine and the
  MongoDB integration.
- **`configurator-web-editor`** (packaging `war`, context root `/editor`) —
  the Backbone authoring SPA, built with Bower + RequireJS (`r.js` optimizer).
- **`configurator-web-embed`** (packaging `war`, context root `/embed`) — the
  embeddable viewer widget, built with npm + Gulp (Browserify).
- **`configurator-ear`** (packaging `ear`, `Configurator.ear`) — aggregates the
  EJB jar and both WARs via `application.xml`.

**Entity layer.** Nine JPA entities in `com.jiekebo.model.persistence` map the
configuration graph to MySQL: `UserEntity`, `AssemblyEntity`, `NodeEntity`,
`PartEntity`, `EdgeEntity` (composite key `source`/`target` via `@IdClass`),
`ConnectorEntity`, `AssetEntity`, `SettingEntity` and `HibernateSequenceEntity`.
Two `@Embeddable` value objects — `Position` (x/y/z) and `Rotation` (i/j/k
Euler angles with full rotation-matrix math) — carry the 3D transforms. An
`AssetStoreEntity` POJO (not `@Entity`) represents a parsed Three.js mesh and
lives in MongoDB, not MySQL.

**DAO layer.** `@Stateless` session beans (`AssemblyDao`, `NodeDao`,
`PartDao`, `EdgeDao`, `ConnectorDao`, `AssetDao`, `UserDao`, `SettingDao`,
`AssetStoreDao`) wrap a `@PersistenceContext` EntityManager. `AssetStoreDao`
is the exception — it talks to MongoDB GridFS for 3D model files and the
`texture` collection for base64-encoded texture images.

**Business layer.** `GraphLoader` (`@Startup @Singleton`) builds an in-memory
graph cache (`ConcurrentHashMap`) of all assemblies at startup; `ProductBuilder`
is the configurator engine — it loads the graph, applies the user's part
choices, walks adjacent nodes and runs `PartAligner` to compute 3D positions,
returning a `ProductResponse` with the added parts and the next available
choices. `UserController` handles PBKDF2 password hashing and authentication.

**REST layer.** `JaxRsActivator` (`@ApplicationPath("rest")`) activates JAX-RS
under `/rest/`. Resources expose full CRUD for assemblies, nodes, parts, edges,
connectors and assets, plus `GraphService` (graph cache rebuild) and
`UserService` (auth). The flagship endpoint is
`POST /rest/configurator/product` (`@PermitAll`) — it takes a `ProductRequest`
and returns the assembled product. `SecurityInterceptor` (a RESTEasy
`PreProcessInterceptor`) enforces HTTP Basic auth with `@RolesAllowed`/`@PermitAll`
and pushes the authenticated `UserEntity` into the request context;
`CorsInterceptor` adds the CORS headers the embed widget needs to call the API
cross-origin.

### 2. Frontend — editor SPA (Backbone + Three.js + d3)

A Backbone.js single-page app built with RequireJS/Bower, deployed as a WAR at
`/editor`. Templating is Handlebars, utility is lodash, DOM is jQuery — the
classic Backbone stack. Routing (`Backbone.Router`) drives four pages:

- **Configurator** (`#configurator/:id`) — the end-user product configuration
  page. A `Product` collection POSTs choices to `/rest/configurator/product`,
  the 3D `EditorView` re-renders the assembled product, and an
  `AvailableChoices` panel shows the dropdowns for the next configurable parts.
  An `EmbedStringView` generates a copy-to-clipboard embed snippet
  (`<script>` tag + `data-uuid` div) via ZeroClipboard.
- **Graph Editor** (`#graphEditor/:id`) — a node/edge graph editor for an
  assembly, drawn with **d3 v3** force layout (`d3.layout.force`, SVG). Add,
  rename and delete nodes; drag to create directed edges.
- **Part Editor** (`#partEditor/:id`) — a Three.js editor for parts, assets and
  connectors within a node. Camera/select/translate/rotate/scale modes via
  `TransformControls`; connectors are pickable magenta cubes selected by
  `THREE.Raycaster`.
- **Dashboard** (`#home`) — the landing page.

Shared 3D components: `EditorView` (a `THREE.WebGLRenderer` viewport with a
`PerspectiveCamera`, `OrbitControls`, `TransformControls` and a per-frame
render loop over three scenes — `modelScene`, `connectorScene`, `editorScene`)
and `SceneManager` (loads Three.js JSON geometries from the backend asset
store via `THREE.JSONLoader`, places/rotates/scales meshes, manages connector
tokens).

State is classic Backbone — models and collections are the stores, a shared
`Backbone.Events` bus decouples views, and **act.js** (a custom command/undo
library) wraps every mutation as an `Act.Action` with `persist`/`destroy`
callbacks that drive `model.save()`/`model.destroy()` against the REST API.

### 3. Frontend — embeddable widget (Backbone + Three.js)

A Browserify/Gulp-built read-only subset of the editor, deployed as a WAR at
`/embed` and consumable by any third-party page with a single script tag and a
container:

```html
<div id="configurator" data-uuid="e2aa656e-cc5b-4871-b752-4e33eaf2ffc9"></div>
<script src="/embed/configurator-0.0.1-SNAPSHOT.js"></script>
```

The widget reads the `data-uuid`, loads the assembly, drives the
`POST /rest/configurator/product` flow and renders the assembled product in
Three.js — the same `Configurator` + `Controls` + `AvailableChoices` +
`EditorView` + `SceneManager` views as the editor, minus the authoring tools.
Example pages (`table.html`, `ikea.html`, `lego.html`, `aiaiai.html`) under
`src/htdocs/example/` show it skinned for different products.

### 4. Databases

Two stores, split by data shape:

- **MySQL 5.6 (InnoDB)** — the configuration graph. Datasource
  `java:jboss/datasources/configurator`, Hibernate dialect
  `MySQL5InnoDBDialect`, `hbm2ddl.auto=update` keeps the schema in sync with
  the entities. Nine tables: `user`, `assembly`, `node`, `part`, `edge`,
  `connector`, `asset`, `setting`, `hibernate_sequence`. Schema and seed data
  live in `docker/mysql/dump.sql`.
- **MongoDB (GridFS)** — the 3D model assets. `AssetEntity.asset_store_id` (an
  ObjectId) links the MySQL `asset` row to a GridFS file containing a Three.js
  JSON geometry; a separate `texture` collection stores base64-encoded texture
  images keyed by `asset_id`. `MongoClientProvider` (`@Singleton`) manages the
  connection; the db and collection names are overridable via the `setting`
  table.

Both databases are bootstrapped with Docker (`docker/1-mysql-run.sh` …
`4-mongodb-dbinit.sh`), and `docker/start-databases.sh` runs them together.

## The UI

The screenshot below shows the configurator in action — the Three.js WebGL
viewer rendering an assembled product with the available-choices panel driving
the configuration.

<figure class="cover-figure">
  <img src="{{ '/assets/projects/product-configurator/ui.png' | relative_url }}" alt="Product Configurator UI — a Three.js WebGL viewer rendering an assembled product alongside an available-choices configuration panel" loading="lazy" />
  <figcaption>The configurator UI — a 3D WebGL view of the assembled product with the available-choices panel driving the configuration.</figcaption>
</figure>

## Technologies used

<div class="tech-grid">
  <div class="tech">
    <h4>Backend</h4>
    <ul>
      <li>Java EE 6 · EJB 3.1 · CDI</li>
      <li>JPA 2.0 / Hibernate 3.6</li>
      <li>JAX-RS 1.1 / RESTEasy</li>
      <li>Hibernate Search (Lucene/Solr)</li>
      <li>JBoss AS 7.1.1</li>
      <li>Maven multi-module (EAR)</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Frontend — editor</h4>
    <ul>
      <li>Backbone.js + lodash + jQuery</li>
      <li>Handlebars templates</li>
      <li>RequireJS / r.js (Bower)</li>
      <li>Three.js r93 (WebGL)</li>
      <li>d3 v3 force layout (graph editor)</li>
      <li>act.js (custom undo/redo)</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Frontend — embed</h4>
    <ul>
      <li>Backbone.js + Three.js (read-only)</li>
      <li>Browserify / Gulp build</li>
      <li>Drop-in &lt;script&gt; widget</li>
      <li>data-uuid assembly loading</li>
      <li>Cross-origin REST (CORS)</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Data &amp; infra</h4>
    <ul>
      <li>MySQL 5.6 (InnoDB)</li>
      <li>MongoDB + GridFS (3D assets)</li>
      <li>Docker (MySQL + MongoDB)</li>
      <li>HTTP Basic / PBKDF2 auth</li>
      <li>JAX-RS interceptors (security + CORS)</li>
      <li>SonarQube (code quality)</li>
    </ul>
  </div>
</div>

<div class="callout">
  <p>
    <strong>Status:</strong> a complete, deployed Java EE product configurator
    — a graph-driven engine that resolves configurable products in 3D, a
    Backbone + Three.js authoring SPA, an embeddable consumer widget, and the
    MySQL + MongoDB data layer backing it all. The architecture above describes
    the EAR as it ships to JBoss.
  </p>
</div>
