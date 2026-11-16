# RFC-8 revision: summary of changes

**Status:** Proposed revisions, Fall 2026

**Scope:** Changes to RFC-8 (Collections and Extensibility) following the round-1 review, Damien Goutte-Gattat's extensibility proposal and Benedikt Best's comment in [ome/ngff#627](https://github.com/ome/ngff/pull/627).

This document summarizes every change between the round-1 RFC-8 text and the revised text in [`upstream/rfc8.md`](upstream/rfc8.md). Each section states the new rule, what it replaces, and an example. The revised RFC text is normative; where this summary and the RFC disagree, the RFC wins. Topics that move to separate sub-RFCs and points left for future work are listed at the end.

---

## 1. Overview

| Area | Round-1 RFC-8 | Revised RFC-8 |
|---|---|---|
| Node types | collection, multiscale, singlescale | collection, multiscale. **Singlescale is removed** and deferred to a sub-RFC. |
| Type-specific metadata | `attributes` dictionary | **`traits`** list of typed objects |
| Free-form metadata | `attributes` | **Removed.** Everything beyond the node fields is a trait. |
| Extensions | Prefixed names (`myorg:foo`) | **URI-based types** declared in an **`extensionManifest`** |
| Type tiers | — | Built-in types are **core** (MUST), **recommended** (SHOULD) or **optional** (MAY); custom types start as optional |
| IDs | Optional on nodes, unique per document | **Required on every declaration**, unique per document, restricted character set, UUIDv7 recommended if UUIDs are used |
| Names | Required, unique within the collection | **Optional**, not required to be unique |
| References | `{"id": …}` | **`{"type": …, "ref": …}`**, plus an optional `path`, or a **plain ID string** where the field fixes the target type |
| Paths | `{"type": "zarr", "path": …}` | **`{"type": "zarr", "format": "relative", "href": …}`** |
| Multiscale layout | Singlescale child nodes | **`data`** list of **`array`** objects with **`downsamplingTransformations`** |
| Multiscales in a collection | Metadata may be inlined | Unchanged: a multiscale node has **either `data` or `path`** (see §8.1) |
| Consolidated metadata | — | Non-normative section: an optional cache for IO performance |
| HCS, labels, bioformats2raw, remote references, snake_case, singlescale, legacy multiscales | In RFC-8 or not covered | **Deferred to sub-RFCs** |

---

## 2. Node interface and node types

RFC sections: [`Node` interface](upstream/rfc8.md#node-interface), [`Collection` node](upstream/rfc8.md#collection-node), [`Multiscale` node](upstream/rfc8.md#multiscale-node).

### 2.1 Node interface

Every node is a [declaration](#71-declarations) and has:

| Field | Required | Description |
|---|---|---|
| `type` | yes | Node type: a keyword for built-in types, an extension-derived identifier for custom types |
| `id` | yes | See §6 |
| `name` | no | Non-empty, human-readable label, not required to be unique |
| `traits` | no | List of trait objects (see §3) |

Fields that a node type requires sit directly on the node, for example `coordinateSystems` and `data` on a multiscale, or `nodes` on a collection.

The `attributes` dictionary is removed. There is no free-form metadata field. Any additional information must be expressed as a trait.

A node used as the root object of the `ome` key also has a `version` field. Non-root nodes SHOULD NOT have a `version` field and MUST NOT have a different `version` than the root node.

### 2.2 Node types

- **Core node types:** `collection` and `multiscale`. The node type defines the structure of a node: which fields it has and how readers traverse it and resolve its children.
- A collection has either `nodes` or a `path`, but not both. A multiscale has either `data` or a `path`, but not both.
- If a node has a `path`, the document it resolves to MUST have a root node of the same type.
- **`singlescale` is removed.** Resolution levels are `array` objects inside the multiscale (see §8). A node type for images with a single resolution level is deferred to a sub-RFC (see §11).
- `array` is not a node type: `array` objects cannot appear in `nodes` and cannot be the root of the `ome` key. A multiscale is a leaf node.
- Extensions MAY define additional node types (see §4), but SHOULD define traits instead when they add functions to existing structures.

---

## 3. Traits

RFC section: [Traits](upstream/rfc8.md#traits).

Traits replace `attributes`. The design is inspired by Rust traits: shared behaviour that a type opts into and combines freely, instead of inheriting it from a parent type.

### 3.1 Rules

- `traits` is a list of objects. Each trait has a `type`: a keyword for built-in traits, an extension-derived identifier for custom traits.
- Node types describe structure; traits describe function, i.e. what the data represents and how it relates to other nodes.
- Traits are composable. A node can carry any number of traits, so independent aspects can be combined without defining a new node type for each combination.
- A trait's fields sit inside the trait object, not on the node.
- Both collections and multiscales can carry traits.
- Readers that do not understand a trait MAY ignore it and still traverse the node based on its node type. Readers MUST preserve data from unknown extensions and pass it through to client code.
- If the trait is a custom type from an extension with `mustUnderstand: true`, readers MUST NOT silently fall back to a generic interpretation of the node that carries it (see §4.3).
- For now, any trait can be used on any node. Rules about which traits can be used on which node types, or combined with each other, are future work (see §12).

### 3.2 The `scene` trait

RFC-8 defines one built-in trait, `scene`. It is used on a collection to hold the spatial relationships between the nodes of the collection:

| Field | Required | Description |
|---|---|---|
| `coordinateSystems` | no | Coordinate systems declared by the scene |
| `coordinateTransformations` | yes | Transformations between coordinate systems of the scene and of its nodes |

Spatial information of an individual multiscale stays in the multiscale's own `coordinateSystems` and `coordinateTransformations` fields.

Other traits such as `plate`, `well` and `labels` are specified in sub-RFCs (see §11).

### 3.3 Example

```json
{
  "type": "collection",
  "id": "000001",
  "name": "Tiles",
  "traits": [
    {
      "type": "scene",
      "coordinateSystems": [
        { "type": "coordinate_system", "id": "000002", "name": "world", "axes": [ ... ] }
      ],
      "coordinateTransformations": [
        {
          "type": "translation",
          "translation": [0, 0, 100],
          "input": {
            "type": "coordinate_system",
            "ref": "000005",
            "path": { "type": "zarr", "format": "relative", "href": "./tile_0.zarr" }
          },
          "output": "000002"
        }
      ]
    }
  ],
  "nodes": [
    {
      "type": "multiscale",
      "id": "000003",
      "name": "Tile 0",
      "path": { "type": "zarr", "format": "relative", "href": "./tile_0.zarr" }
    }
  ]
}
```

---

## 4. Extension mechanism

RFC section: [Extensions](upstream/rfc8.md#extensions). Based on Damien Goutte-Gattat's extensibility proposal.

### 4.1 Identifiers and extension points

- An extension MUST have a unique identifier, which SHOULD be an absolute URI ending in `#` or `/`.
- Every custom type defined by an extension is identified by appending its local name to the extension identifier, e.g. `https://example.org/my-ngff-extension/NonLinearTransform`. This replaces prefixed names such as `napari:rendering`, so independently developed extensions cannot clash and no central prefix registry is needed.
- Custom types can be used at **extension points**: `type` fields that the specification explicitly marks as extensible. These are node types, trait types, path types, coordinate transformation types and axis types.
- Extensions that add functions to existing structures SHOULD define new trait types rather than new node types.

### 4.2 Extension manifest

Every JSON document that uses custom types MUST declare an **`extensionManifest`** under the `ome` key, listing every extension used in that document. When the metadata is spread over several documents, each document carries its own manifest, and the root document SHOULD list every extension used anywhere below it.

Each manifest entry has:

| Field | Required | Description |
|---|---|---|
| `id` | yes | Extension identifier, unique within the manifest |
| `version` | yes | Version of the extension used in the document |
| `name` | no | Short display name |
| `description` | no | One-sentence description |
| `mustUnderstand` | no | Whether the extension must be understood for faithful processing (default `false`) |
| `homepage` | no | Homepage or documentation URL |
| `schemas` | yes | List of `{ "type": "jsonschema" \| "linkml", "url": … }` |

- An extension MUST provide at least one schema. Each custom type is described by a `$defs` entry (JSON Schema) or a class (LinkML) named after its local name.
- Readers can list every extension a file uses, with its schemas, without a central registry. Readers that don't support an extension MAY use its schemas to validate or display its data.

### 4.3 The `mustUnderstand` flag

- Writers SHOULD set `mustUnderstand: true` only when ignoring the extension would cause an *incorrect* interpretation of the data, e.g. a coordinate transformation the reader cannot apply, not merely a *less complete* one, e.g. display settings.
- A reader that meets an unsupported custom type from such an extension MUST NOT silently fall back to a generic interpretation. It MAY skip the object and report it as unsupported, and MAY continue with the rest of the dataset. For a trait, this applies to the node that carries it.
- Readers MAY use the manifest to decide up front whether a dataset is likely to be fully supported.

The design of the flag (boolean vs. levels, per-extension vs. per-type, whether it is needed at all) is listed as future work in the RFC.

### 4.4 Example

```json
{
  "ome": {
    "version": "0.x",
    "extensionManifest": [
      {
        "id": "https://viewer-b.example.org/rendering/",
        "version": "0.2",
        "name": "Viewer-B visualisation settings",
        "homepage": "https://viewer-b.example.org/",
        "schemas": [
          { "type": "jsonschema", "url": "https://viewer-b.example.org/ngff-embedded-settings.json" }
        ]
      }
    ],
    "type": "collection",
    "id": "000001",
    "nodes": [
      {
        "type": "multiscale",
        "id": "000002",
        "name": "nuclei",
        "path": { "type": "zarr", "format": "relative", "href": "./nuclei" },
        "traits": [
          { "type": "https://viewer-b.example.org/rendering/Display", "colormap": "blue", "contrastLimits": [110, 3800] }
        ]
      }
    ]
  }
}
```

---

## 5. Type tiers

RFC section: [Type tiers](upstream/rfc8.md#type-tiers).

Every type is either **built-in** (defined by the specification, identified by a keyword) or **custom** (defined by an extension, identified by an extension-derived identifier).

| Tier | Requirement | Examples |
|---|---|---|
| **Core** built-in types | MUST be supported | node types `collection`, `multiscale`; path types `zarr`, `json`; trait `scene`; `array`; axis types; transformations `identity`, `scale`, `translation` |
| **Recommended** built-in types | SHOULD be supported | transformations `mapAxis`, `affine`, `rotation` |
| **Optional** types | MAY be supported | transformations `sequence`, `displacements`, `coordinates`, `bijection`, `byDimension`; all custom types |

- Custom types start as optional. A future RFC may promote a custom type to a recommended type.
- Every future RFC that defines types MUST state the tier of each type. Types without such a statement are optional.

---

## 6. IDs and names

RFC section: [IDs](upstream/rfc8.md#ids).

- `id` is **required** on every declaration: nodes and coordinate systems.
- IDs MUST be non-empty strings using only `a-z`, `A-Z`, `0-9`, `-`, `_` and `.`.
- IDs MUST be unique within the JSON document.
- IDs MAY be UUIDs. If so, they SHOULD be UUIDv7 or later, e.g. `01a13520-b697-70c4-979b-b52c01fbbe30`.
- IDs SHOULD NOT encode semantics such as the name, type or location of the object, because these may change while the ID must stay stable. Human-readable information belongs in `name`.
- `name` is **optional**. The rule that names must be unique within the enclosing collection is removed.

The examples in the RFC use short IDs such as `"000001"` for readability.

---

## 7. Declarations, references and paths

RFC sections: [Building blocks](upstream/rfc8.md#building-blocks).

### 7.1 Declarations

A declaration is the one place where an object's `id` and its other fields are defined. Declarations appear in arrays such as `nodes` and `coordinateSystems`.

- **Every declaration carries `type`**, which is the object's own kind: for example `"collection"` or `"multiscale"` for nodes and `"coordinate_system"` for coordinate systems.
- Every declaration carries an `id` (see §6).
- Fields, traits and child nodes are defined on the declaration.

### 7.2 References

A reference points to an object declared elsewhere, e.g. in the `input` and `output` fields of coordinate transformations. A reference is either an object (full form) or a string (short form).

**Full form:**

- **`type`**: the type of the target declaration, for example `"coordinate_system"` or `"multiscale"`.
- **`ref`**: the `id` of the target. References use `ref`, not `id`, so a reference can always be told apart from a declaration.
- **`path`** (optional): a path object locating the document that contains the target.
- No other fields. In particular, a reference MUST NOT carry traits or child nodes.

**Short form:** a plain ID string. It MAY only be used where the field already fixes the type of the target (e.g. `output` of a transformation is always a coordinate system), and only for targets in the same document. It is equivalent to the full form with that type and no `path`.

**Resolution:** without a `path`, `ref` resolves in the current document; with a `path`, in the document the path locates. The target MUST exist and MUST have the expected `type`; otherwise implementations MUST report an error.

Prior art: OME-XML (`…Ref` elements), FHIR (`reference`) and STAC (`href`) also separate declaration and reference keys.

```json
"output": "000002"
```

```json
"output": { "type": "coordinate_system", "ref": "000002" }
```

Reference into another document:

```json
"input": {
  "type": "coordinate_system",
  "path": { "type": "zarr", "format": "relative", "href": "./raw" },
  "ref": "000005"
}
```

**Coordinate systems get IDs and are referenced by ID**, not by name as in RFC-5.

### 7.3 Paths

Path objects have the form:

```json
{ "type": "zarr", "format": "relative", "href": "./raw" }
```

- `path.path` is renamed to **`href`**.
- **`type`** is `zarr` (a Zarr group or array; `zarr.json` is appended to access its metadata) or `json` (a standalone JSON file). Path types are an extension point.
- **`format`** is explicit. Readers MUST NOT infer it from the string, for example from an `https://` prefix.
- Only `relative` is defined. Readers that do not support a path's format MUST NOT resolve it and SHOULD treat the target as unavailable. Further formats, such as absolute paths, S3 URLs or chained paths into archives, are left to future RFCs (see §11).
- Relative paths resolve from the document that contains them and follow IETF RFC 1808 notation. The storage protocol (file system, HTTP, S3, …) is that of the containing document.
- **Typed paths:** the document at a node's `path` MUST have a root node of the same node type.
- **No cycles:** resolving node paths MUST NOT form a cycle, and implementations MUST detect cycles and report an error. The `path` of a reference is exempt: it may point back to an enclosing document.

---

## 8. Multiscales

RFC sections: [`Array` object](upstream/rfc8.md#array-object), [`Multiscale` node](upstream/rfc8.md#multiscale-node), [Coordinate transformations](upstream/rfc8.md#coordinate-transformations).

### 8.1 Inline or referenced

A multiscale node has either `data` (the multiscale metadata inline) or a `path` to the image's own document, but not both. Inside a collection, both forms remain allowed, so a collection may inline multiscale metadata (see "A collection with an inlined multiscale" and "Redundant metadata" in the RFC).

The revision discussions favoured always referencing multiscales by path (one source of truth, self-contained images, small collections). The RFC text does not currently require this.

### 8.2 Multiscale layout

- The 0.6 `multiscales` array is replaced by a single multiscale per node.
- **`coordinateSystems`** sit directly on the multiscale node, each with `"type": "coordinate_system"` and an `id`.
- **`data`** replaces 0.6 `datasets` and the round-1 singlescale child nodes. It lists **`array`** objects, one per resolution level, each with `"type": "array"`, an optional `name` and a path object. Array objects are not declarations and have no `id`.
- **`coordinateTransformations`** on the multiscale map the highest-resolution array into a coordinate system:
  - `input` MUST reference the enclosing multiscale (type `multiscale`), standing for its highest-resolution array.
  - `output` MUST reference a coordinate system in the multiscale's `coordinateSystems`, and all transformations MUST have the same `output`.
  - Both SHOULD use the short (string) form.
- Every array other than the highest-resolution one MUST have **`downsamplingTransformations`**: exactly one `scale`, optionally followed by one `translation`, from this array to the highest-resolution array. They have no `input` or `output` fields; these are implicit. They can be cast into a regular coordinate transformation (a `scale`, or a `sequence` of `scale` and `translation`).
- The multiscale MAY carry `traits`.

### 8.3 Example

Collection, `zarr.json`:

```json
{
  "ome": {
    "version": "0.x",
    "type": "collection",
    "id": "000001",
    "nodes": [
      {
        "type": "multiscale",
        "id": "000002",
        "name": "raw",
        "path": { "type": "zarr", "format": "relative", "href": "./raw" }
      }
    ]
  }
}
```

Multiscale, `raw/zarr.json`:

```json
{
  "ome": {
    "version": "0.x",
    "type": "multiscale",
    "id": "000001",
    "name": "raw",
    "coordinateSystems": [
      {
        "type": "coordinate_system",
        "id": "000002",
        "name": "physical",
        "axes": [
          { "name": "z", "type": "space", "unit": "micrometer" },
          { "name": "y", "type": "space", "unit": "micrometer" },
          { "name": "x", "type": "space", "unit": "micrometer" }
        ]
      }
    ],
    "coordinateTransformations": [
      { "type": "scale", "scale": [0.9, 0.45, 0.45], "input": "000001", "output": "000002" }
    ],
    "data": [
      {
        "type": "array",
        "path": { "type": "zarr", "format": "relative", "href": "./s0" }
      },
      {
        "type": "array",
        "path": { "type": "zarr", "format": "relative", "href": "./s1" },
        "downsamplingTransformations": [
          { "type": "scale", "scale": [1, 2, 2] },
          { "type": "translation", "translation": [0, 0.5, 0.5] }
        ]
      }
    ]
  }
}
```

IDs are unique per document, so the multiscale node in the collection and the root of `raw/zarr.json` do not need to share an ID.

### 8.4 Remaining differences from 0.6

`data` with `array` objects instead of `datasets`, path objects instead of path strings, downsampling transformations relative to the highest-resolution array instead of per-dataset transformations into a coordinate system, `type` and `id` on coordinate systems, coordinate systems referenced by ID instead of name, and one multiscale per node instead of the `multiscales` array.

---

## 9. Metadata storage and consolidated metadata

RFC section: [Metadata storage](upstream/rfc8.md#metadata-storage).

Node metadata is stored under the `ome` key, either in the `attributes` of a Zarr group's `zarr.json`, or at the root of a standalone `.json` file.

A non-normative section on IO performance is added:

- Zarr consolidated metadata MAY be used to bundle the `zarr.json` documents of referenced nodes into the root, so readers can load the whole tree in one request. This also applies to nested collections, which stay separate documents.
- The **individual `zarr.json` files remain the source of truth**. The consolidated copy is only a cache and can go stale.
- Keeping the cache consistent with the individual files would require an external transaction system, such as Icechunk or a custom database.
- It is pure Zarr with no OME-specific rules: readers without support fetch each node. It only covers Zarr nodes within the root group's hierarchy; standalone JSON files are always fetched individually.
- The Zarr-level specification of consolidated metadata is currently incomplete.

---

## 10. Naming conventions

Keys use camelCase as in previous versions (`coordinateSystems`, `coordinateTransformations`, `downsamplingTransformations`, `extensionManifest`, `mustUnderstand`). Built-in `type` values such as `coordinate_system` use snake_case. Moving keys to snake_case is left to the snake_case sub-RFC (see §11) and is not changed by this revision.

---

## 11. Deferred to future sub-RFCs

These topics are removed from RFC-8 and will be specified in separate sub-RFCs, using the RFC-8 building blocks. All of them go through the RFC process before the v1.0 specification:

| Topic | Scope |
|---|---|
| **HCS** | Plate and well traits |
| **Labels** | Label image traits and per-label properties |
| **bioformats2raw and OME-XML** | Series layout and the OME/METADATA.ome.xml mapping |
| **Absolute and remote references** | Absolute paths and URLs, authentication and security |
| **snake_case** | Making keys and identifiers consistent with Zarr |
| **Legacy multiscales** | A `legacy_multiscale` node type for unchanged 0.5 and 0.6 images |
| **Singlescale images** | A node type for images with a single resolution level |

### Legacy multiscales (proposal for the sub-RFC)

A `legacy_multiscale` node mounts an existing 0.5 or 0.6 image in a collection without converting it:

```json
{
  "type": "legacy_multiscale",
  "id": "000005",
  "path": { "type": "zarr", "format": "relative", "href": "./scan_2023" },
  "traits": [ ... ]
}
```

- Existing data stays untouched, including `omero` and `OME/METADATA.ome.xml`.
- The target is read with the rules of its own version. Traits on the declaring node add to it.
- Coordinate systems inside the legacy image are addressed by name in 0.6, or as the implicit one in 0.5.

---

## 12. Future work

The RFC lists these under "Future possibilities":

- **Limiting where traits can be used.** Currently any trait can be used on any node. A system (to be designed) could state which traits apply to which node types, which traits exclude each other, and constraints on child nodes (e.g. a plate's children are wells).
- **Revising `mustUnderstand`:** more requirement levels, per-type granularity, or dropping it.
- **Extension management:** e.g. a community namespace in which extension IDs are minted only with community approval.
- **Additional node types and path formats**, e.g. tables, meshes and S3 URLs.
- **User story 11 (training crops):** a `source` trait ("derived from") and a node type for non-OME-Zarr data; traits for per-level extent and sparse chunks; tying a node's `id` to provenance records.
- Rendering state, relationships between nodes, and the URL pipeline specification.

---

## 13. Other additions to the RFC text

- **Extensibility rationale and desired characteristics:** a condensed form of the Extensibility Manifesto.
- **User story 11:** training crops cut from non-OME-Zarr sources.
- **Abandoned ideas:** custom types identified by prefixed keywords, prefixed attribute keys, and subtyped node types (vs. traits).
- **Prior art:** Zarr Extensions & Zarr Conventions, Rust traits, extensions, separate declaration and reference keys.
- **Security:** narrowed to the `relative` path format (path traversal, and resource exhaustion through cycles). Considerations for absolute and remote paths move to the sub-RFC.

---

## 14. Feedback addressed

| Feedback | Resolution |
|---|---|
| `attributes` mixes required, optional and extension metadata | Replaced by typed traits; node-type fields on the node (§2, §3) |
| Node subtypes inferred by inspecting `attributes` | Typed traits with schemas (§3) |
| Prefix namespacing is weak; no schema discovery | URI-based types and `extensionManifest` (§4) |
| Extensions should be a separate RFC | Extension mechanism specified in RFC-8; content types moved to sub-RFCs (§4, §11) |
| Name uniqueness is hard to validate | Names optional, not required to be unique (§6) |
| Required / unique IDs, ideally UUIDs | IDs required on every declaration; UUIDv7 recommended where UUIDs are used (§6) |
| References point to different kinds of objects depending on context | Typed references with `type` and `ref` (§7.2) |
| `path.path` reads awkwardly; path format inferred from the string | `href` and explicit `format` (§7.3) |
| No rules for inline vs referenced metadata conflicts | Not resolved: inlining is still allowed; see "Redundant metadata" in the RFC (§8.1) |
| Purpose of singlescale unclear; transform split confusing | Singlescale removed; `array` objects with downsampling transformations relative to the highest resolution (§8.2) |
| Scaling to many nodes; IO performance | Consolidated metadata as an optional cache (§9) |
| snake_case | Deferred to sub-RFC (§10, §11) |
| HCS gaps, label colour metadata, bioformats2raw / OME-XML upgrade path | Deferred to sub-RFCs (§11) |
| Security and portability of remote references | Deferred to sub-RFC (§11) |
| No upgrade path for existing 0.5 / 0.6 data | `legacy_multiscale` proposed for a sub-RFC (§11) |
