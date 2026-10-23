# Module 4 - Vector Styling & Feature Interaction

Welcome to Module 4! In this module, we transition from merely loading raw vector coordinates to crafting expressive, responsive, and interactive Web GIS interfaces. We will follow the complete production engineering workflow:

```text
Feature Data & Geometry
          ↓
Dynamic Style Function (with Caching & Decluttering)
          ↓
Symbology, Halo Labels & Custom Icons
          ↓
Canvas Pointer Detection (Hover Highlight & Cursor State)
          ↓
Selection Interaction & Coordinate-Anchored Popups
```

---

## 1. Introduction to Vector Styling

### 1.1 Why Vector Styling is Different from WMS Styling
In Module 3, we explored server-side WMS styling via GeoServer SLD, where the server renders features into static pixels before sending them across the network. Vector styling in OpenLayers is fundamentally different:

| Feature | Server-Side WMS Styling (GeoServer / SLD) | Client-Side Vector Styling (OpenLayers) |
|---|---|---|
| **Rendering Engine** | Server CPU / GPU (GeoServer) | Browser HTML5 Canvas / WebGL |
| **Output Type** | Pre-rendered raster image (PNG, JPEG) | Live DOM / Canvas vector paths |
| **Interactivity** | Requires server roundtrips (`GetFeatureInfo`) | Instant client-side inspection, selection & hover |
| **Styling Flexibility** | Static XML rules on the server | Full JavaScript logic (conditional, dynamic, animated) |
| **Styling Speed** | Re-requests images on style changes | Updates instantly in browser memory |
| **Data Payload** | Map image pixels | Raw coordinates and attributes (GeoJSON/WFS) |

### 1.2 Client-Side Styling in OpenLayers

In OpenLayers, vector layers use the `setStyle()` method to apply visual symbology. Styling can be assigned at two distinct scopes:

- **`VectorLayer`**: `vectorLayer.setStyle(...)` sets the default appearance for all features on the layer.
- **Individual `Feature`**: `feature.setStyle(...)` overrides the layer style for a specific feature instance (ideal for selection flags, hover highlights, or custom status indicators).

### 1.3 Static vs. Dynamic Styling

- **Static Styling:** A fixed `Style` object applied uniformly to every feature across all zoom levels.
- **Dynamic Styling:** A JavaScript **Style Function** that inspects each feature's attributes (`feature.get('type')`) or the map's current zoom resolution to return tailored styles dynamically.

### 1.4 Geometry Targets

Styles apply across the three core vector geometry categories:

- **Point / MultiPoint:** Styled with circular vector symbols (`Circle`) or external bitmap/SVG graphics (`Icon`).
- **LineString / MultiLineString:** Styled with strokes (`Stroke`).
- **Polygon / MultiPolygon:** Styled with interior fills (`Fill`) and outer boundary strokes (`Stroke`).

---

## 2. OpenLayers Style Components

OpenLayers builds symbology using specialized building blocks imported from `ol/style.js`:

```text
Point Feature
 ├── image: Circle / Icon
 └── text:  Text (Label)

LineString Feature
 ├── stroke: Stroke (Color, Width, Dash)
 └── text:   Text (Label)

Polygon Feature
 ├── fill:   Fill (Color, Opacity)
 ├── stroke: Stroke (Boundary)
 └── text:   Text (Centroid Label)
```

### 2.1 Core Style Classes

- **`Style`:** The root container that groups together fill, stroke, image, and text symbolizers.
- **`Fill`:** Defines interior colors and transparencies for polygons, circles, and text backgrounds. Supports HEX, RGB, RGBA, and HSL strings.
- **`Stroke`:** Defines boundary lines, road paths, and outlines with width, color, and dash patterns.
- **`Circle` (`CircleStyle`):** Vector circular marker designed specifically for Point geometries.
- **`Icon`:** External raster (PNG/JPEG) or vector (SVG) image symbol for Point geometries.
- **`Text`:** Renders dynamic typographic labels anchored to feature geometries.

