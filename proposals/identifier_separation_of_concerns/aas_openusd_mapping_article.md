# Industrial Digital Twin Semantics in OpenUSD: Bringing AAS Submodel Templates into the Scene Graph

*OpenUSD is rapidly becoming the common language for industrial digital twins. The AAS Submodel Template library offers something valuable: well-defined, industry-vetted semantic schemas for real-world assets. This article shows how to bring those two things together — and what it takes to do it cleanly.*

---

## OpenUSD as a Digital Twin Substrate

OpenUSD is not a 3D format. It is a composable semantic scene graph for representing, simulating, and linking complex systems across domains. If that framing is new to you, we covered it in depth in our previous article — *OpenUSD Is Not a 3D Format* — and it is the foundation everything here builds on.

The short version: an OpenUSD stage can carry geometry, physical properties, sensor data, compliance metadata, and lifecycle records in a single composable, layered format. Applied API schemas attach typed, discoverable properties to any prim. Relationships create explicit graph edges between prims. The composition engine — with its LIVERPS arc ordering — assembles data from multiple sources non-destructively.

That infrastructure is mature and widely adopted. The question for industrial digital twins is not whether to use OpenUSD. The question is what semantic content to put in it.

---

## The Asset Administration Shell: Valuable Semantics, Difficult Architecture

The Asset Administration Shell (AAS) was developed within Germany's Industrie 4.0 initiative as the standardized digital twin framework for manufacturing. Backed by substantial public funding and institutional support from organizations such as Plattform Industrie 4.0 and the Industrial Digital Twin Association (IDTA), it defined a rigorous metamodel for describing industrial assets: their identity, their technical properties, their lifecycle history, their documentation.

In practice, AAS adoption has been limited. Even in Germany, where investment was heaviest, industrial deployments have remained largely confined to research and pilot contexts. Outside Germany, uptake has been close to negligible. The reasons are well understood in the industry: the AAS architectural stack is complex, the tooling ecosystem is thin, and the technology choices have a distinctly academic feel compared to the pragmatic, ecosystem-driven approaches that have succeeded at scale in software infrastructure.

The most useful part of AAS is its Submodel Templates (SMTs).

SMTs are standardized, versioned, semantically typed schemas for recurring industrial concerns: digital nameplates, technical data sheets, handover documentation, carbon footprints, predictive maintenance records, bill of materials. Each is developed by domain working groups and published in the IDTA content hub. In my experience, they vary in maturity and depth — but the strongest ones represent genuine, hard-won agreement across manufacturers, regulators, and system integrators about what information a particular type of object should carry and what it means.

That semantic precision is exactly what a general-purpose scene graph format like OpenUSD needs when it moves into regulated industrial contexts. OpenUSD does not know what a Digital Product Passport requires. AAS Submodel Templates do.

The opportunity is to use the SMTs as semantic source material for OpenUSD schemas — taking the well-defined vocabulary and regulatory grounding of AAS data definitions and expressing them as typed, discoverable, composable OpenUSD applied schemas.

---

## The Identifier Problem That Has to Be Solved First

Before any AAS semantic content can be cleanly encoded in OpenUSD, there is a foundational conflict to resolve.

OpenUSD addresses every object in a scene by a **namespace path** — a hierarchical string like `/Plant/Line3/Pump_CM5_3`. These paths are the keys of OpenUSD's composition engine. They follow strict grammar rules. They must not contain characters that external systems routinely use in identifiers: slashes in the middle of a name, leading digits, hyphens used as separators, URI schemes.

Industrial assets carry identifiers from external systems that do not respect OpenUSD's grammar: product URIs from regulatory registries, serial numbers from MES systems, order codes from ERP systems, document IDs from DMS platforms, facility identifiers under EU regulations. A battery's product URI under EU Battery Regulation might be `https://www.example.com/batteries/CM5-3/2024`. An ECLASS-coded semantic identifier looks like `0173-1#02-AAO677#002`. Neither can become an OpenUSD prim name without transformation — and transformation means data loss and broken cross-system traceability.

A recent OpenUSD Enhancement Proposal, **Separation of Concerns for Identifiers in OpenUSD**, led by Aaron Luk and currently a draft submitted to the OpenUSD Alliance, addresses this directly. The principle is simple: OpenUSD namespace paths are for scene composition and hierarchy navigation. External identifiers should be stored *alongside* them, verbatim, in a standardized mechanism that is independent of OpenUSD grammar and discoverable by tooling.

Two candidate mechanisms are under evaluation — extending the existing `assetInfo` metadata dictionary with structured sub-dictionaries, or defining typed applied schemas with named properties. The mappings in this work use a `sourceId:*` attribute namespace (e.g., `sourceId:dpp:uriOfTheProduct`, `sourceId:doc:documentId`) as the practical expression of this principle: external identifiers are stored as plain strings on the prim, alongside OpenUSD-native typed attributes that encode the same data in OpenUSD-idiomatic form where applicable.

