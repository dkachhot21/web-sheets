# Web Sheets: The Architectural Blueprint

## Phase 1: Architectural Foundation & Tech Stack
​To ensure the library remains lightweight, highly performant, and tree-shakeable, the core stack focuses on strict typing, atomic state, and native-feeling storage.

- **​Core Framework:** React (using TypeScript to strictly type cell data, user preferences, and configuration objects).

- **​State Management (Headless State):** `Zustand` or `Jotai`. Atomic state is critical for Excel-like grids; if a single cell changes, the architecture must ensure only that specific cell data updates in the state without triggering a re-render of the entire grid.

- **​User Preferences Management:** `localforage` integrated with Zustand's persist middleware. This handles user configurations (theme preferences, column widths, active modes) natively and persistently via asynchronous IndexedDB/localStorage without blocking the main JavaScript thread.

- **​Build & Bundling:** `Rollup` or `tsup` to compile clean, tree-shakeable ES modules, ensuring developers only ship the features they actively import.


## ​Phase 2: The Hybrid Rendering Engine
The core of the library relies on separating the data state from the visual representation. This allows the UI to seamlessly swap rendering engines based on the active mode while maintaining a single source of truth.

- **​The DOM Engine (View Mode):** Uses standard React DOM elements optimized with `@tanstack/react-virtual` (or adapting techniques from highly optimized grids like `react-data-grid`). Rendering massive DOM tables natively will immediately freeze the browser. By leveraging DOM virtualization, only the cells currently visible within the viewport's bounding box are rendered into the DOM. This makes the View mode incredibly snappy, even with thousands/millions of rows and accessible for screen readers, and easy to customize with standard CSS.

- **​The Canvas Engine (Edit Mode):** Bypasses the HTML DOM entirely. React mounts a single `<canvas>` element. An internal JavaScript drawing loop takes the Zustand state and paints the grid, text, and selection layers directly onto the canvas context. It scales precisely using `window.devicePixelRatio` for retina-display crispness and achieves 60 FPS performance on massive datasets by avoiding DOM reflows.

​
## Phase 3: The Unified Style State
​Visual metadata is treated as first-class data within the headless state manager to achieve seamless, bi-directional styling synchronization between the View and Edit modes.

- **Single Source of Truth:** Cell state is an object containing both the `value` and a `style` property (e.g., `{ value: "Total", style: { bold: true, bgColor: "#f4f4f4", align: "right" } }`).

- **​Canvas Styling Interpretation:** The Canvas drawing loop (Edit mode) reads this state and maps it to native Canvas API calls. Before calling `ctx.fillText()`, the engine dynamically sets `ctx.font` and `ctx.fillStyle` based on the cell's style object.

- **​Bi-directional Sync:** Because both the Virtualized DOM Engine and the Canvas Engine subscribe to the exact same Zustand store, changing a cell's background color in Edit mode instantly updates the centralized state. This forces the View mode to reflect that exact same styling the moment a user toggles modes, requiring zero manual data syncing.


## ​Phase 4: Interaction Management & The DOM Bridge
​Standard DOM events do not apply to individual cells inside an HTML5 Canvas. A specialized interaction layer bridges the gap between raw pixel drawing and user input.

- **​Central Event Delegator:** A single listener attached to the `<canvas>` captures mouse and keyboard events, calculates the exact row/column intersection based on the X/Y coordinates, and dispatches the corresponding action to the Zustand store.

- **​Layered Canvas Drawing:** The rendering loop is split into visual layers to maximize efficiency:

    - ​*Layer 1 (Base):* Grid lines and background colors.
    - ​*Layer 2 (Data):* Text rendering and cell values.
    - *​Layer 3 (Interactive):* Highlight borders, multi-select overlays, and cursors. Only this top layer redraws during rapid interactions like drag-and-drop.

- **The Input Overlay (DOM Bridge):** Canvas cannot natively handle blinking text cursors or text highlighting. When a user double-clicks a cell in Edit mode, the library instantly positions a transparent, absolutely-positioned HTML `<input>` or `<textarea>` directly over those exact Canvas coordinates. Upon pressing Enter, the DOM element unmounts, the Zustand state updates, and the Canvas repaints the new value.


## ​Phase 5: Core Module & Plugin Ecosystem
​By isolating features into optional packages, developers can build everything from a lightweight read-only data table to a fully-fledged enterprise Excel clone.