### 2.2 Core Style Properties Reference

| Component | Property | Description | Example |
|---|---|---|---|
| **Fill** | Fill color | HEX, RGB, RGBA, or HSL color string | `color: 'rgba(33, 150, 243, 0.5)'` |
| **Stroke** | Stroke color | Outline line color | `color: '#1565C0'` |
| | Stroke width | Line thickness in pixels | `width: 2.5` |
| | Line dash | Dash-gap array pattern | `lineDash: [6, 6]` (6px dash, 6px space) |
| **Circle** | Point radius | Radius of circle marker in pixels | `radius: 8` |
| **Icon** | Icon image | URL or path to image file | `src: '/icons/hospital.svg'` |
| | Scale | Scaling factor | `scale: 0.8` |
| | Anchor | Alignment point coordinates | `anchor: [0.5, 1]` (Bottom-center pin) |
| | Rotation | Rotation angle in radians | `rotation: Math.PI / 4` |
| **Text** | Text font | Font style, size, and family | `font: 'bold 13px Inter, sans-serif'` |
| | Text color | Color of the text characters | `fill: new Fill({ color: '#212121' })` |
| | Text offset | Pixel displacement (X, Y) | `offsetY: -15` (Places text above marker) |

---

## 3. Static Styling

When all features in a layer share identical cartography, apply a single reusable `Style` object to the `VectorLayer`.

### 3.1 Applying Styles to Points, Lines, and Polygons

```javascript
import VectorLayer from 'ol/layer/Vector.js';
import VectorSource from 'ol/source/Vector.js';
import { Style, Fill, Stroke, Circle as CircleStyle } from 'ol/style.js';

// 1. Polygon Style (Fills and Outlines)
const parcelStyle = new Style({
  fill: new Fill({
    color: 'rgba(76, 175, 80, 0.3)'
  }),
  stroke: new Stroke({
    color: '#2E7D32',
    width: 2
  })
});

// 2. LineString Style (Roads / Rivers)
const roadStyle = new Style({
  stroke: new Stroke({
    color: '#FF9800',
    width: 3,
    lineDash: [4, 4]
  })
});

// 3. Point Style (Circular Marker)
const pointStyle = new Style({
  image: new CircleStyle({
    radius: 7,
    fill: new Fill({ color: '#E91E63' }),
    stroke: new Stroke({ color: '#FFFFFF', width: 2 })
  })
});

// Apply style to VectorLayer
const vectorLayer = new VectorLayer({
  source: new VectorSource({ /* ... */ }),
  style: parcelStyle
});
```

### 3.2 The Importance of Reusable Style Objects
Always define style objects outside render loops:

```javascript
vectorLayer.setStyle(myStyle);
```

> **The Mechanics of Reusable Styles:** OpenLayers executes the styling pipeline every time a frame is rendered (during pan, zoom, or animations at 60 FPS). Reusing pre-instantiated style objects prevents allocating thousands of temporary JavaScript objects, avoiding browser garbage-collection pauses and stuttering animations.

---

## 4. Attribute-Based Dynamic Styling

Real-world applications require thematic styling where feature appearance reflects underlying attributes.

### 4.1 Reading Feature Attributes
Feature attributes are accessed using `.get()`:
```javascript
const category = feature.get('type');
const population = feature.get('population');
```

### 4.2 Dynamic Style Function
Pass a function to `vectorLayer.setStyle()`. OpenLayers evaluates the function for each feature:

```javascript
const styleFunction = (feature) => {
  const type = feature.get('type');

  if (type === 'hospital') {
    return hospitalStyle;
  }

  if (type === 'school') {
    return schoolStyle;
  }

  return defaultStyle;
};

vectorLayer.setStyle(styleFunction);
```

### 4.3 Categorized Thematic Styling
Pre-define reusable styles in a lookup table for maximum efficiency:

```javascript
const facilityStyles = {
  hospital: new Style({
    image: new CircleStyle({
      radius: 8,
      fill: new Fill({ color: '#D32F2F' }), // Red
      stroke: new Stroke({ color: '#FFFFFF', width: 2 })
    })
  }),
  school: new Style({
    image: new CircleStyle({
      radius: 8,
      fill: new Fill({ color: '#1976D2' }), // Blue
      stroke: new Stroke({ color: '#FFFFFF', width: 2 })
    })
  }),
  default: new Style({
    image: new CircleStyle({
      radius: 6,
      fill: new Fill({ color: '#757575' }), // Gray
      stroke: new Stroke({ color: '#FFFFFF', width: 1.5 })
    })
  })
};

vectorLayer.setStyle((feature) => {
  const category = feature.get('type');
  return facilityStyles[category] || facilityStyles.default;
});
```

### 4.4 Basic Graduated Styling (Numeric Ranges)
Adjust visual properties (color, size) based on numeric attributes:

```javascript
function earthquakeStyle(feature) {
  const mag = feature.get('magnitude');

  let color = '#4CAF50'; // Low < 4.0
  let radius = 5;

  if (mag >= 6.0) {
    color = '#D32F2F'; // Severe >= 6.0
    radius = 12;
  } else if (mag >= 4.0) {
    color = '#FB8C00'; // Moderate 4.0 - 5.9
    radius = 8;
  }

  return new Style({
    image: new CircleStyle({
      radius: radius,
      fill: new Fill({ color: color }),
      stroke: new Stroke({ color: '#FFFFFF', width: 1.5 })
    })
  });
}
```

---

## 5. Text Labels

Labels display contextual information directly on the map.

```javascript
new Text({
  text: feature.get('name'),
  font: '14px sans-serif',
  offsetY: -15
});
```

### 5.1 Label Configuration

- **`text`:** The string to display, typically retrieved from a feature attribute (`feature.get('name')`).
- **`font`:** Standard CSS font string (e.g., `'bold 12px Inter, sans-serif'`).
- **`fill`:** Font character color (`new Fill({ color: '#212121' })`).
- **`stroke`:** Outer halo around letters (`new Stroke({ color: '#FFFFFF', width: 3 })`). Critical for legibility over satellite imagery and street maps.
- **`backgroundFill` / `backgroundStroke`:** Optional background pill box behind the text.
- **`offsetY` / `offsetX`:** Pixel displacement relative to the feature geometry center.

### 5.2 Scale & Visibility Control
Displaying thousands of labels when zoomed out causes visual clutter and slows rendering. Use the `resolution` argument in your style function to display labels only when zoomed in close:

```javascript
vectorLayer.setStyle((feature, resolution) => {
  // Only show text labels when resolution is less than 30 meters/pixel (zoomed in)
  const showLabel = resolution < 30;

  return new Style({
    image: pointMarker,
    text: showLabel ? new Text({
      text: feature.get('name') || '',
      font: 'bold 12px sans-serif',
      fill: new Fill({ color: '#111111' }),
      stroke: new Stroke({ color: '#FFFFFF', width: 3 }),
      offsetY: -15
    }) : undefined
  });
});
```

### 5.3 Automated Label Decluttering
OpenLayers features built-in collision detection to automatically hide overlapping labels and icons. Enable decluttering directly on the `VectorLayer`:

```javascript
const declutteredLayer = new VectorLayer({
  source: vectorSource,
  style: featureStyleFunction,
  declutter: true // Automatically hides colliding text labels and icons
});
```

---

## 6. Icons & Custom Symbols

Vector points can be styled with custom bitmap (PNG) or vector (SVG) images using `ol/style/Icon`.

```text
Common Web GIS Use Cases:
 Hospital → hospital icon
 School   → school icon
 Airport  → airport icon
```

### 6.1 Creating an Icon Style
```javascript
import { Style, Icon } from 'ol/style.js';

const hospitalIconStyle = new Style({
  image: new Icon({
    src: '/assets/icons/hospital.svg',
    anchor: [0.5, 1],          // Bottom-center pin tip aligns with coordinate
    anchorXUnits: 'fraction',
    anchorYUnits: 'fraction',
    scale: 0.8,                // Resizes icon
    rotation: 0                // Rotation in radians
  })
});
```