Without this mechanism, any AAS-to-OpenUSD mapping that involves external identifiers — which is essentially all of them — must choose between data loss and ad-hoc workarounds. The identifier separation is the enabling layer that makes clean mappings possible.

---

## The Mapping Approach

Our mappings follow the **OpenUSD Conceptual Data Mapping Template**, a community document that provides a consistent structure for cross-format mappings. Each submodel mapping document specifies:

- **Directionality**: one-way import or full bidirectional round-trip
- **High-level concept table**: AAS elements mapped to OpenUSD constructs, with round-trip fidelity rated per field (lossless, lossy, partial, or dropped)
- **Per-concept drilldowns**: encoding rules, fallback strategies, and data transformations
- **Illustrative OpenUSD schema definitions**: typed API schema classes formalizing the mapping
- **Round-trip analysis**: explicit documentation of what survives and what is lost in an AAS → OpenUSD → AAS cycle

The general encoding pattern is consistent across all three submodel templates mapped so far:

1. **The prim** represents the physical or logical asset. Its OpenUSD namespace path is chosen for composition purposes, independent of any AAS identifier.
2. **Applied API schemas** carry the submodel data as typed, discoverable properties. Tooling can find all prims carrying battery nameplate data by checking for `BatteryNameplateAPI` — no metadata parsing required.
3. **Source identifier attributes** store external identifiers verbatim on the prim. They are distinct from the OpenUSD-native attributes that encode the same data in OpenUSD-idiomatic form.
4. **Child scope prims** handle structured sub-elements with variable cardinality — markings, documents, document versions — using a consistent `Concept_NN` zero-padded naming convention that preserves AAS list ordering.
5. **`customData`** preserves AAS-specific metadata for round-trip fidelity: semantic IDs, language variants, and original AAS idShort values.

The OpenUSD schema hierarchy mirrors the AAS inheritance hierarchy. `DppNameplateAPI` is the shared base schema covering the full IDTA 02006-3-0 Digital Nameplate fields. Domain-specific schemas compose on top: `BatteryNameplateAPI` adds the five battery-specific fields from IDTA 02035-1; `IndustrialEquipmentAPI` adds the industrial equipment extensions. A battery prim carries both `DppNameplateAPI` and `BatteryNameplateAPI` — parallel to how the AAS Battery Nameplate SMT extends the Digital Nameplate SMT.

---

## The Three Submodel Templates Mapped

### IDTA 02006-3-0 — Digital Nameplate for Industrial Equipment

The Digital Nameplate is the foundational AAS SMT for industrial asset identity. It defines the core fields present in virtually every product passport: product URI, manufacturer name, product designation, order codes, article numbers, serial number, date of manufacture, facility identifier, markings, and asset-specific properties. Domain-specific nameplates (battery, machinery, software) derive from it.

A critical distinction the mapping makes explicit is between **type-level** and **instance-level** identifiers:

- **Type-level identifiers** — `ManufacturerProductType`, `OrderCodeOfManufacturer`, `ProductArticleNumberOfManufacturer` — identify a *product model*: a class of equipment manufactured to a specification, shared across all units.
- **Instance-level identifiers** — `SerialNumber`, `URIOfTheProduct`, `UniqueFacilityIdentifier` — identify a *specific serialized unit*: a particular physical object at a particular location.

This distinction is fundamental in AAS modeling and must survive in the OpenUSD encoding. Downstream systems — maintenance platforms, compliance registries, recycling operators — need to know whether they are looking at a model reference or a unit reference. Both classes of identifiers use the `sourceId:dpp:*` dual-encoding pattern, but the semantic distinction is preserved in the schema documentation and is available to tooling.

The `idta_02006_digital_nameplate_sample.usda` file shows a complete encoding for an industrial pump (`Pump_MusterAG_CM5_3`) including contact information, markings, ATEX explosion safety data, and asset-specific technical properties.

### IDTA 02035-1 — Digital Battery Passport (Part 1: Digital Nameplate)

The battery passport nameplate is the Digital Nameplate extended for compliance with EU Battery Regulation (EU) 2023/1542. It adds five fields to the 02006-3-0 base: `LifeCycleStage`, `OperatorIdentifier`, `ManufacturerIdentifier`, `EUDeclarationOfConformity`, and `ResultsOfTestReportsProvingCompliance`.

In OpenUSD these are encoded in `BatteryNameplateAPI`, applied alongside `DppNameplateAPI`. `ManufacturerIdentifier` is an opaque external string stored in `sourceId:dpp:manufacturerIdentifier`. `EUDeclarationOfConformity` and `ResultsOfTestReportsProvingCompliance` are arrays of document identifier strings that cross-reference compliance documents in a Handover Documentation submodel — they are stored as `sourceId:dpp:*` string arrays, and the Handover Documentation mapping defines how to resolve the cross-reference on a shared OpenUSD stage. `LifeCycleStage` uses an OpenUSD `token` that maps to an ECLASS IRDI via a lookup table, preserving both OpenUSD-native enumeration semantics and AAS semantic identifier round-trip fidelity.