- **​Core & Interaction Plugins:**
    - `​@web-sheets/core`: The barebones state manager and rendering engines (View/Edit switch) with all performance optimizations like pagination and browser caching.
    - `​@web-sheets/plugin-selection`: Enables multi-cell selection, range highlighting, and active cell tracking.
    - `​@web-sheets/plugin-clipboard`: Parses TSV/CSV formats natively to manage Excel-like copy/paste operations across the browser clipboard.
    - `​@web-sheets/plugin-drag-drop`: Handles the heavy coordinate math and event listeners required for resizing columns, moving rows, and dragging cell data.

- **​Advanced Feature Plugins:**
    - `​@web-sheets/plugin-formulas`: Parses and evaluates Excel-like formulas (e.g., `=SUM(A1:A5)`) using a dependency graph via an internal parser like `hot-formula-parser`.
    - `​@web-sheets/plugin-collaboration`: Hooks into the state manager to support real-time, multiplayer editing. Developers can connect it to their own WebSocket backend (e.g., `Socket.io`) to broadcast state patches using CRDTs (like Yjs) to prevent data collisions.
    - `​@web-sheets/plugin-history`: Adds Undo/Redo functionality by acting as middleware for the Zustand store, taking lightweight state snapshots using `immer`.
    - `​@web-sheets/plugin-validation`: Ensures data integrity by restricting inputs. Invalid inputs trigger an error tooltip via the DOM Bridge.
    - `​@web-sheets/plugin-export`: Converts the proprietary state into `.xlsx` or `.csv` files entirely on the client side using libraries like `exceljs`.
    - `​@web-sheets/plugin-filters`: Modifies the rendering loops to hide rows that do not match active filter criteria, sorting the row indices logically without permanently altering the underlying raw data array.


## ​Phase 6: Data Architecture & Inversion of Control
​To handle millions of rows without crashing the browser or exhausting network bandwidth, `web-sheets` operates as a strict "Controlled Component." The library acts as a dumb, highly-optimized UI engine that handles zero networking itself.

- **​Agnostic Data Consumption:** `web-sheets` does not call APIs, nor does it know about databases or Redis. It simply accepts a JSON object passed to it via props.

- **​Event-Driven Pagination:** As the user scrolls through the Canvas or Virtualized DOM, the library emits an `onViewportChange(startIndex, endIndex)` event.

- **​Developer Responsibility:** The developer listens for this event, executes their own network logic (checking their backend API, which in turn checks Redis or PostgreSQL), and updates the React state. The updated state trickles back down into `web-sheets`, which instantly paints the new rows.

- **​Client-Side Caching (Optional Plugin):** Developers can use `@web-sheets/plugin-caching` to automatically store chunks of the provided JSON into the browser's IndexedDB, preventing redundant API calls when the user scrolls back up.


## ​Phase 7: API Design (Developer Experience)
​The resulting API cleanly separates the state, styling, and plugins. It provides maximum flexibility, allowing developers to wire the grid into any stack (REST, GraphQL, WebSockets) effortlessly.

``` ts
import { useState } from 'react';
import { WebSheet } from '@web-sheets/core';
import { 
  SelectionPlugin, 
  DragDropPlugin,
  FormulaPlugin, 
  HistoryPlugin, 
  CollaborationPlugin,
  CachingPlugin
} from '@web-sheets/plugins';

export function DeveloperApp() {
  // 1. The developer manages the actual data state in their app
  const [sheetData, setSheetData] = useState({}); 

  // 2. The developer writes their own custom fetching logic (hitting their Node/Redis backend)
  const fetchMissingRows = async (start, end) => {
    const response = await myCustomApiClient.get(`/api/rows?start=${start}&end=${end}`);
    setSheetData(prev => ({ ...prev, ...response.data }));
  };

  return (
    <WebSheet 
      // --- DATA & STATE (Inversion of Control) ---
      data={sheetData} 
      totalRowCount={1000000}
      
      // The library shouts that the user scrolled. The developer handles the fetching.
      onViewportChange={(visibleStart, visibleEnd) => {
        fetchMissingRows(visibleStart, visibleEnd);
      }}

      // The library shouts that a cell was edited. The developer saves it to their DB.
      onCellEdit={(rowIndex, colIndex, newValue) => {
        myCustomApiClient.post('/api/update', { rowIndex, colIndex, newValue });
      }}

      // --- RENDERING & STYLING ---
      mode="edit" 
      allowEditModeStyling={true}
      theme={{ 
        primaryColor: '#0052CC', 
        fontFamily: 'Inter',
        cellHeight: 32 
      }}

      // --- MODULAR PLUGINS ---
      plugins={[
        SelectionPlugin,
        DragDropPlugin,
        FormulaPlugin,
        HistoryPlugin,
        CachingPlugin.configure({ strategy: 'indexedDB' }), 
        CollaborationPlugin 
      ]}
    />
  );
}
```