### 6.2 Anchor Positioning

The `anchor` property dictates which point of the icon image aligns with the feature's geographic coordinate:

- **`[0.5, 0.5]` (Center):** Default. Best for circular badges, POI symbols, and airport markers.
- **`[0.5, 1.0]` (Bottom-Center):** Standard for map pin markers so the needle tip points to the location.

```text
       ┌──────────┐
       │  HOSPITAL│
       └────┬─────┘
            ▼  ← Anchor [0.5, 1.0] aligns with geographic coordinate
```

---

## 7. Feature Selection

OpenLayers provides the **`Select` interaction** (`ol/interaction/Select`) to capture user clicks, manage selection state, and highlight active features.

```javascript
import Select from 'ol/interaction/Select.js';
import { click } from 'ol/events/condition.js';
import { Style, Fill, Stroke } from 'ol/style.js';

// 1. Configure the selection interaction
const select = new Select({
  condition: click, // Trigger on mouse click
  style: new Style({
    fill: new Fill({ color: 'rgba(255, 235, 59, 0.5)' }),
    stroke: new Stroke({ color: '#F57F17', width: 3.5 })
  })
});

map.addInteraction(select);

// 2. Listen to the select event
select.on('select', (event) => {
  const feature = event.selected[0];

  if (feature) {
    console.log('Selected feature properties:', feature.getProperties());
  }
});
```

### 7.1 Selection Features

- **Selection Condition:** `condition: click` triggers on single clicks; `pointerMove` triggers on hover.
- **Deselection:** Clicking anywhere on the empty map canvas automatically deselects features.
- **Selected Array:** `event.selected` contains features just selected; `event.deselected` contains features whose selection was cleared.

---

## 8. Hover Interaction

Hover effects provide instant visual feedback as the user navigates across the map canvas.

```text
Mouse moves (pointermove)
     ↓
Feature detected?
     ↓
    Yes
     ↓
Apply highlight style & change cursor to pointer
```

### 8.1 Changing the Mouse Cursor
Inform the user when an element is interactive by updating CSS cursor styling on `pointermove`:

```javascript
map.on('pointermove', (evt) => {
  if (evt.dragging) return;

  const hit = map.hasFeatureAtPixel(evt.pixel);
  map.getTargetElement().style.cursor = hit ? 'pointer' : '';
});
```

### 8.2 Highlighting Features on Hover
```javascript
let currentHoveredFeature = null;

const hoverHighlightStyle = new Style({
  stroke: new Stroke({
    color: '#00E5FF',
    width: 4
  })
});

map.on('pointermove', (evt) => {
  if (evt.dragging) return;

  // 1. Reset previous hovered feature
  if (currentHoveredFeature) {
    currentHoveredFeature.setStyle(null); // Reverts back to layer style
    currentHoveredFeature = null;
  }

  // 2. Highlight new feature under cursor
  map.forEachFeatureAtPixel(evt.pixel, (feature, layer) => {
    if (layer === vectorLayer) {
      currentHoveredFeature = feature;
      feature.setStyle(hoverHighlightStyle); // Apply feature-level highlight
      return true; // Stop iterating
    }
  });
});
```

---

## 9. Feature Information / Popup

Popups display detailed attributes in an HTML card anchored to the feature's coordinate using `ol/Overlay`.

```text
User clicks feature
        ↓
Feature selected
        ↓
Read properties (feature.getProperties())
        ↓
Create popup content
        ↓
Display information at coordinate
```

