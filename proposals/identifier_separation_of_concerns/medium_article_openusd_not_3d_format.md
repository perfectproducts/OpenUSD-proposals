# OpenUSD Is Not a 3D Format

**It is a composable semantic scene graph for representing, simulating, and linking complex systems across domains. Born at Pixar, it is widely mistaken for a rendering format. Let's correct that — and show why it matters for Digital Twins, DPP, and Physical AI.**

---

It is easy to understand how the misconception formed.

Universal Scene Description was created at Pixar. Its first public showcase was a film pipeline tool. Its file extensions are `.usd`, `.usda`, `.usdc`, `.usdz`. Its ecosystem features DCC tools — Maya, Houdini, Blender — and rendering workflows at Apple, NVIDIA, and Adobe. If you encountered OpenUSD for the first time through any of those entry points, "3D format" is a reasonable first impression.

But OpenUSD has moved far beyond that origin. Today it is actively adopted in industry, construction, and robotics. NVIDIA's Omniverse platform has been the primary force pushing OpenUSD into industrial contexts — factory simulation, robotic teleoperation, digital factory twins — and is now central to Physical AI workflows where simulation and the real world must stay in sync. Global industrial software leaders like Siemens and PTC are adopting OpenUSD as part of their digital twin and PLM strategies. The format that assembled CGI scenes at Pixar is now assembling digital representations of production lines, buildings, and autonomous systems.

So why does the "3D format" label persist? Because the mental model never caught up with the technology.

---

## What OpenUSD Actually Is

OpenUSD is best understood as a **composable, layered scene graph and data model for representing complex systems** — with geometry, time, relationships, and semantics.

Not just 3D.

It sits closer to:

- A graph-based digital twin substrate
- A scene + data + relationship model
- A composition system for heterogeneous industrial data

The formal definition from the Alliance for OpenUSD captures it well: a "rich and extensible ecosystem of software and standards for collaboratively constructing animated 3D scenes." But notice what that definition does not say: it does not say *only* 3D scenes. The word "extensible" is doing a lot of work. The schemas that make OpenUSD meaningful for geometry — `UsdGeomMesh`, `UsdGeomCamera` — are just schemas. You can define any schema. You can represent anything.

---

## The Correct Mental Model

Instead of:

> ❌ OpenUSD = 3D rendering format

The accurate model is:

> **OpenUSD = Scene Graph + Composition Engine + Typed Data Model + Relationship Layer**

Each component matters independently:

**Scene graph** — a hierarchy of objects. Not meshes. Objects. A battery cell, a compliance record, a robot joint, and a triangle mesh are all equally valid prim candidates.

**Composition engine** — OpenUSD defines a precise set of composition arcs, remembered by the acronym **LIVERPS**: **L**ocal, **I**nherits, **V**ariantSets, r**E**locates, **R**eferences, **P**ayloads, **S**pecializes. These arcs resolve in a defined strength order, enabling non-destructive assembly of data from multiple sources and teams — overrides, configurations, deferred loading, and structural inheritance all composing cleanly without conflict.

**Typed data model** — schemas and attributes with declared types. `battery:capacity = 75.0 (kWh)` is as valid as `xformOp:translate = (1, 0, 0)`.

**Relationship layer** — named, typed graph edges between prims. A sensor stream relationship, a supplier relationship, a simulation input binding — these are first-class OpenUSD constructs, not hacks.

Take away the geometry schemas and OpenUSD is still a complete, functional system. That is not an accident.

---

## Why It Is Fundamentally a Graph

OpenUSD describes a directed scene graph:

- **Nodes** → primitives (Prims)
- **Edges** → relationships (references, inherits, payloads, explicit `rel` declarations)
- **Attributes** → typed metadata on nodes

Consider this:

```
Car
 ├── Body
 │    ├── Material: Aluminum
 │    └── Mass: 120 kg
 ├── Battery
 │    ├── Capacity: 75 kWh
 │    └── Supplier: DID:example:supplier:123
 └── Wheels
      ├── Geometry
      └── SensorData
```

Nothing in this structure requires rendering. It is a **semantic graph of a system** — its composition, its properties, its provenance. The `Geometry` on `Wheels` is one attribute set among several. The `SensorData` on `Wheels` is equally valid OpenUSD.

This is where the mental model shift matters. A 3D format is for describing surfaces. OpenUSD is for describing systems.

---

## Where Semantics Live in OpenUSD

OpenUSD has four distinct semantic mechanisms, and only one of them involves geometry.

**Prim structure (hierarchy)** defines system decomposition:
`product → subsystem → component → part`

**Custom attributes (typed metadata)** carry domain data:
```usda
battery:capacity = 75.0
battery:recycledContent = 0.42
battery:cellChemistry = "NMC"
```

**Relationships** create graph edges beyond parent-child hierarchy:
```usda
rel materialBinding
rel supplier
rel simulationInput
rel sensorStream
```

