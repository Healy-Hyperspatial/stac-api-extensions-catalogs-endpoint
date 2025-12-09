# STAC API - Catalogs Endpoint Extension

- **Title:** Catalogs Endpoint
- **Conformance Classes:**
  - `https://api.stacspec.org/v1.0.0/core` (required)
  - `https://api.stacspec.org/v1.0.0-beta.1/catalogs-endpoint` (required)
- **Scope:** STAC API - Core
- **Extension Maturity Classification:** Proposal
- **Dependencies:**
  - [STAC API - Core](https://github.com/radiantearth/stac-api-spec/blob/main/core)
  - [STAC API - Collections](https://github.com/radiantearth/stac-api-spec/tree/main/ogcapi-features)
  - [STAC API - Transaction](https://github.com/radiantearth/stac-api-spec/tree/main/ogcapi-features/extensions/transaction) (Reference pattern)
- **Owner**: @jonhealy1

## Introduction

This extension enables a **Federated STAC API** architecture. It adds a dedicated **`/catalogs` endpoint** that serves as a machine-readable registry for multiple independent data providers, without altering the standard behavior of the API Root.

In addition to discovery, this extension defines **Transactional** endpoints to create, update, and delete Catalogs and their child Collections, effectively acting as a management API for the federation.

In this model, the API has a fixed-depth "Hub and Spoke" structure:
1.  **Global Root (`/`)**: The standard entry point. It remains a clean STAC Landing Page but includes a link to the **Catalogs Registry**.
2.  **The Registry (`/catalogs`)**: A machine-readable list of all available Sub-Catalogs.
3.  **Sub-Catalogs (`/catalogs/{id}`)**: The actual data providers. These behave as standard STAC API Landing Pages containing Collections.

## Endpoints

### Discovery (Read-Only)
| Method | URI | Description |
| :--- | :--- | :--- |
| `GET` | `/catalogs` | **The Registry.** Lists all available sub-catalogs. |
| `GET` | `/catalogs/{catalogId}` | **Sub-Catalog Root.** Acts as the Landing Page for the provider. |
| `GET` | `/catalogs/{catalogId}/conformance` | Conformance classes specific to this sub-catalog. |
| `GET` | `/catalogs/{catalogId}/queryables` | Filter Extension. Lists fields available for filtering in this sub-catalog. |
| `GET` | `/catalogs/{catalogId}/collections` | Lists collections belonging to this sub-catalog. |
| `GET` | `/catalogs/{catalogId}/collections/{collectionId}` | Gets a specific collection definition. |
| `GET` | `/catalogs/{catalogId}/collections/{collectionId}/items` | **Item Search.** Fetches items from this specific collection. |
| `GET` | `/catalogs/{catalogId}/collections/{collectionId}/items/{itemId}` | Gets a single specific item. |

### Transactions (Management)
These endpoints allow for the dynamic creation and deletion of the federation structure.

| Method | URI | Description |
| :--- | :--- | :--- |
| `POST` | `/catalogs` | **Create Catalog.** Registers a new sub-catalog. |
| `DELETE` | `/catalogs/{catalogId}` | **Delete Catalog.** Removes a sub-catalog. Supports `?cascade=true`. |
| `POST` | `/catalogs/{catalogId}/collections` | **Create Collection.** Creates a collection and links it to this catalog. |
| `DELETE` | `/catalogs/{catalogId}/collections/{collectionId}` | **Delete Collection.** Deletes the collection or removes the link from the parent catalog. |

## Poly-Hierarchy (Multi-Parenting)

This extension explicitly supports **Poly-hierarchy**, allowing a single STAC Collection to belong to **multiple Catalogs simultaneously**.

Unlike a standard file system where a folder can only live in one path, this architecture allows for logical grouping across different dimensions without duplicating data.

* **Example:** A `Sentinel-2` collection can be linked as a child of the `USGS` Catalog (Provider) AND the `Optical-Data` Catalog (Theme).
* **Behavior:** When accessing a Collection via a specific Catalog endpoint (e.g., `/catalogs/{id}/collections/{col_id}`), the API SHOULD preserve the navigation context by generating a `rel="parent"` link pointing back to that specific Catalog.

## Transaction Behavior

Implementations supporting the Transaction endpoints MUST adhere to the following behaviors:

### 1. Catalog Creation (`POST /catalogs`)
* **Body:** Accepts a standard [STAC Catalog](https://github.com/radiantearth/stac-spec/blob/master/catalog-spec/catalog-spec.md) JSON object.
* **Behavior:** The API creates the Catalog resource and makes it available in the `/catalogs` registry list.

### 2. Catalog Deletion (`DELETE /catalogs/{id}`)
* **Default Behavior (`cascade=false`):**
    * The Catalog object is deleted.
    * Child Collections are **Unlinked** (the catalog ID is removed from their parent list). They are NOT deleted. If a collection has no other parents, it becomes a root-level collection.
* **Cascade Behavior (`cascade=true`):**
    * The Catalog object is deleted.
    * All Child Collections linked to this catalog are **Deleted** from the database entirely. (Destructive operation).

### 3. Scoped Collection Creation (`POST /catalogs/{id}/collections`)
* **Body:** Accepts a standard [STAC Collection](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md) JSON object.
* **Behavior:**
    1.  Creates the Collection resource (or updates links if it exists).
    2.  **Automatic Linking:** Automatically registers the Collection as a child of `{catalogId}`.
    3.  **Reverse Linking:** Automatically adds a `rel="child"` link in the Catalog pointing to the new Collection.

### 4. Scoped Collection Deletion (`DELETE .../collections/{id}`)
* **Behavior:**
    * If the Collection belongs to **multiple** catalogs: It is unlinked from the current catalog only (Relationship delete).
    * If the Collection belongs to **only this** catalog: It is deleted from the database entirely (Resource delete).

## Link Relations

Proper linking is critical for clients to navigate the federation structure.

### 1. The Global Root (`/`)
This is the entry point.
- `rel="catalogs"`: MUST point to the `/catalogs` endpoint (the registry).
- `rel="service-desc"`: Points to the OpenAPI definition.

*Note: We use `rel="catalogs"` instead of `rel="data"` to avoid confusing standard clients that expect `data` to point to a Collections list.*

### 2. The Sub-Catalog (`/catalogs/{catalogId}`)
This resource acts as the **Landing Page** for the provider.
- `rel="self"`: MUST point to `/catalogs/{catalogId}`.
- `rel="parent"`: MUST point to the Global Root (`/`).
- `rel="root"`: SHOULD point to the Global Root (`/`) to maintain a single navigation tree.
- `rel="data"`: MUST point to `/catalogs/{catalogId}/collections`.

## Response Examples

### 1. The Registry List (`GET /catalogs`)

This endpoint returns a JSON object structurally similar to a standard `/collections` response, but it contains a list of **Catalog** objects in a `catalogs` array.

```json
{
  "catalogs": [
    {
      "id": "usgs-landsat",
      "type": "Catalog",
      "title": "USGS Landsat",
      "description": "Landsat collections provided by USGS.",
      "stac_version": "1.0.0",
      "links": [
        { "rel": "self", "href": "https://api.example.com/catalogs/usgs-landsat" },
        { "rel": "root", "href": "https://api.example.com/" },
        { "rel": "child", "href": "https://api.example.com/catalogs/usgs-landsat/collections" }
      ]
    },
    {
      "id": "esa-sentinel",
      "type": "Catalog",
      "title": "ESA Sentinel",
      "description": "Sentinel collections provided by ESA.",
      "stac_version": "1.0.0",
      "links": [
        { "rel": "self", "href": "https://api.example.com/catalogs/esa-sentinel" },
        { "rel": "root", "href": "https://api.example.com/" },
        { "rel": "child", "href": "https://api.example.com/catalogs/esa-sentinel/collections" }
      ]
    }
  ],
  "links": [
    {
      "rel": "self",
      "href": "https://api.example.com/catalogs"
    },
    {
      "rel": "root",
      "href": "https://api.example.com/"
    }
  ]
}
```

### 2. Catalog Deletion (`DELETE /catalogs/{id}`)
* **Default Behavior:** Deletes the Catalog object.
* **Cascade Parameter:** Implementations SHOULD support a `?cascade=true` query parameter.
    * If `cascade=true`: The API deletes the Catalog **AND** all Collections linked as children of that Catalog.
    * If `cascade=false` (default): The API deletes the Catalog, but orphaned Collections may remain in the database (implementation dependent).

### 3. Scoped Collection Creation (`POST /catalogs/{id}/collections`)
* **Body:** Accepts a standard [STAC Collection](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md) JSON object.
* **Behavior:**
    1.  Creates the Collection resource.
    2.  **Automatic Linking:** Automatically adds a `rel="parent"` (or `rel="catalog"`) link in the Collection pointing to `{catalogId}`.
    3.  **Reverse Linking:** Automatically adds a `rel="child"` link in the Catalog pointing to the new Collection.

### 4. Scoped Collection Deletion (`DELETE .../collections/{id}`)
* **Behavior:** Deletes the Collection resource and removes the corresponding `child` link from the parent Catalog to ensure referential integrity.

## Link Relations

Proper linking is critical for clients to navigate the federation structure.

### 1. The Global Root (`/`)
This is the entry point.
- `rel="catalogs"`: MUST point to the `/catalogs` endpoint (the registry).
- `rel="service-desc"`: Points to the OpenAPI definition.

*Note: We use `rel="catalogs"` instead of `rel="data"` to avoid confusing standard clients that expect `data` to point to a Collections list.*

### 2. The Sub-Catalog (`/catalogs/{catalogId}`)
This resource acts as the **Landing Page** for the provider.
- `rel="self"`: MUST point to `/catalogs/{catalogId}`.
- `rel="parent"`: MUST point to the Global Root (`/`).
- `rel="root"`: SHOULD point to the Global Root (`/`) to maintain a single navigation tree.
- `rel="data"`: MUST point to `/catalogs/{catalogId}/collections`.

## Response Examples

### 1. The Registry List (`GET /catalogs`)

This endpoint returns a JSON object structurally similar to a standard `/collections` response, but it contains a list of **Catalog** objects in a `catalogs` array.

```json
{
  "catalogs": [
    {
      "id": "usgs-landsat",
      "type": "Catalog",
      "title": "USGS Landsat",
      "description": "Landsat collections provided by USGS.",
      "stac_version": "1.0.0",
      "links": [
        { "rel": "self", "href": "https://api.example.com/catalogs/usgs-landsat" },
        { "rel": "root", "href": "https://api.example.com/" },
        { "rel": "child", "href": "https://api.example.com/catalogs/usgs-landsat/collections" }
      ]
    },
    {
      "id": "esa-sentinel",
      "type": "Catalog",
      "title": "ESA Sentinel",
      "description": "Sentinel collections provided by ESA.",
      "stac_version": "1.0.0",
      "links": [
        { "rel": "self", "href": "https://api.example.com/catalogs/esa-sentinel" },
        { "rel": "root", "href": "https://api.example.com/" },
        { "rel": "child", "href": "https://api.example.com/catalogs/esa-sentinel/collections" }
      ]
    }
  ],
  "links": [
    {
      "rel": "self",
      "href": "https://api.example.com/catalogs"
    },
    {
      "rel": "root",
      "href": "https://api.example.com/"
    }
  ]
}
```

### 2. The Global Root (`GET /`)

The global root remains a standard STAC Landing Page. Note the addition of the `rel="catalogs"` link.

```json
{
  "stac_version": "1.0.0",
  "type": "Catalog",
  "id": "stac-api",
  "title": "Standard STAC API with Federation",
  "description": "A standard STAC API that also supports federated catalogs.",
  "conformsTo": [
    "https://api.stacspec.org/v1.0.0/core",
    "https://api.stacspec.org/v1.0.0-beta.1/catalogs-endpoint"
  ],
  "links": [
    {
      "rel": "self",
      "type": "application/json",
      "href": "https://api.example.com/"
    },
    {
      "rel": "service-desc",
      "type": "application/vnd.oai.openapi+json;version=3.0",
      "href": "https://api.example.com/api"
    },
    {
      "rel": "data",
      "type": "application/json",
      "href": "https://api.example.com/collections",
      "title": "Global Collections List"
    },
    {
      "rel": "catalogs",
      "type": "application/json",
      "href": "https://api.example.com/catalogs",
      "title": "Federated Catalogs Registry"
    }
  ]
}
```

## Optional Capabilities

### Filter Extension & Queryables

This extension reserves the path `/catalogs/{catalogId}/queryables` to support the **STAC Filter Extension** within a sub-catalog context.

However, implementation of this endpoint is **OPTIONAL**.

* A sub-catalog **MUST** only expose this endpoint if it advertises conformance to the Filter Extension URI (e.g., `https://api.stacspec.org/v1.0.0-rc.2/filter`) in the Sub-Catalog Landing Page (`/catalogs/{catalogId}`).
* If implemented, the queryables response must be scoped specifically to that sub-catalog.
