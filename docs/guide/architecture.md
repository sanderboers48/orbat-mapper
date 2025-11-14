# Architecture Overview

This document explains the technical architecture of ORBAT Mapper, including how the application is structured, where map files are stored, and how different components work together.

## Project Overview

ORBAT Mapper is a **client-side only** web application built with:

- **Vue 3** - Frontend framework with Composition API
- **TypeScript** - Type-safe development
- **Vite** - Build tool and development server
- **OpenLayers** - Map rendering and interaction library
- **Pinia** - State management
- **IndexedDB** - Client-side data storage
- **Tailwind CSS** - Styling framework

### No Backend Server

**Important**: ORBAT Mapper does not have any backend code. It is a completely static web application that runs entirely in the browser. All data processing, map rendering, and scenario management happens on the client side.

## Project Structure

```
orbat-mapper/
├── public/                  # Static assets served as-is
│   ├── config/
│   │   └── mapConfig.json  # Basemap layer configuration
│   ├── images/             # Image assets
│   └── scenarios/          # Example scenario files
├── src/                    # Source code
│   ├── components/         # Vue components
│   ├── composables/        # Vue composables (reusable logic)
│   ├── geo/               # Geographic/mapping utilities
│   ├── modules/           # Feature modules
│   │   └── scenarioeditor/ # Main scenario editor module
│   ├── scenariostore/     # Scenario state management
│   ├── stores/            # Pinia stores
│   ├── types/             # TypeScript type definitions
│   └── views/             # Top-level view components
├── docs/                  # Documentation (VitePress)
└── dist/                  # Production build output (generated)
```

## Map Configuration and Basemaps

### Where Map Configuration Lives

The basemap configuration is stored in:
- **Development**: `/public/config/mapConfig.json`
- **Production**: `/dist/config/mapConfig.json` (copied during build)

### mapConfig.json Structure

The configuration file contains a JSON array of basemap layer definitions:

```json
[
  {
    "title": "OSM",
    "name": "osm",
    "layerSourceType": "osm",
    "sourceOptions": {
      "crossOrigin": "anonymous"
    }
  },
  {
    "title": "ESRI World imagery",
    "name": "esriWorldImagery",
    "layerSourceType": "xyz",
    "sourceOptions": {
      "crossOrigin": "anonymous",
      "url": "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}"
    }
  }
]
```

### How Basemaps Are Plotted

The basemap rendering process follows these steps:

1. **Configuration Loading** (`src/geo/baseLayers.ts`)
   - The `createBaseLayers()` function fetches `/config/mapConfig.json`
   - If fetching fails, it falls back to a default OSM layer
   - Each layer configuration is validated and processed

2. **Layer Creation**
   - Based on `layerSourceType`, the appropriate OpenLayers source is created:
     - `"osm"` → `OSM` source
     - `"xyz"` → `XYZ` source (for XYZ tile services)
   - Coordinate extents are transformed from EPSG:4326 to the map projection (typically Web Mercator EPSG:3857)

3. **Layer Registration**
   - Each basemap layer is added to the OpenLayers map as a `TileLayer`
   - Only one basemap is visible at a time (user-selectable)
   - Layers have properties: `title`, `name`, and `layerType: "baselayer"`

4. **Integration with OpenLayers**
   - Basemap layers are added to the map before any feature or unit layers
   - The map view manages zoom levels, center position, and projection
   - Tiles are fetched on-demand as the user pans and zooms

**Key Files:**
- `src/geo/baseLayers.ts` - Basemap loading and creation
- `src/geo/layerConfigTypes.ts` - TypeScript types for layer configuration
- `public/config/mapConfig.json` - Default basemap configuration

## Feature Layers

Feature layers allow users to add custom vector data (points, lines, polygons, circles) to the map with styling and time-based animation.

### Data Model

Feature layers are organized hierarchically:

```
Scenario
└── ScenarioLayer[] (Feature layers)
    └── ScenarioFeature[] (Individual features)
        ├── geometry: GeoJSON geometry
        ├── style: SimpleStyleSpec (colors, strokes, fills)
        ├── meta: Name, description, type
        └── state[]: Time-based geometry states
```

**Key Type Definitions** (`src/types/scenarioGeoModels.ts`):

- **ScenarioLayer**: A container for related features
  - `id`: Unique identifier
  - `name`: Display name
  - `features`: Array of ScenarioFeature IDs
  - `isHidden`: Visibility toggle
  - `opacity`: Layer opacity (0-1)
  - `locked`: Prevents editing

- **ScenarioFeature**: A GeoJSON feature with extensions
  - `id`: Unique identifier
  - `geometry`: GeoJSON geometry (Point, LineString, Polygon, Circle, etc.)
  - `meta`: Metadata (name, description, type, visibility settings)
  - `style`: Visual styling (colors, stroke width, fill patterns)
  - `state[]`: Array of time-based states for animation
  - `_state`: Current computed state at the current timeline position