**Applied schemas** are where domain meaning gets encoded formally. Unlike typed schemas that define what a prim *is*, API schemas define what a prim *carries* — typed properties and behaviors declared once in a schema definition file, then applied to any prim via `apiSchemas` metadata. Domain concepts like these become first-class schema types:

```usda
class "BatteryNameplateAPI" (inherits = </APISchemaBase>) { ... }
class "DrivetrainAPI" (inherits = </APISchemaBase>) { ... }
class "SensorBindingAPI" (inherits = </APISchemaBase>) { ... }
```

Once registered via the `usdGenSchema` toolchain, these schemas generate typed accessors and make domain properties formally declared and discoverable — laying the groundwork for interoperability, with the degree of validation depending on the tooling built around them.

---

## How This Maps to Digital Product Passports

The EU Battery Regulation (2023/1542) mandates a Digital Product Passport for batteries entering the European market: lifecycle data, material composition, recycled content, carbon footprint, compliance certifications.

This is not a rendering problem.
It is a data, structure, and relationship problem.

OpenUSD can represent the structural and relational model of a battery product precisely because it is a graph — not a 3D format. A battery is not a mesh; it is a system. Its hierarchy of subsystems, its typed regulatory attributes, and its relationships to suppliers, certifications, and lifecycle records map directly to native OpenUSD constructs.

What emerges is a clean architectural separation:

| Layer | Role |
|-------|------|
| OpenUSD graph model | System structure, typed properties, relationships |
| Semantic vocabularies (ECLASS, IEC CDD) | Shared meaning for attributes |
| Verifiable Credentials | Trust and provenance |
| Data space connectivity (Catena-X, etc.) | Exchange and sovereignty |

Each layer solves a different problem.

OpenUSD does not define meaning — it structures it.
It does not establish trust — it carries references to it.
It does not handle data exchange — it provides the model being exchanged.

That distinction matters.

Treating OpenUSD as a rendering layer beneath a "real" data architecture leads to unnecessary complexity: duplicated models, fragmented semantics, and brittle integrations. The correct approach is to use OpenUSD as the structural substrate — the system model that everything else attaches to.

Digital Product Passports are not documents. They are graphs of products, properties, and relationships over time.

That is exactly what OpenUSD was built to represent.

In a companion article, we map the EU Digital Battery Passport to OpenUSD applied schemas in full — showing field by field how regulatory data lands cleanly in the scene graph, and what the bidirectional round-trip looks like in practice.

---

## Why OpenUSD Is Especially Powerful for Physical AI

Physical AI — robots, autonomous systems, digital factories — requires a representation layer that can hold:

- **Multi-domain models**: mechanical, electrical, software, simulation, sensor data, simultaneously
- **Time and variants**: configurations, states, versions — a robot arm in maintenance mode is a different variant of the same prim hierarchy
- **Simulation integration**: OpenUSD's physics schemas (from NVIDIA's PhysicsSchemas) connect directly to real-time simulation

This is why OpenUSD is emerging as the structural backbone of Physical AI systems. It is not that game engines and robotics frameworks happen to use 3D formats for data exchange. It is that OpenUSD's composition and schema system provides the right level of abstraction for representing complex, multi-domain physical systems — and geometry is simply one of those domains.

NVIDIA Omniverse is explicit about this: the platform is not a 3D viewer, it is a simulation and collaboration substrate. OpenUSD is its data model. Apple's visionOS uses OpenUSD as its spatial computing format — not because Apple needed a 3D format, but because OpenUSD's composition model handles the complexity of layered, multi-source spatial scenes better than anything else available.

---

## The One-Sentence Correction

> **OpenUSD is not a 3D format — it is a composable semantic scene graph for representing, simulating, and linking complex systems across domains.**

Its origins at Pixar gave it excellent geometry support. Its architecture gave it something more general: a principled, extensible, composable model for describing any structured system.

The geometry schemas are an application of OpenUSD, not its definition.

Treating OpenUSD as a 3D format in industrial and Physical AI contexts is a missed opportunity. Existing industrial semantics — AAS Submodel Templates, regulatory data models, PLM schemas — can be mapped directly to OpenUSD applied schemas, making them composable, layerable, and simulation-ready without rebuilding them from scratch. The composition engine does the heavy lifting; you just need to bring the right schemas to it.

Define the schemas. Use the relationships. Trust the composition engine. That is what OpenUSD is for.

---

*Michael Wagner is CEO of [SyncTwin GmbH](https://synctwin.ai), a pioneer in OpenUSD-native middleware for industrial digital twins, and an NVIDIA Omniverse Ambassador. He is a contributor to the OpenUSD Enhancement Proposal "Separation of Concerns for Identifiers in USD," currently a draft under review by the Alliance for OpenUSD.*
