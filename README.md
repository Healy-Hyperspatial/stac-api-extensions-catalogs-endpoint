# STAC API - Catalogs Extension

- **Title:** Catalogs
- **Conformance Classes:**
  - https://api.stacspec.org/v1.0.0/core (required)
  - https://api.stacspec.org/v1.0.0-beta.1/catalogs (required)
- **Scope:** STAC API - Core
- **Extension Maturity Classification:** Proposal
- **Dependencies:**
  - [STAC API - Core](https://github.com/radiantearth/stac-api-spec/blob/main/core)
  - [STAC API - Collections](https://github.com/radiantearth/stac-api-spec/tree/main/ogcapi-features)
- **Owner**: @jonhealy1

## Introduction

This extension enables a **Federated STAC API** architecture. It transforms the API Root into a "Catalog of Catalogs" (Portal), allowing a single API to serve as a registry for multiple independent data providers.

In this model, the API has a fixed-depth "Hub and Spoke" structure:
1.  **Global Root (`/`)**: The entry point. Contains links to Sub-Catalogs.
2.  **The Registry (`/catalogs`)**: A machine-readable list of all available Sub-Catalogs.
3.  **Sub-Catalogs (`/catalogs/{id}`)**: The actual data providers. These behave as standard STAC API Landing Pages containing Collections.

## Endpoints

This extension introduces a new root path `/catalogs` and nests standard STAC API endpoints under specific catalog IDs.

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


## Link Relations

Proper linking is critical for clients to navigate the federation structure.

### 1. The Global Root (`/`)
This is the entry point.
- `rel="data"`: MUST point to the `/catalogs` endpoint (the registry).
- `rel="child"`: MUST point to specific sub-catalogs (e.g., `/catalogs/usgs`).
- `rel="service-desc"`: Points to the OpenAPI definition.

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
        { "rel": "self", "href": "[https://api.example.com/catalogs/usgs-landsat](https://api.example.com/catalogs/usgs-landsat)" },
        { "rel": "root", "href": "[https://api.example.com/](https://api.example.com/)" },
        { "rel": "child", "href": "[https://api.example.com/catalogs/usgs-landsat/collections](https://api.example.com/catalogs/usgs-landsat/collections)" }
      ]
    },
    {
      "id": "esa-sentinel",
      "type": "Catalog",
      "title": "ESA Sentinel",
      "description": "Sentinel collections provided by ESA.",
      "stac_version": "1.0.0",
      "links": [
        { "rel": "self", "href": "[https://api.example.com/catalogs/esa-sentinel](https://api.example.com/catalogs/esa-sentinel)" },
        { "rel": "root", "href": "[https://api.example.com/](https://api.example.com/)" },
        { "rel": "child", "href": "[https://api.example.com/catalogs/esa-sentinel/collections](https://api.example.com/catalogs/esa-sentinel/collections)" }
      ]
    }
  ],
  "links": [
    {
      "rel": "self",
      "href": "[https://api.example.com/catalogs](https://api.example.com/catalogs)"
    },
    {
      "rel": "root",
      "href": "[https://api.example.com/](https://api.example.com/)"
    }
  ]
}
```

### 2. The Global Root (`GET /`)

The global root acts as a portal. Note that `rel="data"` points to the catalogs endpoint, not collections.

```json
{
  "stac_version": "1.0.0",
  "type": "Catalog",
  "id": "stac-federation",
  "title": "Global Data Portal",
  "description": "Entry point for the Federated STAC API.",
  "conformsTo": [
    "[https://api.stacspec.org/v1.0.0/core](https://api.stacspec.org/v1.0.0/core)",
    "[https://api.stacspec.org/v1.0.0-beta.1/catalogs](https://api.stacspec.org/v1.0.0-beta.1/catalogs)"
  ],
  "links": [
    {
      "rel": "self",
      "type": "application/json",
      "href": "[https://api.example.com/](https://api.example.com/)"
    },
    {
      "rel": "data",
      "type": "application/json",
      "href": "[https://api.example.com/catalogs](https://api.example.com/catalogs)",
      "title": "List of available catalogs"
    },
    {
      "rel": "child",
      "type": "application/json",
      "href": "[https://api.example.com/catalogs/usgs-landsat](https://api.example.com/catalogs/usgs-landsat)",
      "title": "USGS Landsat"
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