An important point about scope: the battery passport nameplate covers *identity and markings* only. The full DPP as defined by the EU Battery Regulation also requires technical data, carbon footprint, state of health, and other submodels. The nameplate mapping is the foundation layer; further submodel mappings build on it using the same schema composition approach.

The `idta_02035_battery_passport_sample.usda` file shows a complete battery instance (`Battery_A12345X75EN`).

### IDTA 02004-2-0 — Handover Documentation

Handover Documentation is the AAS SMT for transferring product-related documents across organizational boundaries over an asset's lifecycle: operating manuals, certificates, drawings, compliance declarations. Its central identifier challenge is the `DocumentIds` SML — an explicit multi-system identifier store where a single document may simultaneously carry IDs from a DMS, an ERP system, a regulatory body, and a certification authority. All of these must survive cross-system transit verbatim.

The mapping defines five coordinated API schemas: `HandoverDocumentationAPI` as the discoverer, `DocumentAPI` per document entry with OpenUSD relationships for intra-stage coverage, `DocumentVersionAPI` for file references and version metadata, `DocumentIdAPI` for individual document identifiers with `sourceId:doc:documentId` as the verbatim string, and `DocumentClassificationAPI` for VDI 2770 / IEC 61355 classifications.

The cross-reference with the battery passport is made explicit: when EU Declaration of Conformity IDs stored in `sourceId:dpp:euDeclarationOfConformityIds` on a battery prim match `docId:documentId` values in `DocumentId_NN` entries on the same stage, the link can be expressed as an OpenUSD relationship — connecting the regulatory compliance requirement on the product prim to the actual document record in the documentation layer.

The `idta_02004_handover_documentation_sample.usda` demonstrates this with two documents: an EU Declaration of Conformity cross-referenced from the battery passport, and an operating manual with German and English versions linked as translations.

---

## What This Enables and What Comes Next

These three mappings demonstrate the methodology. The AAS SMT library contains many more: Technical Data (IDTA 02003), Carbon Footprint (IDTA 02023), Software Nameplate (IDTA 02007), Predictive Maintenance (IDTA 02017), and others. Each can follow the same pattern: map the SMT fields to typed OpenUSD applied schema properties, encode external identifiers verbatim via `sourceId:*` attributes, preserve semantic IDs and language metadata in `customData`, and define child prim structures for variable-cardinality sub-elements.

The result is an OpenUSD stage that can carry the full semantic richness of an AAS-defined asset description — with the geometry, physics, and simulation data that AAS has no vocabulary for — in a single composable, layered format that OpenUSD-native tooling can traverse, validate, and render without an AAS runtime.

OpenUSD does not need to become AAS. But it can carry what AAS Submodel Templates know how to define.

---

## Try the Files

The mapping documents and USDA samples are available in the `identifier_separation_of_concerns` directory of the OpenUSD proposals repository (PR #105 against `PixarAnimationStudios/OpenUSD-proposals`):

| File | What It Is |
|---|---|
| `README.md` | Identifier Separation of Concerns proposal |
| `conceptual_data_mapping_template.md` | Reusable template for new submodel mappings |
| `idta_02006_digital_nameplate_openusd_mapping.md` | IDTA 02006-3-0 mapping |
| `idta_02006_digital_nameplate_sample.usda` | Industrial pump nameplate example |
| `idta_02035_battery_passport_openusd_mapping.md` | IDTA 02035-1 battery passport mapping |
| `idta_02035_battery_passport_sample.usda` | Battery unit nameplate example |
| `idta_02004_handover_documentation_openusd_mapping.md` | IDTA 02004-2-0 mapping |
| `idta_02004_handover_documentation_sample.usda` | Documentation submodel with cross-references |

The USDA files are valid USD ASCII and can be opened in any OpenUSD-capable tool. Schema classes are defined inline for illustration; in production they would be registered as proper OpenUSD schema plugins.

---

## In Summary

AAS Submodel Templates contain something genuinely useful: semantically precise, industry-vetted definitions for real-world asset data, grounded in regulatory requirements and developed by domain experts. OpenUSD contains something AAS lacks: a proven, high-performance, widely-adopted composition and interchange infrastructure backed by the most significant technology investment in the 3D and simulation space.

The mappings presented here show how to take the valuable part of AAS — its submodel semantics — and express it cleanly in the format that is actually winning. The identifier separation proposal is the enabling mechanism that makes the encodings clean rather than lossy. The bidirectional mapping structure ensures that data can move in both directions without silent loss.

Industrial digital twins need good semantics. They also need infrastructure that the industry will actually use. These two things do not have to be in tension.

---

*The mapping documents in this article are informed by the OpenUSD Enhancement Proposal "Separation of Concerns for Identifiers in USD," led by Aaron Luk and currently a draft under review by the Alliance for OpenUSD. The IDTA Submodel Template library is published at https://industrialdigitaltwin.org/en/content-hub/submodels.*

*This article was written with the assistance of [Claude Code](https://claude.ai/code) (Claude Sonnet 4.6) by Anthropic.*