### 9.1 Popup Setup (`main.js`)
```javascript
import Overlay from 'ol/Overlay.js';

// 1. HTML DOM elements
const container = document.getElementById('popup');
const content = document.getElementById('popup-content');
const closer = document.getElementById('popup-closer');

// 2. Instantiate Overlay
const popupOverlay = new Overlay({
  element: container,
  autoPan: { animation: { duration: 250 } }
});
map.addOverlay(popupOverlay);

// 3. Close button
closer.onclick = () => {
  popupOverlay.setPosition(undefined);
  closer.blur();
  return false;
};

// 4. Click interaction to display popup
map.on('singleclick', (evt) => {
  const feature = map.forEachFeatureAtPixel(evt.pixel, (feat) => feat);

  if (feature) {
    const props = feature.getProperties();
    content.innerHTML = `
      <h4 style="margin: 0 0 6px 0;">${props.name || 'Feature'}</h4>
      <p style="margin: 0; font-size: 13px; color: #555;"><strong>Type:</strong> ${props.type || 'N/A'}</p>
    `;
    popupOverlay.setPosition(evt.coordinate);
  } else {
    popupOverlay.setPosition(undefined);
  }
});
```

---

## 10. Layer-Level vs. Feature-Level Styling

Understanding how style inheritance operates ensures optimal performance and modular code:

```javascript
// Layer-level styling
vectorLayer.setStyle(...)
```

versus:

```javascript
// Feature-level styling
feature.setStyle(...)
```

| Scope | Invocation | Characteristics | Primary Use Cases |
|---|---|---|---|
| **Layer-Level** | `vectorLayer.setStyle(...)` | • Applied globally across all features<br>• Highest rendering efficiency<br>• Evaluated dynamically with style functions | Default layer symbology, thematic classification, zoom-dependent scales |
| **Feature-Level** | `feature.setStyle(...)` | • Overrides the layer style for a specific feature instance<br>• Reverts when set to `null`: `feature.setStyle(null)` | Temporary hover highlights, active selection flags, custom pin icons |

---

## 11. Styling Performance

When rendering vector layers in production, keep these performance principles in mind:

- **Reuse `Style` Objects:** Maintain a cached object dictionary (`styles[category]`) rather than constructing `new Style()` instances inside style functions.
- **Avoid Creating Complex Styles Unnecessarily:** Every extra stroke, text shadow, and nested fill increases canvas draw call time.
- **Don't Use Huge Numbers of Unique Icons:** Loading 200 distinct icon URLs triggers 200 separate asynchronous image downloads. Prefer SVG icons or standardized icon sheets.
- **Avoid Excessive Labels:** Restrict label rendering to close zoom levels using the `resolution` argument or enable `declutter: true`.
- **Be Careful with Thousands of Client-Side Features:** If your dataset exceeds 5,000 features, pure GeoJSON/WFS rendering can degrade browser frame rates.
- **Use Server-Side / Vector Tiles for Very Large Datasets:** As covered in Module 2, switch to **Vector Tiles (MVT)** or server-side **WMS** for datasets with tens of thousands of geometries.

---

### Reference resources

