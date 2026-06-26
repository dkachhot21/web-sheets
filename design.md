# Web Sheets: The Architectural Blueprint

## Phase 1: Architectural Foundation & Tech Stack
​To ensure the library remains lightweight, highly performant, and tree-shakeable, the core stack focuses on strict typing, atomic state, and native-feeling storage.

- **​Core Framework:** React (using TypeScript to strictly type cell data, user preferences, and configuration objects).

- **​State Management (Headless State):** `Zustand` or `Jotai`. Atomic state is critical for Excel-like grids; if a single cell changes, the architecture must ensure only that specific cell data updates in the state without triggering a re-render of the entire grid.

- **​User Preferences Management:** `localforage` integrated with Zustand's persist middleware. This handles user configurations (theme preferences, column widths, active modes) natively and persistently via asynchronous IndexedDB/localStorage without blocking the main JavaScript thread.

- **​Build & Bundling:** `Rollup` or `tsup`. These tools excel at creating clean ES modules, ensuring the library remains completely tree-shakeable so end-users only ship the code they actually use in their final bundles.


## ​Phase 2: The Hybrid Rendering Engine
The core of the library relies on separating the data state from the visual representation. This allows the UI to seamlessly swap rendering engines based on the active mode while maintaining a single source of truth.

- **​The DOM Engine (View Mode):** Uses standard React DOM elements optimized with `@tanstack/react-virtual` (or adapting techniques from highly optimized grids like `react-data-grid`). Rendering massive DOM tables natively will immediately freeze the browser. By leveraging DOM virtualization, only the cells currently visible within the viewport's bounding box are rendered into the DOM, making the View mode incredibly snappy and accessible, even with thousands of rows.

- **​The Canvas Engine (Edit Mode):** Bypasses the HTML DOM entirely. React mounts a single `<canvas>` element. An internal JavaScript drawing loop takes the Zustand state and paints the grid, text, and selection layers directly onto the canvas context (conceptually similar to high-performance libraries like Glide Data Grid). It scales precisely using `window.devicePixelRatio` for retina-display crispness and achieves 60 FPS performance on massive datasets by avoiding DOM reflows entirely.


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


## ​Phase 5: Core Module Design (The Plugin System)
​To fulfill the requirement that users can choose their features to keep the library lightweight, `web-sheets` will be built on a modular plugin architecture.

- **Core & Interaction Plugins:**
    - `​@web-sheets/core`: The barebones grid engine. It handles only the core Zustand state manager, the View/Edit mode-switcher, and the base Canvas/DOM rendering loops.​
    - `@web-sheets/plugin-selection`: Enables multi-cell selection, range highlighting, and active cell tracking.
    - `​@web-sheets/plugin-clipboard`: Manages Excel-like copy/paste operations, parsing TSV/CSV formats natively so users can copy from Microsoft Excel or Google Sheets and paste directly into the web UI.
    - `​@web-sheets/plugin-drag-drop`: Handles the heavy coordinate math and event listeners required for resizing columns, moving rows, or dragging cell data.

- **Advance Feature Plugins:**
    - `​@web-sheets/plugin-formulas`: Parses and evaluates Excel-like formulas (e.g., =SUM(A1:A5)) using a dependency graph. If a cell updates, it automatically recalculates any dependent cells via an internal parser like hot-formula-parser.
    - `@web-sheets/plugin-collaboration`: Enables real-time, Google Sheets-style multiplayer editing. Leveraging Node.js and WebSockets via socket.io establishes a highly reliable real-time event pipeline, broadcasting Zustand state patches to active users while utilizing CRDTs (like Yjs) to prevent data collisions.
    - `​@web-sheets/plugin-history`: Adds Undo/Redo functionality by acting as middleware for the Zustand store. It takes lightweight state snapshots using immer and pushes them to a history stack.
    - `​@web-sheets/plugin-validation`: Ensures data integrity by restricting inputs (e.g., forcing a date format or dropdown list). Invalid inputs trigger an error tooltip via the DOM Bridge.
    - `​@web-sheets/plugin-export`: Converts the proprietary state into universally recognized .xlsx or .csv files entirely on the client side using libraries like exceljs.
    - `​@web-sheets/plugin-filters`: Modifies the View and Edit rendering loops to hide rows that do not match active filter criteria, sorting the row indices logically without permanently altering the underlying raw data array.

​
## Phase 6: API Design (Developer Experience)
The resulting API cleanly separates the data, the styling, and the active plugins into a highly declarative React component. Unused plugins will be ignored by bundlers like Webpack and Vite, keeping the final application size extremely small.

**​Conceptual Implementation:**

``` js
import { WebSheet } from '@web-sheets/core';
import { 
  SelectionPlugin, 
  DragDropPlugin,
  FormulaPlugin, 
  HistoryPlugin, 
  CollaborationPlugin 
} from '@web-sheets/plugins';
import { io } from 'socket.io-client';

// Establish the WebSocket connection for real-time multiplayer
const socket = io('http://localhost:3000');

export function App() {
  return (
    <WebSheet 
      data={sheetData}
      mode="edit" 
      allowEditModeStyling={true}
      plugins={[
        SelectionPlugin,
        DragDropPlugin,
        FormulaPlugin,
        HistoryPlugin,
        CollaborationPlugin.configure({ socket, room: 'sheet-123' })
      ]}
      theme={{ 
        primaryColor: '#0052CC', 
        fontFamily: 'Inter',
        cellHeight: 32 
      }}
      onDataChange={(newData) => saveToDatabase(newData)}
    />
  );
}
```