---
orphan: true
---
# Conceptual Data Mapping: IDTA 02035-1 Digital Battery Passport (Part 1: Digital Nameplate) to OpenUSD

```{important}
**{octicon}`tag;1em` Document Version:** {bdg-secondary}`0.1.0`
<br>**{octicon}`calendar;1em` Last Update:** {bdg-secondary}`2026-04-02`
```

## Introduction

### Overview

This document maps the **IDTA 02035-1 Digital Battery Passport – Part 1: Digital Nameplate** (DBP Nameplate) to OpenUSD. The DBP Nameplate is a Submodel Template of the Asset Administration Shell (AAS), defining the static identification and marking attributes required by EU Battery Regulation (EU) 2023/1542 and DIN DKE SPEC 99100.

The mapping is one-directional: **DBP → OpenUSD**. The goal is to show how a battery asset described in an AAS can carry its regulatory nameplate data within or alongside a USD representation—enabling USD-native tooling (DCC, simulation, digital twin pipelines) to consume DBP-conformant information without requiring a separate AAS runtime.

This mapping is also intended as a concrete example supporting the **Identifier Separation of Concerns** proposal, since the DBP Nameplate contains multiple distinct identifier types (product URI, serial number, manufacturer ID, facility ID) that the proposal explicitly addresses.

### References

#### DBP Reference

| Version | Reference Documents |
|---|---|
| 1.0 (February 2026) | IDTA 02035-1: Digital Battery Passport – Part 1: Digital Nameplate |
| — | DIN DKE SPEC 99100: Requirements for data attributes of the battery passport (February 2025) |
| — | EU Battery Regulation (EU) 2023/1542, Article 77 and Annex XIII |
| — | IDTA-02006: Digital Nameplate for Industrial Equipment 3.0 (parent template) |

#### OpenUSD Reference