- **OpenLayers Style API:** [https://openlayers.org/en/latest/apidoc/module-ol_style_Style-Style.html](https://openlayers.org/en/latest/apidoc/module-ol_style_Style-Style.html)
- **OpenLayers Select Interaction:** [https://openlayers.org/en/latest/apidoc/module-ol_interaction_Select-Select.html](https://openlayers.org/en/latest/apidoc/module-ol_interaction_Select-Select.html)
- **OpenLayers Overlay API:** [https://openlayers.org/en/latest/apidoc/module-ol_Overlay-Overlay.html](https://openlayers.org/en/latest/apidoc/module-ol_Overlay-Overlay.html)
- **OpenLayers Vector Labels Example:** [https://openlayers.org/en/latest/examples/vector-labels.html](https://openlayers.org/en/latest/examples/vector-labels.html)

---

### Practical activity

#### Your Task:

##### Step 1 — Dynamic Styling
1. Take the vector layer (GeoJSON or WFS) and apply a dynamic style function based on the `type` attribute:
   ```javascript
   vectorLayer.setStyle((feature) => {
     const type = feature.get('type');

     return type === 'hospital'
       ? hospitalStyle
       : defaultStyle;
   });
   ```

##### Step 2 — Labels
2. Extend the style function to display the feature's `name` attribute using `new Text()`:
   ```javascript
   new Text({
     text: feature.get('name'),
     font: 'bold 12px sans-serif',
     offsetY: -15
   });
   ```

##### Step 3 — Select Interaction
3. Add an OpenLayers `Select` interaction configured to trigger on click:
   ```javascript
   const select = new Select({
     condition: click
   });
   map.addInteraction(select);

   select.on('select', (event) => {
     const feature = event.selected[0];

     if (feature) {
       console.log('Selected Properties:', feature.getProperties());
     }
   });
   ```

##### Step 4 — Bonus (Hover Effect)
4. Add a hover interaction that changes the mouse cursor to `'pointer'` and applies a stroke highlight when the mouse moves over a feature.

---

#### Example Implementation (`main.js`):

```javascript
import Map from 'ol/Map.js';
import View from 'ol/View.js';
import TileLayer from 'ol/layer/Tile.js';
import VectorLayer from 'ol/layer/Vector.js';
import OSM from 'ol/source/OSM.js';
import VectorSource from 'ol/source/Vector.js';
import GeoJSON from 'ol/format/GeoJSON.js';
import Overlay from 'ol/Overlay.js';
import Select from 'ol/interaction/Select.js';
import { click } from 'ol/events/condition.js';
import { Style, Circle as CircleStyle, Fill, Stroke, Text } from 'ol/style.js';
import { fromLonLat } from 'ol/proj.js';
import 'ol/ol.css';

// 1. Base Map Layer
const baseLayer = new TileLayer({
  source: new OSM()
});

// 2. Pre-defined Reusable Styles (Natural Earth Populated Places)
const capitalStyle = new Style({
  image: new CircleStyle({
    radius: 9,
    fill: new Fill({ color: '#D97706' }), // Amber/Gold for National Capitals
    stroke: new Stroke({ color: '#FFFFFF', width: 2 })
  })
});

const stateCapitalStyle = new Style({
  image: new CircleStyle({
    radius: 7,
    fill: new Fill({ color: '#2563EB' }), // Blue for Regional / State Capitals
    stroke: new Stroke({ color: '#FFFFFF', width: 2 })
  })
});

const defaultCityStyle = new Style({
  image: new CircleStyle({
    radius: 5,
    fill: new Fill({ color: '#475569' }), // Slate gray for other populated places
    stroke: new Stroke({ color: '#FFFFFF', width: 1.5 })
  })
});

// 3. Dynamic Style Function with Attribute Labels
const styleFunction = (feature, resolution) => {
  const featureClass = feature.get('FEATURECLA') || feature.get('featurecla');
  const base = featureClass === 'Admin-0 capital'
    ? capitalStyle
    : (featureClass === 'Admin-1 capital' ? stateCapitalStyle : defaultCityStyle);

  // Show city labels when zoomed in (resolution < 5000 meters/pixel)
  if (resolution < 5000) {
    const cityName = feature.get('NAME') || feature.get('name') || '';
    return new Style({
      image: base.getImage(),
      text: new Text({
        text: cityName,
        font: 'bold 12px Inter, sans-serif',
        fill: new Fill({ color: '#0f172a' }),
        stroke: new Stroke({ color: '#ffffff', width: 3 }),
        offsetY: -14
      })
    });
  }

  return base;
};

// 4. Vector Dataset (Natural Earth Populated Places Sample)
// In production, load via URL: url: './data/ne_10m_populated_places.geojson'
const vectorSource = new VectorSource({
  features: new GeoJSON().readFeatures({
    type: 'FeatureCollection',
    features: [
      {
        type: 'Feature',
        properties: { NAME: 'Tokyo', ADM0NAME: 'Japan', FEATURECLA: 'Admin-0 capital', POP_MAX: 37400068 },
        geometry: { type: 'Point', coordinates: fromLonLat([139.6917, 35.6895]) }
      },
      {
        type: 'Feature',
        properties: { NAME: 'Delhi', ADM0NAME: 'India', FEATURECLA: 'Admin-0 capital', POP_MAX: 28514000 },
        geometry: { type: 'Point', coordinates: fromLonLat([77.2090, 28.6139]) }
      },
      {
        type: 'Feature',
        properties: { NAME: 'Mumbai', ADM0NAME: 'India', FEATURECLA: 'Admin-1 capital', POP_MAX: 19980000 },
        geometry: { type: 'Point', coordinates: fromLonLat([72.8777, 19.0760]) }
      },
      {
        type: 'Feature',
        properties: { NAME: 'London', ADM0NAME: 'United Kingdom', FEATURECLA: 'Admin-0 capital', POP_MAX: 9046485 },
        geometry: { type: 'Point', coordinates: fromLonLat([-0.1278, 51.5074]) }
      },
      {
        type: 'Feature',
        properties: { NAME: 'New York', ADM0NAME: 'United States', FEATURECLA: 'Populated place', POP_MAX: 8419000 },
        geometry: { type: 'Point', coordinates: fromLonLat([-74.0060, 40.7128]) }
      }
    ]
  })
});

const vectorLayer = new VectorLayer({
  source: vectorSource,
  style: styleFunction,
  declutter: true
});

// 5. Popup Overlay Setup
const popupContainer = document.getElementById('popup');
const popupContent = document.getElementById('popup-content');
const popupCloser = document.getElementById('popup-closer');

const overlay = new Overlay({
  element: popupContainer,
  autoPan: { animation: { duration: 250 } }
});

if (popupCloser) {
  popupCloser.onclick = () => {
    overlay.setPosition(undefined);
    popupCloser.blur();
    return false;
  };
}

// 6. Map Initialization
const map = new Map({
  target: 'map',
  layers: [baseLayer, vectorLayer],
  overlays: [overlay],
  view: new View({
    center: fromLonLat([0, 25]),
    zoom: 2
  })
});

// 7. Select Interaction
const select = new Select({
  condition: click,
  style: new Style({
    image: new CircleStyle({
      radius: 12,
      fill: new Fill({ color: '#E11D48' }), // Highlighted red on selection
      stroke: new Stroke({ color: '#FFFFFF', width: 3 })
    })
  })
});
map.addInteraction(select);

select.on('select', (event) => {
  const feature = event.selected[0];

  if (feature) {
    const props = feature.getProperties();
    const pop = props.POP_MAX ? Number(props.POP_MAX).toLocaleString() : 'N/A';
    popupContent.innerHTML = `
      <h4 style="margin:0 0 6px 0; font-size:14px;">${props.NAME}</h4>
      <p style="margin:0 0 4px 0; font-size:12px; color:#475569;"><strong>Country:</strong> ${props.ADM0NAME || 'N/A'}</p>
      <p style="margin:0 0 4px 0; font-size:12px; color:#475569;"><strong>Classification:</strong> ${props.FEATURECLA || 'N/A'}</p>
      <p style="margin:0; font-size:12px; color:#475569;"><strong>Population:</strong> ${pop}</p>
    `;
    overlay.setPosition(feature.getGeometry().getCoordinates());
  } else {
    overlay.setPosition(undefined);
  }
});

// 8. Hover Effect (Cursor & Stroke Highlight)
let hoveredFeature = null;
const highlightStyle = new Style({
  image: new CircleStyle({
    radius: 11,
    fill: new Fill({ color: '#00E5FF' }),
    stroke: new Stroke({ color: '#FFFFFF', width: 3 })
  })
});

map.on('pointermove', (evt) => {
  if (evt.dragging) return;

  if (hoveredFeature) {
    hoveredFeature.setStyle(null);
    hoveredFeature = null;
  }

  const hit = map.forEachFeatureAtPixel(evt.pixel, (feat, layer) => {
    if (layer === vectorLayer && !select.getFeatures().getArray().includes(feat)) {
      hoveredFeature = feat;
      feat.setStyle(highlightStyle);
      return true;
    }
  });

  map.getTargetElement().style.cursor = hit ? 'pointer' : '';
});
```