### How Feature Layers Are Saved

Feature layers are part of the scenario data model and are saved in multiple locations:

1. **In-Memory State** (`src/scenariostore/newScenarioStore.ts`)
   - The active scenario is held in a reactive Pinia store
   - State structure:
     ```typescript
     {
       layers: LayerId[],           // Ordered array of layer IDs
       layerMap: { [id]: ScenarioLayer },
       featureMap: { [id]: ScenarioFeature }
     }
     ```

2. **IndexedDB** (`src/scenariostore/localdb.ts`)
   - Scenarios are automatically saved to browser IndexedDB
   - Database name: `scenario-db`
   - Two object stores:
     - `scenario-blobs`: Complete scenario JSON
     - `scenario-metadata`: Quick-access metadata (name, dates, description)
   - Provides local persistence between sessions

3. **File Export**
   - Users can export scenarios as `.json` files
   - The entire scenario, including all feature layers, is serialized to JSON
   - This provides backup and sharing capabilities

### How Feature Layers Are Plotted

The rendering pipeline synchronizes the scenario state with OpenLayers:

1. **Initialization** (`src/modules/scenarioeditor/scenarioFeatureLayers.ts`)
   - `useScenarioFeatureLayers()` composable manages the OpenLayers feature layers
   - Creates a LayerGroup to hold all feature layers
   - Registers event handlers for scenario changes

2. **Layer Creation**
   - For each `ScenarioLayer`, a corresponding `VectorLayer` is created
   - Each `VectorLayer` has a `VectorSource` containing OpenLayers `Feature` objects
   - Features are converted from GeoJSON to OpenLayers geometry

3. **Styling** (`src/geo/featureStyles.ts`)
   - Each feature's `style` property defines its appearance
   - Follows SimpleStyle specification (similar to Mapbox/GitHub)
   - Supports:
     - Marker icons and colors
     - Stroke width and color
     - Fill color and patterns
     - Text labels with fonts and colors
   - Styles are cached for performance

4. **Time-Based Animation**
   - Features can have multiple `state` entries with timestamps
   - As the scenario timeline advances, the current state is computed:
     - The system finds all states with `t <= currentTime`
     - The most recent state is applied to the feature
   - Geometry can change over time (e.g., moving a point, extending a line)

5. **Event Synchronization**
   - Changes to the scenario state trigger events:
     - `addLayer` / `removeLayer` / `updateLayer`
     - `addFeature` / `deleteFeature` / `updateFeature`
     - `moveFeature` / `moveLayer`
   - Event handlers update the corresponding OpenLayers objects
   - This keeps the map view synchronized with the data model

6. **Z-Index Management**
   - Features within a layer have a `_zIndex` property
   - Controls drawing order (lower values drawn first)
   - Users can reorder features by dragging in the UI

**Key Files:**
- `src/modules/scenarioeditor/scenarioFeatureLayers.ts` - OpenLayers integration
- `src/modules/scenarioeditor/featureLayerUtils.ts` - Utilities for feature layers
- `src/scenariostore/geo.ts` - Feature layer state management
- `src/geo/featureStyles.ts` - Feature styling logic
- `src/types/scenarioGeoModels.ts` - Type definitions

## Map Overlay Layers

In addition to feature layers, ORBAT Mapper supports temporary map overlay layers:

### Supported Layer Types

1. **XYZ Tile Layers**: External tile services (configurable tile URLs)
2. **TileJSON Layers**: Tile layers defined by TileJSON specification
3. **Image Layers**: Static images positioned on the map (georeferenced)
4. **KML/KMZ Layers**: Keyhole Markup Language files (temporary, not saved with scenario)

### Implementation

Map overlay layers are managed separately from feature layers:

- **State**: Stored in `mapLayers` and `mapLayerMap` in the scenario store
- **Rendering**: Handled by `src/modules/scenarioeditor/scenarioMapLayers.ts`
- **Layer Group**: Added to a separate LayerGroup in OpenLayers
- **Persistence**: Saved with the scenario (except KML/KMZ and blob: URLs)

## Unit Layers

Units (military symbols) are rendered in their own specialized layers:

1. **Unit Layer** (`src/composables/geoUnitLayers.ts`)
   - Vector layer containing unit positions
   - Each unit is represented as a Point feature
   - Uses MilSymbol library for rendering military symbols

2. **Label Layer**
   - Separate layer for unit labels
   - Improves performance by separating symbol rendering from labels

3. **History Layer**
   - Shows unit movement paths over time
   - Rendered as LineString features
   - Waypoints shown as interactive points

4. **Range Rings Layer**
   - Displays range circles around units
   - Useful for showing weapon ranges or areas of effect

## State Management

ORBAT Mapper uses a centralized state management system:

### Scenario Store (`src/scenariostore/newScenarioStore.ts`)

The core state container using Pinia:

```typescript
{
  // Scenario metadata
  name: string
  description: string
  startTime: number
  currentTime: number
  
  // Units and ORBAT structure
  sides: SideId[]
  sideMap: { [id]: Side }
  sideGroupMap: { [id]: SideGroup }
  unitMap: { [id]: Unit }
  
  // Feature layers
  layers: LayerId[]
  layerMap: { [id]: ScenarioLayer }
  featureMap: { [id]: ScenarioFeature }
  
  // Map overlay layers
  mapLayers: LayerId[]
  mapLayerMap: { [id]: ScenarioMapLayer }
  
  // ... other state
}
```

### Key Store Features

- **Immutable Updates**: Uses Immer for immutable state updates
- **Undo/Redo**: Full undo/redo support with labeled actions
- **Event System**: Event hooks for reacting to state changes
- **Computed Properties**: Reactive computed values derived from state
- **Time Management**: Timeline controls for animating scenarios

### Other Stores

- `geoStore.ts` - Geographic operations (zoom, pan)
- `mapSettingsStore.ts` - Map display settings
- `mapSelectStore.ts` - Selection and interaction modes
- `uiStore.ts` - UI state (panels, toolbars)

## Data Flow

### Loading a Scenario

```
User Action (Open File/URL/IndexedDB)
    ↓
Load JSON → Parse and Validate
    ↓
Initialize Store State
    ↓
Initialize OpenLayers Map
    ↓
Load Basemaps (from mapConfig.json)
    ↓
Create Feature Layers (from scenario.layers)
    ↓
Create Unit Layers (from scenario.unitMap)
    ↓
Render Map
```

### Editing Features

```
User Draws/Edits on Map
    ↓
OpenLayers Interaction Events
    ↓
Composable Handlers (e.g., useDrawInteraction)
    ↓
Update Scenario Store (via geo.addFeature/updateFeature)
    ↓
Trigger Event (featureLayerEvent)
    ↓
Event Handler Updates OpenLayers Layer
    ↓
Map Re-renders
```

### Saving a Scenario

```
User Triggers Save
    ↓
Serialize Current Store State to JSON
    ↓
Save to IndexedDB (automatic) or File (manual)
```

## Technology Stack Details

### OpenLayers

OpenLayers is the core mapping library:

- **Map**: The main map component (`ol/Map`)
- **View**: Controls zoom, center, rotation, projection
- **Layers**: TileLayer, VectorLayer, LayerGroup
- **Sources**: OSM, XYZ, VectorSource, TileJSON
- **Features**: Geographic elements with geometry and properties
- **Interactions**: Select, Modify, Draw, DragAndDrop
- **Controls**: Zoom, ScaleLine, MousePosition

**Custom Extensions:**
- `ol-ext` library for advanced features (GeoImage layers)
- Custom interactions for military symbols

### Vue 3 Composables

The application heavily uses Vue 3 Composition API patterns:

- **Composables** (`src/composables/`): Reusable logic
  - `geoUnitLayers.ts` - Unit layer management
  - `geoMeasurement.ts` - Distance/area measurement
  - `openlayersHelpers.ts` - OpenLayers utilities
  - `geoImageLayerInteraction.ts` - Image layer transforms

### TypeScript Types

Strong typing throughout with key type categories:

- `scenarioModels.ts` - Core domain models (Unit, Side, etc.)
- `scenarioGeoModels.ts` - Geographic features and layers
- `internalModels.ts` - Internal runtime models with computed state
- `base.ts` - Base types and utilities

## Offline Usage

ORBAT Mapper can work offline with some configuration:

1. **Basemaps**: Configure local tile server URLs in `mapConfig.json`
2. **Scenarios**: Stored in IndexedDB (available offline)
3. **Application**: Can be served as static files from any web server
4. **Limitation**: Place name search requires online service

## Build and Deployment

### Development

```bash
npm install       # Install dependencies
npm run dev       # Start dev server (port 5173)
```

### Production

```bash
npm run build     # Build optimized static files to dist/
npm run preview   # Preview production build locally
```

### Deployment

The `dist/` folder contains a complete static website:
- Deploy to any static hosting (Netlify, Vercel, GitHub Pages, S3, etc.)
- No server-side processing required
- Configure basemaps in `dist/config/mapConfig.json` after build if needed

## Summary

ORBAT Mapper is a sophisticated client-side web application that:

- **Has no backend** - Everything runs in the browser
- **Stores map configs** in `/public/config/mapConfig.json` (basemaps) and scenario JSON (feature layers)
- **Plots basemaps** by fetching tile images from configured tile servers using OpenLayers
- **Saves feature layers** in IndexedDB and JSON files as part of the scenario data model
- **Plots feature layers** by converting GeoJSON to OpenLayers features and synchronizing with the scenario state
- **Uses OpenLayers** for all map rendering, interaction, and coordinate transformations
- **Manages state** with Pinia stores and Vue 3 reactivity
- **Supports time-based animation** by computing feature states at each timeline position

This architecture provides a powerful, fully client-side mapping application with no server dependencies.