| Version | Reference Documents |
|---|---|
| 24.08 | [OpenUSD C++ and Schema Documentation](https://openusd.org/release/api/index.html), [OpenUSD Github Repository](https://github.com/PixarAnimationStudios/OpenUSD), [USD Terms and Concepts](https://openusd.org/release/glossary.html) |

### General Assumptions and Constraints

- **One-way mapping** (DBP → OpenUSD). Roundtrip fidelity to AAS is not a goal.
- The DBP Nameplate describes a **physical battery instance**. In USD, the corresponding entity is a prim (typically of kind `component`) representing that battery asset.
- AAS uses a typed metamodel (Submodel, SubmodelElementCollection, SubmodelElementList, Property, MultiLanguageProperty, File). OpenUSD has no direct equivalent of this hierarchy; the mapping uses **applied API schemas**, **prim attributes**, **metadata dictionaries**, and **child prims** as appropriate.
- Two encoding strategies are discussed for identifier fields (see [Appendix A](#appendix-a-identifier-encoding-strategies)), corresponding directly to the two candidate approaches in the Identifier Separation of Concerns proposal.
- USD has no native `date` type; ISO 8601 date strings (`YYYY-MM-DD`) stored as `string` attributes are used.
- AAS MultiLanguageProperty (MLP) stores one string per language tag. USD has no native multi-language string type. The mapping stores the primary string value as a `string` attribute; multi-language variants are out of scope unless a domain-specific convention is adopted.
- Nested AAS structures (SMC, SML) are mapped either to **namespaced attribute groups** (flat namespace) or **child prims** depending on cardinality and complexity.

### Definitions, Acronyms, Abbreviations

| Term or Abbreviation | Description |
|---|---|
| AAS | Asset Administration Shell – the IDTA/Industrie 4.0 digital twin framework |
| DBP | Digital Battery Passport (IDTA 02035-1 series) |
| SMT | Submodel Template – a standardized AAS submodel definition |
| SM | Submodel – top-level container of related data elements in AAS |
| SMC | SubmodelElementCollection – an unordered named group of SubmodelElements |
| SML | SubmodelElementList – an ordered list of SubmodelElements of the same type |
| Prop | Property – a typed scalar SubmodelElement |
| MLP | MultiLanguageProperty – a Property with per-language-code string values |
| IRDI | International Registration Data Identifier – semantic ID format (IEC CDD/ECLASS) |
| semanticId | AAS mechanism for linking a SubmodelElement to a concept definition |
| Applied API Schema | A USD schema that can be applied to any prim to add typed attributes |
| `assetInfo` | A USD metadata dictionary on a prim for asset-level identification metadata |
| `customData` | A USD metadata dictionary for arbitrary user/domain data on a prim |
| `token` | A USD value type for interned strings, used for enumerated values |
| `asset` | A USD value type (`SdfAssetPath`) for file or URI references |
| kind | A USD metadata string classifying a prim's role in the model hierarchy |
| scenegraph | A data structure organizing a scene's logical and spatial representation |

---

## Concepts

The DBP Nameplate Submodel is structured into three groups:

1. **Nameplate core properties** – identification, manufacturer, dates, lifecycle
2. **AddressInformation** (SMC drop-in) – physical address of the economic operator
3. **Markings** (SML) – regulatory and conformity symbols

In OpenUSD, these map to:

| DBP (AAS) | Schema | OpenUSD | Description |
|---|---|---|---|
| [BatteryNameplate (Submodel)](#batterynameplate-submodel) | both | `DppNameplateAPI` + `BatteryNameplateAPI` on a `component` prim | The submodel maps to two composed API schemas: a generic nameplate (IDTA-02006 origin) and a battery-specific extension. |
| [URIOfTheProduct](#urioftheproduct) | `DppNameplateAPI` | `assetInfo["identifier"]` or `dpp:uriOfTheProduct` (asset) | Unique battery passport URI. Primary candidate for `assetInfo["identifier"]`; see [Appendix A](#appendix-a-identifier-encoding-strategies). |
| [ManufacturerName](#manufacturername) | `DppNameplateAPI` | `dpp:manufacturerName` (string) | Manufacturer name. MLP; store primary string value. |
| [AddressInformation (SMC)](#addressinformation) | `DppNameplateAPI` | Namespaced attributes `dpp:address:*` or child prim | Nested address fields. |
| [SerialNumber](#serialnumber) | `DppNameplateAPI` | `dpp:serialNumber` (string) | Per-instance serial number. |
| [DateOfManufacture](#dateofmanufacture) | `DppNameplateAPI` | `dpp:dateOfManufacture` (string, ISO 8601) | Manufacturing date. |
| [DateOfPuttingIntoService](#dateofputtingintoservice) | `DppNameplateAPI` | `dpp:dateOfPuttingIntoService` (string, ISO 8601) | Optional service start date. |
| [UniqueFacilityIdentifier](#uniquefacilityidentifier) | `DppNameplateAPI` | `dpp:uniqueFacilityIdentifier` (string) | Manufacturing location identifier. |
| [Markings (SML)](#markings) | `DppNameplateAPI` | Child prims under `Markings/` scope | Regulatory markings and conformity symbols. |
| [LifeCycleStage](#lifecyclestage) | `BatteryNameplateAPI` | `dpp:lifeCycleStage` (token) | Battery-specific. Enumerated lifecycle state per EU Battery Regulation. |
| [OperatorIdentifier](#operatoridentifier) | `BatteryNameplateAPI` | `dpp:operatorIdentifier` (string) | Battery-specific. Optional operator ID. |
| [ManufacturerIdentifier](#manufactureridentifier) | `BatteryNameplateAPI` | `assetInfo["name"]` or `dpp:manufacturerIdentifier` (string) | Battery-specific. Formal manufacturer identifier. See [Appendix A](#appendix-a-identifier-encoding-strategies). |
| [EUDeclarationOfConformity (SML)](#eudeclarationofconformity) | `BatteryNameplateAPI` | `dpp:euDeclarationOfConformity` (asset[]) | Battery-specific. EU DoC document asset paths or URIs. |
| [ResultsOfTestReportsProvingCompliance (SML)](#resultsoftestreportsprovingcompliance) | `BatteryNameplateAPI` | `dpp:testReportCompliance` (asset[]) | Battery-specific. Compliance test report asset paths or URIs. |
| — | — | `assetInfo["version"]` | No DBP equivalent; USD-native asset versioning. |

---

### BatteryNameplate (Submodel)

The `BatteryNameplate` submodel maps to **two composed applied API schemas** in USD, mirroring the spec's own derivation structure:

- **`DppNameplateAPI`** — the generic digital product nameplate, encoding the fields inherited from IDTA-02006 (Digital Nameplate for Industrial Equipment 3.0). Applicable to any product with a digital product passport.
- **`BatteryNameplateAPI`** — the battery-specific extension, encoding the five fields added by IDTA-02035: `LifeCycleStage`, `OperatorIdentifier`, `ManufacturerIdentifier`, `EUDeclarationOfConformity`, and `ResultsOfTestReportsProvingCompliance`.

Both schemas use the `dpp:` namespace prefix. The prim `kind` should be `component`. The AAS `semanticId` of the submodel is preserved in `customData` for provenance.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| Submodel semanticId (IDTA-02006) | `customData["aas:submodelSemanticId:nameplate"]` (string) | Preserves the IDTA-02006 parent template semantic ID. |
| Submodel semanticId (IDTA-02035-1) | `customData["aas:submodelSemanticId:battery"]` (string) | Preserves `https://admin-shell.io/idta/digitalbatterypassport/nameplate/1/0/Nameplate`. |

#### Schema Definitions (illustrative)

```usda
# ── Generic DPP nameplate (IDTA-02006 origin) ─────────────────────────────────
class "DppNameplateAPI" (
    inherits = </APISchemaBase>
    doc = "Generic digital product nameplate. Encodes IDTA-02006 Digital Nameplate for Industrial Equipment fields. Applicable to any product with a digital product passport."
    customData = {
        token apiSchemaType = "singleApply"
    }
) {
    asset dpp:uriOfTheProduct (
        doc = "Unique URI identifying this product passport instance. (AAS: URIOfTheProduct)"
    )
    string dpp:manufacturerName (
        doc = "Legally valid manufacturer name. (AAS: ManufacturerName, MLP)"
    )
    string dpp:serialNumber (
        doc = "Per-instance serial number. (AAS: SerialNumber)"
    )
    string dpp:dateOfManufacture (
        doc = "Manufacturing date in ISO 8601 format YYYY-MM-DD. (AAS: DateOfManufacture)"
    )
    string dpp:dateOfPuttingIntoService (
        doc = "Service start date in ISO 8601 format YYYY-MM-DD. Optional. (AAS: DateOfPuttingIntoService)"
    )
    string dpp:uniqueFacilityIdentifier (
        doc = "Unique identifier of the manufacturing facility. (AAS: UniqueFacilityIdentifier)"
    )
}

# ── Battery-specific extension (IDTA-02035 additions) ─────────────────────────
class "BatteryNameplateAPI" (
    inherits = </APISchemaBase>
    doc = "Battery-specific nameplate extension. Encodes the five fields added by IDTA-02035-1 over the base IDTA-02006 template. Apply together with DppNameplateAPI."
    customData = {
        token apiSchemaType = "singleApply"
    }
) {
    token dpp:lifeCycleStage (
        allowedTokens = ["original", "repurposed", "re-used", "remanufactured", "waste"]
        doc = "Life cycle status of the battery per EU Battery Regulation. (AAS: LifeCycleStage)"
    )
    string dpp:operatorIdentifier (
        doc = "Unique operator identifier per ISO/IEC 15459. Optional. (AAS: OperatorIdentifier)"
    )
    string dpp:manufacturerIdentifier (
        doc = "Formal identifier of the manufacturer. (AAS: ManufacturerIdentifier)"
    )
    asset[] dpp:euDeclarationOfConformity (
        doc = "Asset paths or URIs to EU Declaration of Conformity documents. (AAS: EUDeclarationOfConformity)"
    )
    asset[] dpp:testReportCompliance (
        doc = "Asset paths or URIs to compliance test reports. (AAS: ResultsOfTestReportsProvingCompliance)"
    )
}
```

#### Usage Example (USDA)

```usda
def Xform "Battery_A12345X75EN" (
    kind = "component"
    prepend apiSchemas = ["DppNameplateAPI", "BatteryNameplateAPI"]
    customData = {
        string "aas:submodelSemanticId:battery" = "https://admin-shell.io/idta/digitalbatterypassport/nameplate/1/0/Nameplate"
    }
    assetInfo = {
        asset identifier = @https://dcqr.com/?m=R123456789@
        string name = "XYZ-123456"
    }
) {
    # DppNameplateAPI fields
    asset dpp:uriOfTheProduct = @https://dcqr.com/?m=R123456789@
    string dpp:manufacturerName = "Muster AG"
    string dpp:serialNumber = "A12345-X75EN"
    string dpp:dateOfManufacture = "2022-01-01"
    string dpp:dateOfPuttingIntoService = "2022-06-01"
    string dpp:uniqueFacilityIdentifier = "987654321"

    # BatteryNameplateAPI fields
    token dpp:lifeCycleStage = "original"
    string dpp:manufacturerIdentifier = "XYZ-123456"
    asset[] dpp:euDeclarationOfConformity = [@./docs/eu_doc_conformity_en.pdf@]
    asset[] dpp:testReportCompliance = [@./docs/test_report_compliance.pdf@]
}
```

---

### URIOfTheProduct

The battery passport URI is the globally unique identifier for this battery passport instance. It is the primary candidate for `assetInfo["identifier"]` in USD, following the pattern established for asset identification in the Identifier Separation of Concerns proposal.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| URIOfTheProduct | `assetInfo["identifier"]` (asset) | **Preferred location.** Universally unique per-instance URI. Maps naturally to USD's established `assetInfo` convention. |
| URIOfTheProduct | `dpp:uriOfTheProduct` (asset) | **Schema attribute.** Redundant with `assetInfo["identifier"]` but preserves semantic traceability to the DBP field name and AAS semanticId. |

##### Property: URIOfTheProduct

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `URIOfTheProduct` | `xs:anyURI` | Mandatory (cardinality 1). AAS semanticId: `0112/2///61987#ABN590#002` |
| OpenUSD | `assetInfo["identifier"]` | `asset` (SdfAssetPath) | `asset` type accommodates both file paths and URIs |
| OpenUSD (schema attr) | `dpp:uriOfTheProduct` | `asset` | Optional redundant encoding for schema-level discoverability |

#### Metadata

The AAS `semanticId` (`0112/2///61987#ABN590#002`) can be preserved in `customData`:

```usda
customData = {
    string "aas:semanticId:uriOfTheProduct" = "0112/2///61987#ABN590#002"
}
```

---

### ManufacturerName

The legally valid name of the manufacturer. In AAS this is a MultiLanguageProperty (MLP); USD has no native per-locale string type, so the primary string value is stored directly.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| ManufacturerName (MLP) | `dpp:manufacturerName` (string) | Store the primary language value. Language tag lost unless a convention like `dpp:manufacturerName_de` is adopted. |

##### Property: ManufacturerName

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `ManufacturerName` | MLP (MultiLanguageProperty) | Mandatory (cardinality 1). AAS semanticId: `0112/2///61987#ABA565#009` |
| OpenUSD | `dpp:manufacturerName` | `string` | Single language value |

#### Metadata

For multi-language use cases, additional `customData` entries may encode language variants:

```usda
customData = {
    dictionary "dpp:manufacturerName:i18n" = {
        string "de" = "Muster AG"
        string "en" = "Muster AG"
    }
}
```

---

### AddressInformation

The manufacturer's physical address is structured in AAS as a SubmodelElementCollection (SMC) drop-in from IDTA 02002-1 Contact Information. The mandatory fields per DIN DKE SPEC 99100 are: Street, Zipcode, CityTown, NationalCode; optionally Email and AddressOfAdditionalLink.

In USD, two encoding options exist:

**Option 1 – Namespaced attributes (flat):** Attributes with the `dpp:address:` prefix on the same prim. Simple to author but produces a flat list of attributes.

**Option 2 – Child prim:** A child prim (e.g., `Address`) under the battery prim. Cleaner namespace separation; allows the address to be referenced or overridden independently.

Child prim (Option 2) is recommended when address data is expected to be shared across multiple battery instances or updated independently.

#### Properties

| DBP (AddressInformation) | OpenUSD | Description |
|---|---|---|
| Street (MLP) | `dpp:address:street` (string) | Street address |
| Zipcode (MLP) | `dpp:address:zipcode` (string) | Postal code |
| CityTown (MLP) | `dpp:address:cityTown` (string) | City or town |
| NationalCode (MLP) | `dpp:address:nationalCode` (string) | ISO country code |
| Email (SMC, optional) | `dpp:address:email` (string) | Contact email |
| AddressOfAdditionalLink (optional) | `dpp:address:additionalLink` (asset) | Web address of the manufacturer |

#### Usage Example (Option 1 – namespaced attributes)

```usda
def Xform "Battery_A12345X75EN" (
    prepend apiSchemas = ["DppNameplateAPI", "BatteryNameplateAPI"]
) {
    string dpp:address:street = "Sample Street 1"
    string dpp:address:zipcode = "12345"
    string dpp:address:cityTown = "City"
    string dpp:address:nationalCode = "DE"
    string dpp:address:email = "contact@muster-ag.de"
    asset dpp:address:additionalLink = @https://www.muster-ag.de@
}
```

#### Usage Example (Option 2 – child prim)

```usda
def Xform "Battery_A12345X75EN" (
    prepend apiSchemas = ["DppNameplateAPI", "BatteryNameplateAPI"]
) {
    def Scope "Address" {
        string street = "Sample Street 1"
        string zipcode = "12345"
        string cityTown = "City"
        string nationalCode = "DE"
        string email = "contact@muster-ag.de"
        asset additionalLink = @https://www.muster-ag.de@
    }
}
```

---

### SerialNumber

The per-instance serial number uniquely identifies an individual battery unit. This is a strong candidate for `assetInfo` per the Identifier Separation of Concerns proposal, but its appropriate location depends on whether USD treats the prim as representing the battery class or instance (see [Appendix A](#appendix-a-identifier-encoding-strategies)).

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| SerialNumber | `dpp:serialNumber` (string) | Mandatory (cardinality 1). |

##### Property: SerialNumber

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `SerialNumber` | `xs:string` | Mandatory. AAS semanticId: `0112/2///61987#ABA951#009` |
| OpenUSD | `dpp:serialNumber` | `string` | Instance-level identifier |

---

### DateOfManufacture

Manufacturing date of the individual battery item. Per DBP, this must be per-instance (not per-model). ISO 8601 format (YYYY-MM-DD).

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| DateOfManufacture | `dpp:dateOfManufacture` (string) | Mandatory (cardinality 1). ISO 8601 date string. |
| DateOfPuttingIntoService | `dpp:dateOfPuttingIntoService` (string) | Optional (cardinality 0..1). ISO 8601 date string. |

##### Property: DateOfManufacture

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `DateOfManufacture` | `xs:date` | Mandatory. AAS semanticId: `0112/2///61987#ABB757#007`. Must comply with ISO 8601-1:2020. |
| OpenUSD | `dpp:dateOfManufacture` | `string` | USD has no native date type; ISO 8601 string convention |

---

### DateOfPuttingIntoService

See [DateOfManufacture](#dateofmanufacture) table above.

---

### UniqueFacilityIdentifier

A unique string identifying the manufacturing location. Per DBP, the manufacturing place should be uniquely identifiable. This is a **location identifier**, distinct from the product identifier (`URIOfTheProduct`) and the manufacturer identifier (`ManufacturerIdentifier`). The Identifier Separation of Concerns proposal explicitly categorizes these as separate concerns.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| UniqueFacilityIdentifier | `dpp:uniqueFacilityIdentifier` (string) | Mandatory (cardinality 1). |

##### Property: UniqueFacilityIdentifier

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `UniqueFacilityIdentifier` | `xs:string` | Mandatory. AAS semanticId: `https://admin-shell.io/idta/nameplate/3/0/UniqueFacilityIdentifier` |
| OpenUSD | `dpp:uniqueFacilityIdentifier` | `string` | Location/facility scope identifier |

---

### LifeCycleStage

> **Schema:** `BatteryNameplateAPI` (battery-specific addition from IDTA-02035)

Indicates the current lifecycle state of the battery. The DBP defines five enumerated values derived from ECLASS IRDIs. In USD, `token` type with `allowedTokens` metadata is the natural fit for enumerated values.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| LifeCycleStage | `dpp:lifeCycleStage` (token) | Mandatory (cardinality 1). Enumerated. |

##### Property: LifeCycleStage

|  | Name | Data Type | Allowed Values |
|---|---|---|---|
| DBP | `LifeCycleStage` | `xs:string` (ECLASS IRDI enum) | `0173-1#07-ACC020#001` (original), `0173-1#07-ACC021#001` (repurposed), `0173-1#07-ACC022#001` (re-used), `0173-1#07-ACC023#001` (remanufactured), `0173-1#07-ACC024#001` (waste) |
| OpenUSD | `dpp:lifeCycleStage` | `token` | `"original"`, `"repurposed"`, `"re-used"`, `"remanufactured"`, `"waste"` |

The IRDI-to-token mapping is:

| ECLASS IRDI | USD Token |
|---|---|
| `0173-1#07-ACC020#001` | `original` |
| `0173-1#07-ACC021#001` | `repurposed` |
| `0173-1#07-ACC022#001` | `re-used` |
| `0173-1#07-ACC023#001` | `remanufactured` |
| `0173-1#07-ACC024#001` | `waste` |

---

### OperatorIdentifier

> **Schema:** `BatteryNameplateAPI` (battery-specific addition from IDTA-02035)

Optional identifier of the battery operator per ISO/IEC 15459 series.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| OperatorIdentifier | `dpp:operatorIdentifier` (string) | Optional (cardinality 0..1). |

---

### ManufacturerIdentifier

> **Schema:** `BatteryNameplateAPI` (battery-specific addition from IDTA-02035)

A formal identifier for the manufacturer (distinct from the human-readable `ManufacturerName`). This is a **manufacturer-scoped identifier** and is a strong candidate for `assetInfo["name"]` in USD, or for a scoped attribute in an applied schema. See [Appendix A](#appendix-a-identifier-encoding-strategies) for the trade-off analysis.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| ManufacturerIdentifier | `assetInfo["name"]` or `dpp:manufacturerIdentifier` (string) | Mandatory (cardinality 1). Formal manufacturer ID. |

##### Property: ManufacturerIdentifier

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `ManufacturerIdentifier` | `xs:string` | Mandatory. AAS semanticId: `urn:samm:io.adminshell.idta.batterypass.technical_data:1.0.0#manufacturerIdentifier` |
| OpenUSD (option A) | `assetInfo["name"]` | `string` | Reuses USD's established `assetInfo` convention for asset name; may conflate with human-readable name |
| OpenUSD (option B) | `dpp:manufacturerIdentifier` | `string` | Schema-namespaced; preserves semantic separation from `ManufacturerName` |

---

### Markings

The `Markings` SML contains one or more `Marking` SMCs, each representing a regulatory or conformity symbol on the physical battery label (e.g., CE marking, WEEE symbol, carbon footprint label, separate collection symbol). Each marking has a name, optional certificate designation, optional validity dates, an optional image file, and optional additional text.

In USD, each marking is best represented as a **child prim** under a `Markings` scope, because:
- Markings have heterogeneous optional fields
- Marking image files (`MarkingFile`) are naturally `asset`-typed attributes
- Multiple markings need to be independently addressable

#### Composition

```
Battery_A12345X75EN (Xform, DppNameplateAPI + BatteryNameplateAPI applied)
└── Markings (Scope)
    ├── Marking_CE (Scope, MarkingAPI applied)
    └── Marking_WEEE (Scope, MarkingAPI applied)
```

#### Properties (per Marking)

| DBP (Markings__00__) | OpenUSD | Description |
|---|---|---|
| MarkingName | `marking:name` (token or string) | Mandatory. IRDI or plain string for the marking type. |
| DesignationOfCertificateOrApproval | `marking:certificateDesignation` (string) | Optional. Certificate or approval number. |
| IssueDate | `marking:issueDate` (string, ISO 8601) | Optional. Certificate issue date. |
| ExpiryDate | `marking:expiryDate` (string, ISO 8601) | Optional. Certificate expiry date. |
| MarkingFile | `marking:file` (asset) | Optional. Image file of the conformity symbol. |
| MarkingAdditionalText | `marking:additionalText` (string[]) | Optional, multi-value. Explanatory text (e.g., notified body ID). |

##### Property: MarkingName

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `MarkingName` | `xs:string` (IRDI preferred) | Mandatory. AAS semanticId: `0112/2///61987#ABA231#009`. Preferred value: IRDI from IEC CDD/ECLASS (e.g., CE = `0173-1#07-DAA603#004`). |
| OpenUSD | `marking:name` | `token` | Token for IRDI-sourced values; `string` fallback for free-text names |

#### Usage Example (USDA)

```usda
def Xform "Battery_A12345X75EN" (
    prepend apiSchemas = ["DppNameplateAPI", "BatteryNameplateAPI"]
) {
    def Scope "Markings" {

        def Scope "Marking_CE" (
            prepend apiSchemas = ["MarkingAPI"]
        ) {
            token marking:name = "0173-1#07-DAA603#004"
            string marking:certificateDesignation = "KEMA99IECEX1105/128"
            string marking:issueDate = "2022-01-01"
            string marking:expiryDate = "2028-01-01"
            asset marking:file = @./markings/marking_ce.png@
            string[] marking:additionalText = ["0123"]
        }

        def Scope "Marking_WEEE" (
            prepend apiSchemas = ["MarkingAPI"]
        ) {
            token marking:name = "WEEE"
            asset marking:file = @./markings/WEEE.png@
        }

    }
}
```

---

### EUDeclarationOfConformity

> **Schema:** `BatteryNameplateAPI` (battery-specific addition from IDTA-02035)

A battery passport must include the EU Declaration of Conformity. In AAS, this is a SubmodelElementList of document identifier strings that reference documents in the Handover Documentation submodel (IDTA-02035-2). In USD, the documents are encoded as an array of `asset`-typed paths.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| EUDeclarationOfConformity (SML of string DocumentIdentifiers) | `dpp:euDeclarationOfConformity` (asset[]) | Mandatory (cardinality 1). One or more asset paths or URIs to EU DoC documents. |

##### Property: EUDeclarationOfConformity

|  | Name | Data Type | Notes |
|---|---|---|---|
| DBP | `[00] DocumentIdentifier` | `xs:string` | AAS semanticId: `urn:samm:io.adminshell.idta.batterypass.digital_nameplate:1.0.0#euDeclarationOfConformity` |
| OpenUSD | `dpp:euDeclarationOfConformity` | `asset[]` | Document identifiers are encoded as SdfAssetPaths (file paths or URIs) |

---

### ResultsOfTestReportsProvingCompliance

> **Schema:** `BatteryNameplateAPI` (battery-specific addition from IDTA-02035)

A battery passport must include test report results proving regulatory compliance. Encoded identically to `EUDeclarationOfConformity`.

#### Properties

| DBP | OpenUSD | Description |
|---|---|---|
| ResultsOfTestReportsProvingCompliance (SML of string DocumentIdentifiers) | `dpp:testReportCompliance` (asset[]) | Mandatory (cardinality 1). One or more asset paths or URIs to compliance test reports. |

---

## Appendices

### Appendix A: Identifier Encoding Strategies

The DBP Nameplate contains four distinct identifier fields:

| Field | Identifier Scope | Cardinality |
|---|---|---|
| `URIOfTheProduct` | Per-instance battery passport URI | 1 (mandatory) |
| `SerialNumber` | Per-instance serial number | 1 (mandatory) |
| `ManufacturerIdentifier` | Manufacturer scope | 1 (mandatory) |
| `UniqueFacilityIdentifier` | Manufacturing location | 1 (mandatory) |
| `OperatorIdentifier` | Operator scope | 0..1 (optional) |

These fields are directly relevant to the **Identifier Separation of Concerns** proposal, which argues that USD should provide a clear mechanism for carrying multiple, semantically distinct identifiers from external systems—not conflating them into a single `assetInfo` field.

Two approaches from the proposal apply here:

#### Approach A: `assetInfo` Dictionary

Place identifiers in the `assetInfo` dictionary using namespaced keys:

```usda
assetInfo = {
    asset identifier = @https://dcqr.com/?m=R123456789@
    string name = "XYZ-123456"
    dictionary "battery" = {
        string serialNumber = "A12345-X75EN"
        string uniqueFacilityIdentifier = "987654321"
        string operatorIdentifier = "123456789"
    }
}
```

**Pros:** Uses USD's established convention for asset metadata; accessible via `UsdModelAPI::GetAssetInfo()`.  
**Cons:** `assetInfo` has weak typing; no schema enforcement; the `identifier` and `name` keys have informal semantics that may not align well with DBP's more specific notions of product URI vs. manufacturer identifier.

#### Approach B: Applied API Schema Attributes

Place all identifiers as typed attributes across the two schemas (namespaced `dpp:`):

```usda
def Xform "Battery_A12345X75EN" (
    prepend apiSchemas = ["DppNameplateAPI", "BatteryNameplateAPI"]
) {
    # DppNameplateAPI — generic product identifiers
    asset dpp:uriOfTheProduct = @https://dcqr.com/?m=R123456789@
    string dpp:serialNumber = "A12345-X75EN"
    string dpp:uniqueFacilityIdentifier = "987654321"

    # BatteryNameplateAPI — battery-specific identifiers
    string dpp:manufacturerIdentifier = "XYZ-123456"
    string dpp:operatorIdentifier = "123456789"
}
```

**Pros:** Strongly typed; schema-enforced; semantically distinct fields; discoverable via schema introspection; no ambiguity between product URI and manufacturer name.  
**Cons:** Requires schema registration; not discoverable via `UsdModelAPI::GetAssetInfo()` without additional tool support.

#### Recommendation

A hybrid approach is recommended:

- `assetInfo["identifier"]` = `URIOfTheProduct` (globally unique, per-instance; this is the most natural fit for USD's `identifier` key)
- `dpp:serialNumber`, `dpp:uniqueFacilityIdentifier` → `DppNameplateAPI` schema attributes (generic product identifiers)
- `dpp:manufacturerIdentifier`, `dpp:operatorIdentifier` → `BatteryNameplateAPI` schema attributes (battery-specific identifiers)

This preserves compatibility with USD tools that consume `assetInfo["identifier"]` for asset tracking, while keeping generic and battery-specific identifier concerns in their respective schemas.

---

### Appendix B: AAS-to-USD Type Mapping

| AAS Type | AAS Description | USD Type | Notes |
|---|---|---|---|
| `xs:string` | Text string | `string` | Direct mapping |
| `xs:anyURI` | Universal Resource Identifier | `asset` (SdfAssetPath) | `asset` accommodates both file paths and URIs |
| `xs:date` | ISO 8601 date | `string` | USD has no native date type; use `YYYY-MM-DD` string convention |
| `MLP` (MultiLanguageProperty) | Per-language-code string map | `string` | Store primary value; language variants in `customData` if needed |
| `File` | Binary file with MIME type | `asset` | SdfAssetPath to file; MIME type stored in `customData` if needed |
| `SMC` (SubmodelElementCollection) | Named group of elements | Namespaced attribute prefix or child `Scope` prim | Child prim preferred for complex/reusable SMCs |
| `SML` (SubmodelElementList, typed elements) | Ordered list of same-typed elements | Array attribute (e.g., `asset[]`, `string[]`) or child prims | Array preferred for scalar lists; child prims for complex element types |
| `token` (ECLASS IRDI enumeration) | IRDI-coded enumeration value | `token` with `allowedTokens` | Map IRDI to human-readable token string; preserve IRDI in `customData` |
