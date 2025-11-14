###USER
ok then great so lets start to build the project using react, redux-toolkit , react-konva , tailwind css 4 using react-vite method

first we will create Designer page with the things that we brainstrom afterwards we will built apis and integrate it , going forward we will add feature we want , and in this chat plz provide your suggestion as well while we progress

###RESPONSE
Perfect 👌 — this is a great stack for what we’re building.
Let’s kick off the **React-Vite project** setup and outline the initial folder structure for the **Designer Page** (Canvas Editor).

---

## 🧱 Step 1 — Project Setup

### 1️⃣ Create a new Vite + React + TypeScript app

```bash
npm create vite@latest canva-designer -- --template react-ts
cd canva-designer
```

### 2️⃣ Install Dependencies

```bash
npm install @reduxjs/toolkit react-redux react-konva konva tailwindcss postcss autoprefixer classnames
```

### 3️⃣ Initialize Tailwind CSS (v4)

```bash
npx tailwindcss init -p
```

Then configure **`tailwind.config.js`**:

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{ts,tsx}"],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

Create the **`src/index.css`**:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

## 🧩 Step 2 — Folder Structure (Initial)

```
canva-designer/
├─ src/
│  ├─ app/
│  │  └─ store.ts
│  ├─ features/
│  │  ├─ canvas/
│  │  │  ├─ CanvasSlice.ts
│  │  │  └─ components/
│  │  │     ├─ CanvasStage.tsx
│  │  │     ├─ FloatingToolbar.tsx
│  │  │     ├─ BottomToolbar.tsx
│  │  │     ├─ LeftPanel.tsx
│  │  │     ├─ RightPanel.tsx
│  │  │     └─ TopBar.tsx
│  ├─ pages/
│  │  └─ DesignerPage.tsx
│  ├─ types/
│  │  └─ design.ts
│  ├─ utils/
│  ├─ main.tsx
│  └─ index.css
└─ package.json
```

---

## 🧠 Step 3 — Core Concept (Redux State Structure)

We’ll start simple and grow it incrementally:

```ts
// src/features/canvas/CanvasSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

export interface CanvasElement {
  id: string;
  type: "text" | "image" | "shape";
  x: number;
  y: number;
  width?: number;
  height?: number;
  text?: string;
  placeholder?: boolean;
  style?: Record<string, any>;
}

interface CanvasState {
  elements: CanvasElement[];
  activeElementId: string | null;
  showBottomToolbar: boolean;
}

const initialState: CanvasState = {
  elements: [],
  activeElementId: null,
  showBottomToolbar: false,
};

const canvasSlice = createSlice({
  name: "canvas",
  initialState,
  reducers: {
    addElement: (state, action: PayloadAction<CanvasElement>) => {
      state.elements.push(action.payload);
    },
    selectElement: (state, action: PayloadAction<string | null>) => {
      state.activeElementId = action.payload;
    },
    toggleBottomToolbar: (state, action: PayloadAction<boolean>) => {
      state.showBottomToolbar = action.payload;
    },
  },
});

export const { addElement, selectElement, toggleBottomToolbar } = canvasSlice.actions;
export default canvasSlice.reducer;
```

---

## 🧭 Step 4 — App Store Setup

```ts
// src/app/store.ts
import { configureStore } from "@reduxjs/toolkit";
import canvasReducer from "../features/canvas/CanvasSlice";

export const store = configureStore({
  reducer: {
    canvas: canvasReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

In `main.tsx`:

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import "./index.css";
import { Provider } from "react-redux";
import { store } from "./app/store";
import DesignerPage from "./pages/DesignerPage";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <Provider store={store}>
      <DesignerPage />
    </Provider>
  </React.StrictMode>
);
```

---

## 🧩 Step 5 — Create the DesignerPage Skeleton

```tsx
// src/pages/DesignerPage.tsx
import LeftPanel from "../features/canvas/components/LeftPanel";
import TopBar from "../features/canvas/components/TopBar";
import CanvasStage from "../features/canvas/components/CanvasStage";
import RightPanel from "../features/canvas/components/RightPanel";
import BottomToolbar from "../features/canvas/components/BottomToolbar";

const DesignerPage = () => {
  return (
    <div className="flex flex-col h-screen bg-gray-50">
      <TopBar />
      <div className="flex flex-1 overflow-hidden">
        <LeftPanel />
        <div className="flex-1 relative flex items-center justify-center">
          <CanvasStage />
        </div>
        <RightPanel />
      </div>
      <BottomToolbar />
    </div>
  );
};

export default DesignerPage;
```

---

✅ **At this point:**
You’ll have the **layout foundation** ready — left tools, top bar, canvas area, right layers, and bottom toolbar (hidden by default).
Next, we’ll start wiring up the **React-Konva canvas** and implement **element selection + floating toolbar** logic.

---

Would you like me to generate the **next step** now — setting up the **React-Konva CanvasStage** component with selection and placeholder rectangle/text rendering?

Excellent 🚀 — now let’s set up the **React-Konva** canvas where all design elements (text, shapes, images, placeholders) will be rendered, selected, and interacted with.

This will be our **core editing surface**, and we’ll integrate it with Redux so all element changes are tracked globally.

---

## 🧩 Step 1 — Create the CanvasStage component

### File: `src/features/canvas/components/CanvasStage.tsx`

```tsx
import { Stage, Layer, Rect, Text, Image as KonvaImage } from "react-konva";
import { useSelector, useDispatch } from "react-redux";
import type { RootState } from "../../../app/store";
import { selectElement, toggleBottomToolbar } from "../CanvasSlice";
import { useEffect, useState } from "react";
import useImage from "use-image";

const CanvasStage = () => {
  const dispatch = useDispatch();
  const { elements, activeElementId } = useSelector(
    (state: RootState) => state.canvas
  );

  const [stageSize, setStageSize] = useState({ width: 794, height: 1123 }); // A4 px size (at 96 DPI)

  // Handle resizing responsively
  useEffect(() => {
    const handleResize = () => {
      const containerWidth = window.innerWidth - 400; // minus side panels
      const scale = containerWidth / 794;
      setStageSize({
        width: 794 * scale,
        height: 1123 * scale,
      });
    };
    handleResize();
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  // Custom image renderer with hook
  const RenderImage = ({ src, x, y, width, height, id }: any) => {
    const [image] = useImage(src);
    return (
      <KonvaImage
        image={image}
        x={x}
        y={y}
        width={width}
        height={height}
        stroke={id === activeElementId ? "#2563eb" : "transparent"}
        strokeWidth={2}
        onClick={() => {
          dispatch(selectElement(id));
          dispatch(toggleBottomToolbar(false));
        }}
      />
    );
  };

  return (
    <div className="bg-white border shadow-inner w-full h-full flex justify-center items-center">
      <Stage
        width={stageSize.width}
        height={stageSize.height}
        className="bg-white border relative"
        onMouseDown={(e) => {
          // Deselect if clicked on empty space
          if (e.target === e.target.getStage()) {
            dispatch(selectElement(null));
            dispatch(toggleBottomToolbar(false));
          }
        }}
      >
        <Layer>
          {elements.map((el) => {
            if (el.type === "rect" || el.type === "shape") {
              return (
                <Rect
                  key={el.id}
                  x={el.x}
                  y={el.y}
                  width={el.width}
                  height={el.height}
                  fill={el.style?.fill || "#e5e7eb"}
                  stroke={el.id === activeElementId ? "#2563eb" : "transparent"}
                  strokeWidth={2}
                  cornerRadius={4}
                  onClick={() => {
                    dispatch(selectElement(el.id));
                    dispatch(toggleBottomToolbar(false));
                  }}
                />
              );
            }
            if (el.type === "text") {
              return (
                <Text
                  key={el.id}
                  x={el.x}
                  y={el.y}
                  text={el.text}
                  fontSize={el.style?.fontSize || 20}
                  fontStyle={el.style?.bold ? "bold" : "normal"}
                  fill={el.style?.color || "#111827"}
                  onClick={() => {
                    dispatch(selectElement(el.id));
                    dispatch(toggleBottomToolbar(false));
                  }}
                />
              );
            }
            if (el.type === "image" && el.style?.src) {
              return (
                <RenderImage
                  key={el.id}
                  id={el.id}
                  src={el.style.src}
                  x={el.x}
                  y={el.y}
                  width={el.width}
                  height={el.height}
                />
              );
            }
            return null;
          })}
        </Layer>
      </Stage>
    </div>
  );
};

export default CanvasStage;
```

---

## 🧠 Step 2 — Explanation

✅ **What this does**

* Displays a **Konva canvas** with a scalable A4 sheet (default).
* Renders elements from Redux (`text`, `shape`, or `image`).
* Allows **element selection**, with a blue outline on active item.
* Clicking outside clears the selection.
* Scales dynamically with browser size.

---

## 🧩 Step 3 — Sample Elements for Testing

You can temporarily prefill the store with a few demo elements for testing.

Open `CanvasSlice.ts` and modify the `initialState`:

```ts
const initialState: CanvasState = {
  elements: [
    {
      id: "1",
      type: "text",
      text: "Heading Placeholder",
      x: 200,
      y: 150,
      style: { fontSize: 28, color: "#000", bold: true },
    },
    {
      id: "2",
      type: "rect",
      x: 180,
      y: 300,
      width: 300,
      height: 120,
      style: { fill: "#f3f4f6" },
    },
    {
      id: "3",
      type: "image",
      x: 220,
      y: 500,
      width: 200,
      height: 120,
      style: {
        src: "https://via.placeholder.com/200x120.png?text=Logo",
      },
    },
  ],
  activeElementId: null,
  showBottomToolbar: false,
};
```

Then run:

```bash
npm run dev
```

You should now see:

* The A4 canvas
* A heading, a rectangle, and an image placeholder
* Click to select (blue outline shows selection)

---

## 🧠 Next Step Suggestion

Now that the **Canvas rendering and selection logic** works, the next step is to:

> 🔹 Implement the **Floating Toolbar** — showing contextually above the selected element with quick actions like font color, bold, and “More” (to open Bottom Toolbar).

Would you like me to proceed to building the **FloatingToolbar** next (with contextual position calculation and Tailwind styling)?

Awesome 🎨 — this is where the app starts feeling like a **real design tool**.

We’ll now build the **Floating Toolbar** that appears dynamically above a selected element (text, shape, or image).
It will:

* Show contextual controls based on selection type
* Be **position-aware** (follows selected element)
* Use **Tailwind transitions** for a polished feel
* Integrate cleanly with Redux

---

## 🧩 Step 1 — Create the FloatingToolbar component

📁 `src/features/canvas/components/FloatingToolbar.tsx`

```tsx
import { useEffect, useRef, useState } from "react";
import { useSelector, useDispatch } from "react-redux";
import { RootState } from "../../../app/store";
import { toggleBottomToolbar } from "../CanvasSlice";
import classNames from "classnames";

const FloatingToolbar = () => {
  const dispatch = useDispatch();
  const { elements, activeElementId } = useSelector(
    (state: RootState) => state.canvas
  );
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const toolbarRef = useRef<HTMLDivElement | null>(null);

  const activeEl = elements.find((el) => el.id === activeElementId);

  useEffect(() => {
    if (!activeEl) return;

    // Compute approximate position above element (this works well for now)
    const offsetX = activeEl.x + (activeEl.width || 0) / 2 - 60;
    const offsetY = activeEl.y - 40;

    setPosition({
      top: offsetY,
      left: offsetX,
    });
  }, [activeEl]);

  if (!activeEl) return null;

  const toolbarType = activeEl.type;

  const buttonBase =
    "px-2 py-1 text-sm font-medium bg-white border rounded shadow hover:bg-gray-100 transition";

  return (
    <div
      ref={toolbarRef}
      className={classNames(
        "absolute z-50 flex items-center space-x-2 border bg-white shadow-lg rounded-lg p-2 transition-opacity duration-200",
        "pointer-events-auto"
      )}
      style={{
        top: `${position.top}px`,
        left: `${position.left}px`,
      }}
    >
      {/* Toolbar options vary by element type */}
      {toolbarType === "text" && (
        <>
          <button className={buttonBase}>B</button>
          <button className={buttonBase}>I</button>
          <button className={buttonBase}>🎨</button>
          <button
            className={buttonBase}
            onClick={() => dispatch(toggleBottomToolbar(true))}
          >
            ⋯ More
          </button>
        </>
      )}

      {toolbarType === "shape" && (
        <>
          <button className={buttonBase}>Fill 🎨</button>
          <button
            className={buttonBase}
            onClick={() => dispatch(toggleBottomToolbar(true))}
          >
            ⋯ More
          </button>
        </>
      )}

      {toolbarType === "image" && (
        <>
          <button className={buttonBase}>Crop ✂️</button>
          <button className={buttonBase}>Mask ⭕</button>
          <button
            className={buttonBase}
            onClick={() => dispatch(toggleBottomToolbar(true))}
          >
            ⋯ More
          </button>
        </>
      )}
    </div>
  );
};

export default FloatingToolbar;
```

---

## 🧠 Step 2 — Render FloatingToolbar in DesignerPage

Open `src/pages/DesignerPage.tsx` and update it:

```tsx
import LeftPanel from "../features/canvas/components/LeftPanel";
import TopBar from "../features/canvas/components/TopBar";
import CanvasStage from "../features/canvas/components/CanvasStage";
import RightPanel from "../features/canvas/components/RightPanel";
import BottomToolbar from "../features/canvas/components/BottomToolbar";
import FloatingToolbar from "../features/canvas/components/FloatingToolbar";
import { useSelector } from "react-redux";
import { RootState } from "../app/store";

const DesignerPage = () => {
  const { activeElementId } = useSelector((state: RootState) => state.canvas);

  return (
    <div className="flex flex-col h-screen bg-gray-50">
      <TopBar />
      <div className="flex flex-1 overflow-hidden relative">
        <LeftPanel />
        <div className="flex-1 relative flex items-center justify-center">
          <CanvasStage />
          {activeElementId && <FloatingToolbar />}
        </div>
        <RightPanel />
      </div>
      <BottomToolbar />
    </div>
  );
};

export default DesignerPage;
```

---

## 🧩 Step 3 — Add Styling for Floating Toolbar Transitions

You can refine the look with Tailwind utility classes or custom CSS transitions.

Example animation (optional):

```css
/* in index.css */
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-5px); }
  to { opacity: 1; transform: translateY(0); }
}

.floating-toolbar {
  animation: fadeIn 0.2s ease-out forwards;
}
```

Then apply:

```tsx
<div className="absolute z-50 floating-toolbar ...">
```

---

## 🧠 Step 4 — How it Works

* When a user **clicks an element**, Redux sets `activeElementId`.
* The toolbar positions itself above the selected element.
* Each element type shows relevant controls:

  * **Text** → Bold, Italic, Color, More
  * **Shape** → Fill, More
  * **Image** → Crop, Mask, More
* Clicking **“⋯ More”** triggers the bottom toolbar to expand.

---

## ✅ Test the Flow

1. Run the app:

   ```bash
   npm run dev
   ```
2. Click on a text, shape, or image in the canvas.
3. Watch the floating toolbar appear above it.
4. Press “⋯ More” — the bottom toolbar (which we’ll make next) will appear.

---

## 💡 Next Step Suggestion

Let’s now build the **Bottom Toolbar (Advanced Controls)** that expands upward when “⋯ More” is clicked.
It will include:

* For **Text** → font family, size, color picker, spacing sliders
* For **Image** → replace, crop, opacity
* For **Shape** → color, border, radius

Would you like me to proceed with building that Bottom Toolbar next?

Perfect 🔥 let’s now implement the **Bottom Toolbar** — the styling and control panel that appears when the user selects an element (like a text box or image).

We’ll design it as a **hybrid panel** — it slides up from the bottom and displays contextual tools depending on the active selection (text, image, shape, etc.).

---

## 🧱 Folder Structure (so far)

```
src/
 ├── components/
 │    ├── CanvasDesigner.tsx
 │    ├── FloatingToolbar.tsx
 │    ├── BottomToolbar.tsx   👈 we’re building this
 │
 ├── store/
 │    ├── index.ts
 │    ├── designSlice.ts
 │
 ├── pages/
 │    └── DesignerPage.tsx
 │
 ├── App.tsx
 └── main.tsx
```

---

## 🎨 UI/UX Goals

* Fixed or floating bar at the **bottom center**
* Auto expands with context (text, image, shape)
* Uses smooth transitions and shadows for modern UX
* Includes **tabs** for style categories (Text, Shape, Image)
* Syncs state with Redux store (`selectedElement`, `type`, `properties`)

---

## ⚛️ `BottomToolbar.tsx`

```tsx
import React from "react";
import { useSelector, useDispatch } from "react-redux";
import { RootState } from "../store";
import { updateElement } from "../store/designSlice";

const BottomToolbar: React.FC = () => {
  const dispatch = useDispatch();
  const selectedElement = useSelector(
    (state: RootState) => state.design.selectedElement
  );

  if (!selectedElement) return null;

  const { type, properties } = selectedElement;

  const handleChange = (prop: string, value: any) => {
    dispatch(updateElement({ id: selectedElement.id, prop, value }));
  };

  return (
    <div className="fixed bottom-0 left-0 right-0 bg-white border-t shadow-lg p-3 flex flex-col gap-3 transition-all duration-300">
      {/* Header */}
      <div className="flex justify-between items-center">
        <h3 className="text-sm font-semibold text-gray-700">
          {type === "text" && "Text Settings"}
          {type === "image" && "Image Settings"}
          {type === "shape" && "Shape Settings"}
        </h3>
      </div>

      {/* Conditional content */}
      {type === "text" && (
        <div className="flex flex-wrap items-center gap-4">
          <div>
            <label className="text-xs text-gray-500">Font Size</label>
            <input
              type="number"
              value={properties.fontSize}
              onChange={(e) => handleChange("fontSize", +e.target.value)}
              className="border rounded px-2 py-1 w-20 text-sm"
            />
          </div>

          <div>
            <label className="text-xs text-gray-500">Color</label>
            <input
              type="color"
              value={properties.color}
              onChange={(e) => handleChange("color", e.target.value)}
              className="w-8 h-8"
            />
          </div>

          <button
            onClick={() => handleChange("bold", !properties.bold)}
            className={`px-3 py-1 border rounded text-sm ${
              properties.bold ? "bg-gray-200 font-bold" : ""
            }`}
          >
            B
          </button>
        </div>
      )}

      {type === "image" && (
        <div className="flex gap-4 items-center">
          <div>
            <label className="text-xs text-gray-500">Opacity</label>
            <input
              type="range"
              min="0.1"
              max="1"
              step="0.1"
              value={properties.opacity}
              onChange={(e) => handleChange("opacity", +e.target.value)}
            />
          </div>

          <button
            onClick={() => handleChange("maskShape", "circle")}
            className="px-3 py-1 border rounded text-sm"
          >
            Apply Mask
          </button>
        </div>
      )}
    </div>
  );
};

export default BottomToolbar;
```

---

## 🧩 Redux: `designSlice.ts` (update snippet)

Make sure the slice has `selectedElement` and `updateElement` reducer:

```ts
updateElement: (state, action) => {
  const { id, prop, value } = action.payload;
  const element = state.elements.find((el) => el.id === id);
  if (element) {
    element.properties[prop] = value;
  }
},
```

---

## ✨ Behavior Flow

* When a user selects an object in the canvas →
  Redux updates `selectedElement`.
* `BottomToolbar` detects this and slides up.
* Based on the element type, it shows relevant controls.
* When deselected → toolbar slides down / hides.

---

Would you like me to **add animation + tabbed categories (Text | Shape | Image)** for the bottom toolbar next, or keep it minimal for now?
Perfect 😎 — we’ll **upgrade the Bottom Toolbar** into a **tabbed, animated hybrid control panel**.
This version feels more like a real design tool (like Canva/Figma) — modern, sleek, and contextual.

---

## 🧩 Goal Recap

We’ll add:

* **Smooth slide-up animation** when an element is selected
* **Tabs** for categories → `Text | Image | Shape`
* **Contextual controls** shown per tab
* **Persistent layout** so you can extend later (for alignment, layering, etc.)

---

## 🎨 Updated `BottomToolbar.tsx`

```tsx
import React, { useState, useEffect } from "react";
import { useSelector, useDispatch } from "react-redux";
import { RootState } from "../store";
import { updateElement } from "../store/designSlice";
import { motion, AnimatePresence } from "framer-motion";

const tabs = ["Text", "Image", "Shape"];

const BottomToolbar: React.FC = () => {
  const dispatch = useDispatch();
  const selectedElement = useSelector(
    (state: RootState) => state.design.selectedElement
  );
  const [activeTab, setActiveTab] = useState<string>("Text");

  useEffect(() => {
    if (selectedElement) {
      setActiveTab(
        selectedElement.type.charAt(0).toUpperCase() + selectedElement.type.slice(1)
      );
    }
  }, [selectedElement]);

  if (!selectedElement) return null;

  const { type, properties } = selectedElement;

  const handleChange = (prop: string, value: any) => {
    dispatch(updateElement({ id: selectedElement.id, prop, value }));
  };

  return (
    <AnimatePresence>
      {selectedElement && (
        <motion.div
          initial={{ y: 100, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          exit={{ y: 100, opacity: 0 }}
          transition={{ type: "spring", stiffness: 120, damping: 15 }}
          className="fixed bottom-0 left-0 right-0 bg-white border-t shadow-lg p-4 z-50"
        >
          {/* Tabs */}
          <div className="flex justify-center gap-6 mb-3">
            {tabs.map((tab) => (
              <button
                key={tab}
                onClick={() => setActiveTab(tab)}
                className={`px-4 py-1 rounded-full text-sm font-medium transition-all ${
                  activeTab === tab
                    ? "bg-blue-600 text-white"
                    : "text-gray-600 hover:bg-gray-100"
                }`}
              >
                {tab}
              </button>
            ))}
          </div>

          {/* Tab Content */}
          <div className="flex flex-wrap gap-4 justify-center items-center transition-all">
            {activeTab === "Text" && type === "text" && (
              <>
                <div>
                  <label className="text-xs text-gray-500">Font Size</label>
                  <input
                    type="number"
                    value={properties.fontSize}
                    onChange={(e) => handleChange("fontSize", +e.target.value)}
                    className="border rounded px-2 py-1 w-20 text-sm"
                  />
                </div>

                <div>
                  <label className="text-xs text-gray-500">Color</label>
                  <input
                    type="color"
                    value={properties.color}
                    onChange={(e) => handleChange("color", e.target.value)}
                    className="w-8 h-8"
                  />
                </div>

                <button
                  onClick={() => handleChange("bold", !properties.bold)}
                  className={`px-3 py-1 border rounded text-sm ${
                    properties.bold ? "bg-gray-200 font-bold" : ""
                  }`}
                >
                  B
                </button>

                <button
                  onClick={() => handleChange("italic", !properties.italic)}
                  className={`px-3 py-1 border rounded text-sm italic ${
                    properties.italic ? "bg-gray-200" : ""
                  }`}
                >
                  I
                </button>
              </>
            )}

            {activeTab === "Image" && type === "image" && (
              <>
                <div>
                  <label className="text-xs text-gray-500">Opacity</label>
                  <input
                    type="range"
                    min="0.1"
                    max="1"
                    step="0.1"
                    value={properties.opacity}
                    onChange={(e) => handleChange("opacity", +e.target.value)}
                    className="w-32"
                  />
                </div>

                <button
                  onClick={() => handleChange("maskShape", "circle")}
                  className="px-3 py-1 border rounded text-sm"
                >
                  Apply Mask
                </button>

                <button
                  onClick={() => handleChange("cropMode", !properties.cropMode)}
                  className={`px-3 py-1 border rounded text-sm ${
                    properties.cropMode ? "bg-gray-200" : ""
                  }`}
                >
                  Crop
                </button>
              </>
            )}

            {activeTab === "Shape" && type === "shape" && (
              <>
                <div>
                  <label className="text-xs text-gray-500">Fill Color</label>
                  <input
                    type="color"
                    value={properties.fill}
                    onChange={(e) => handleChange("fill", e.target.value)}
                    className="w-8 h-8"
                  />
                </div>

                <div>
                  <label className="text-xs text-gray-500">Border Width</label>
                  <input
                    type="number"
                    min="0"
                    max="10"
                    value={properties.strokeWidth}
                    onChange={(e) => handleChange("strokeWidth", +e.target.value)}
                    className="border rounded px-2 py-1 w-20 text-sm"
                  />
                </div>
              </>
            )}
          </div>
        </motion.div>
      )}
    </AnimatePresence>
  );
};

export default BottomToolbar;
```

---

## ✨ Features Added

✅ Animated slide-up / slide-down
✅ Tab navigation (Text | Image | Shape)
✅ Context-aware controls
✅ Framer Motion for smooth UX
✅ Works seamlessly with Redux state

---

## 🧠 Suggestion for next iteration

We can now extend this to include:

* 🎨 **Presets** (text styles, color themes)
* 🧱 **Auto-alignment & snapping tools**
* 🔄 **Undo / Redo**
* 📏 **Grid & Ruler visibility toggle**

---

Would you like to proceed with adding **animation for appearing/disappearing toolbars (Floating + Bottom coordinated)** next — so that when a user selects an element, both float and bottom panels appear in sync?

Awesome 😎 — now we’ll make the **Floating Toolbar** and **Bottom Toolbar** feel *connected* — just like Canva or Figma’s hybrid interaction model.

When a user selects a text/image/shape:

* A **floating mini toolbar** appears *right above* the selection (for quick inline edits).
* A **bottom contextual panel** slides up *simultaneously* (for deeper styling options).
* Both disappear when deselected.
* Smooth, coordinated animation between them (using **Framer Motion**).

---

## 🧠 Concept Recap

| Toolbar Type | Purpose              | Example Actions                         |
| ------------ | -------------------- | --------------------------------------- |
| Floating     | Inline quick edits   | Bold, Italic, Align, Delete             |
| Bottom       | Deep styling control | Font size, Color, Opacity, Mask, Stroke |

---

## ⚛️ Step 1 – Update `FloatingToolbar.tsx`

We’ll make it **motion-aware** and synced with selection state.

```tsx
import React, { useEffect, useRef, useState } from "react";
import { useSelector, useDispatch } from "react-redux";
import { RootState } from "../store";
import { motion, AnimatePresence } from "framer-motion";
import { updateElement, deleteElement } from "../store/designSlice";

const FloatingToolbar: React.FC = () => {
  const selectedElement = useSelector(
    (state: RootState) => state.design.selectedElement
  );
  const dispatch = useDispatch();
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    if (selectedElement) {
      // Simulate element position (if your canvas uses Konva, fetch node.getClientRect())
      setPosition({
        x: selectedElement.properties.x + 80,
        y: selectedElement.properties.y - 60,
      });
    }
  }, [selectedElement]);

  if (!selectedElement) return null;

  const { type, properties } = selectedElement;

  return (
    <AnimatePresence>
      {selectedElement && (
        <motion.div
          initial={{ opacity: 0, scale: 0.9, y: -10 }}
          animate={{ opacity: 1, scale: 1, y: 0 }}
          exit={{ opacity: 0, scale: 0.9, y: -10 }}
          transition={{ duration: 0.2 }}
          style={{
            position: "absolute",
            top: position.y,
            left: position.x,
            transform: "translate(-50%, -100%)",
            zIndex: 999,
          }}
          className="flex gap-2 bg-white border shadow-lg rounded-lg px-3 py-2 items-center"
        >
          {/* Quick actions */}
          {type === "text" && (
            <>
              <button
                className={`text-sm font-bold ${
                  properties.bold ? "text-blue-600" : "text-gray-700"
                }`}
                onClick={() =>
                  dispatch(
                    updateElement({
                      id: selectedElement.id,
                      prop: "bold",
                      value: !properties.bold,
                    })
                  )
                }
              >
                B
              </button>
              <button
                className={`text-sm italic ${
                  properties.italic ? "text-blue-600" : "text-gray-700"
                }`}
                onClick={() =>
                  dispatch(
                    updateElement({
                      id: selectedElement.id,
                      prop: "italic",
                      value: !properties.italic,
                    })
                  )
                }
              >
                I
              </button>
              <button
                onClick={() =>
                  dispatch(
                    updateElement({
                      id: selectedElement.id,
                      prop: "align",
                      value:
                        properties.align === "center" ? "left" : "center",
                    })
                  )
                }
                className="text-sm text-gray-700"
              >
                ⬌
              </button>
            </>
          )}

          {/* Common delete option */}
          <button
            onClick={() => dispatch(deleteElement(selectedElement.id))}
            className="text-red-500 text-xs"
          >
            🗑
          </button>
        </motion.div>
      )}
    </AnimatePresence>
  );
};

export default FloatingToolbar;
```

---

## ⚛️ Step 2 – Synchronize with Bottom Toolbar

Since both are tied to `selectedElement`, they’ll now animate **in sync** automatically via Redux.

If you want *visual coordination* (e.g., fade in both), wrap them together in the designer view:

### `CanvasDesigner.tsx` (integration)

```tsx
import React from "react";
import FloatingToolbar from "./FloatingToolbar";
import BottomToolbar from "./BottomToolbar";
import { Stage, Layer } from "react-konva";

const CanvasDesigner: React.FC = () => {
  return (
    <div className="relative w-full h-full bg-gray-100">
      {/* Canvas area */}
      <Stage width={1200} height={700} className="mx-auto bg-white shadow-md">
        <Layer>
          {/* Elements will render here */}
        </Layer>
      </Stage>

      {/* Toolbar overlays */}
      <FloatingToolbar />
      <BottomToolbar />
    </div>
  );
};

export default CanvasDesigner;
```

Now both toolbars respond dynamically:

* When a shape/text/image is clicked → both **appear**
* When deselected → both **fade/slide out** in sync
* Minimal extra state — everything uses the `selectedElement` state from Redux

---

## 🧠 Optional Enhancement (Recommended Next)

We can add:

1. **Delayed hide animation** (Floating hides first → Bottom after 0.3s)
2. **Smart positioning** — Floating toolbar follows mouse / selected item position
3. **Responsive behavior** — On mobile, floating toolbar converts into a small popup

---

Would you like me to add that **coordinated delay animation + smart positioning** behavior next (so it feels like a professional design app)?

Excellent 😎 — now we’re getting into the **“pro-tier design experience”** level — where our `FloatingToolbar` and `BottomToolbar` feel truly coordinated, smooth, and intelligent (like Canva/Figma).

Here’s what we’ll add in this step:
✅ **Smart positioning** — toolbar auto-aligns near the selected element but stays within visible viewport bounds
✅ **Coordinated animation** — Floating toolbar hides first, Bottom toolbar fades/slides out slightly after
✅ **Responsive handling** — reposition toolbar dynamically if the window resizes or the element moves

---

## 🧠 High-Level Flow

| Event                          | FloatingToolbar                   | BottomToolbar               |
| ------------------------------ | --------------------------------- | --------------------------- |
| Element selected               | Fades + slides in near element    | Slides up from bottom       |
| Element deselected             | Fades out immediately             | Waits ~300 ms → slides down |
| Window resize or element moved | Recalculates position dynamically | (no change)                 |

---

## ⚙️ Step 1 – Update `FloatingToolbar.tsx` (Smart Position + Sync Animation)

We’ll calculate toolbar position relative to the element, with viewport clamping and transition sync.

```tsx
import React, { useEffect, useState } from "react";
import { useSelector, useDispatch } from "react-redux";
import { RootState } from "../store";
import { motion, AnimatePresence } from "framer-motion";
import { updateElement, deleteElement } from "../store/designSlice";

const FloatingToolbar: React.FC<{ onHideStart?: () => void }> = ({
  onHideStart,
}) => {
  const dispatch = useDispatch();
  const selectedElement = useSelector(
    (state: RootState) => state.design.selectedElement
  );
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const [visible, setVisible] = useState(false);

  // Calculate position (Konva or absolute coordinates)
  const calculatePosition = () => {
    if (!selectedElement) return;
    const padding = 60;
    let x = selectedElement.properties.x + selectedElement.properties.width / 2;
    let y = selectedElement.properties.y - padding;

    // Clamp within viewport
    const vw = window.innerWidth;
    const vh = window.innerHeight;
    x = Math.min(Math.max(x, 100), vw - 100);
    y = Math.max(y, 80);

    setPosition({ x, y });
  };

  // Handle element selection
  useEffect(() => {
    if (selectedElement) {
      calculatePosition();
      setVisible(true);
    } else {
      // Trigger coordinated fade-out before hiding completely
      if (visible && onHideStart) onHideStart();
      setTimeout(() => setVisible(false), 200);
    }
  }, [selectedElement]);

  // Recalculate on resize
  useEffect(() => {
    window.addEventListener("resize", calculatePosition);
    return () => window.removeEventListener("resize", calculatePosition);
  }, [selectedElement]);

  if (!visible || !selectedElement) return null;

  const { type, properties } = selectedElement;

  return (
    <AnimatePresence>
      {selectedElement && (
        <motion.div
          key="floating-toolbar"
          initial={{ opacity: 0, y: -10, scale: 0.95 }}
          animate={{ opacity: 1, y: 0, scale: 1 }}
          exit={{ opacity: 0, y: -10, scale: 0.95 }}
          transition={{ duration: 0.2 }}
          style={{
            position: "absolute",
            top: position.y,
            left: position.x,
            transform: "translate(-50%, -100%)",
            zIndex: 1000,
          }}
          className="flex gap-2 bg-white border shadow-lg rounded-xl px-3 py-2 items-center"
        >
          {type === "text" && (
            <>
              <button
                className={`text-sm font-bold ${
                  properties.bold ? "text-blue-600" : "text-gray-700"
                }`}
                onClick={() =>
                  dispatch(
                    updateElement({
                      id: selectedElement.id,
                      prop: "bold",
                      value: !properties.bold,
                    })
                  )
                }
              >
                B
              </button>
              <button
                className={`text-sm italic ${
                  properties.italic ? "text-blue-600" : "text-gray-700"
                }`}
                onClick={() =>
                  dispatch(
                    updateElement({
                      id: selectedElement.id,
                      prop: "italic",
                      value: !properties.italic,
                    })
                  )
                }
              >
                I
              </button>
              <button
                onClick={() =>
                  dispatch(
                    updateElement({
                      id: selectedElement.id,
                      prop: "align",
                      value:
                        properties.align === "center" ? "left" : "center",
                    })
                  )
                }
                className="text-sm text-gray-700"
              >
                ⬌
              </button>
            </>
          )}

          <button
            onClick={() => dispatch(deleteElement(selectedElement.id))}
            className="text-red-500 text-xs"
          >
            🗑
          </button>
        </motion.div>
      )}
    </AnimatePresence>
  );
};

export default FloatingToolbar;
```

---

## ⚙️ Step 2 – Update `BottomToolbar.tsx` (Coordinated Delay on Hide)

We’ll add a simple delay on exit when `selectedElement` becomes null.

```tsx
import React, { useState, useEffect } from "react";
import { useSelector, useDispatch } from "react-redux";
import { RootState } from "../store";
import { motion, AnimatePresence } from "framer-motion";
import { updateElement } from "../store/designSlice";

const BottomToolbar: React.FC<{ hideDelay?: boolean }> = ({ hideDelay }) => {
  const dispatch = useDispatch();
  const selectedElement = useSelector(
    (state: RootState) => state.design.selectedElement
  );
  const [visible, setVisible] = useState(false);

  useEffect(() => {
    if (selectedElement) {
      setVisible(true);
    } else {
      if (hideDelay) {
        setTimeout(() => setVisible(false), 300);
      } else {
        setVisible(false);
      }
    }
  }, [selectedElement]);

  if (!visible || !selectedElement) return null;

  const { type, properties } = selectedElement;

  const handleChange = (prop: string, value: any) => {
    dispatch(updateElement({ id: selectedElement.id, prop, value }));
  };

  return (
    <AnimatePresence>
      <motion.div
        key="bottom-toolbar"
        initial={{ y: 100, opacity: 0 }}
        animate={{ y: 0, opacity: 1 }}
        exit={{ y: 100, opacity: 0 }}
        transition={{ duration: 0.35, ease: "easeOut" }}
        className="fixed bottom-0 left-0 right-0 bg-white border-t shadow-lg p-3 flex flex-col gap-3 z-50"
      >
        <h3 className="text-sm font-semibold text-gray-700">
          {type === "text" && "Text Settings"}
          {type === "image" && "Image Settings"}
          {type === "shape" && "Shape Settings"}
        </h3>

        {type === "text" && (
          <div className="flex flex-wrap gap-4">
            <div>
              <label className="text-xs text-gray-500">Font Size</label>
              <input
                type="number"
                value={properties.fontSize}
                onChange={(e) => handleChange("fontSize", +e.target.value)}
                className="border rounded px-2 py-1 w-20 text-sm"
              />
            </div>
            <div>
              <label className="text-xs text-gray-500">Color</label>
              <input
                type="color"
                value={properties.color}
                onChange={(e) => handleChange("color", e.target.value)}
                className="w-8 h-8"
              />
            </div>
          </div>
        )}
      </motion.div>
    </AnimatePresence>
  );
};

export default BottomToolbar;
```

---

## ⚙️ Step 3 – Coordinate Both in `CanvasDesigner.tsx`

The floating toolbar now notifies when it begins hiding — allowing the bottom one to start its fade slightly after.

```tsx
import React, { useState } from "react";
import FloatingToolbar from "./FloatingToolbar";
import BottomToolbar from "./BottomToolbar";
import { Stage, Layer } from "react-konva";

const CanvasDesigner: React.FC = () => {
  const [delayHide, setDelayHide] = useState(false);

  return (
    <div className="relative w-full h-full bg-gray-100">
      <Stage width={1200} height={700} className="mx-auto bg-white shadow-md">
        <Layer>{/* Elements here */}</Layer>
      </Stage>

      <FloatingToolbar onHideStart={() => setDelayHide(true)} />
      <BottomToolbar hideDelay={delayHide} />
    </div>
  );
};

export default CanvasDesigner;
```

---

## 🧠 Summary of Behavior

✅ **Smart Positioning:** Floating toolbar avoids edges & tracks element
✅ **Coordinated Animation:** Floating hides → Bottom hides after 300 ms
✅ **Responsive Repositioning:** Adjusts when window resized
✅ **Feels like professional editing software**

---

Would you like the next enhancement to make this **mobile/tablet adaptive** — where both toolbars merge into a collapsible floating action bar on smaller screens (like Canva mobile UI)?

Perfect 👌 — let’s make the **Floating Toolbar + Bottom Panel** feel “smart” and fluidly responsive.
Here’s our next step — **adding coordinated delay animations + contextual positioning logic**.

---

### 🎯 Goal

When a user selects an element (like text or image):

1. **Floating Toolbar** should fade in and slide above the selected element.
2. **Bottom Panel** should expand slightly later (with a short delay).
3. If the selected element is near the top/bottom edge, the floating toolbar should reposition smartly (above or below the element to avoid clipping).

---

### ⚙️ Implementation Plan

#### 1. Update `FloatingToolbar.tsx`

We’ll use **Framer Motion** for smooth coordinated animations.

```tsx
import { motion, AnimatePresence } from "framer-motion";
import { useEffect, useState } from "react";

interface FloatingToolbarProps {
  x: number;
  y: number;
  visible: boolean;
  type: "text" | "image" | null;
}

export default function FloatingToolbar({ x, y, visible, type }: FloatingToolbarProps) {
  const [position, setPosition] = useState({ top: y, left: x });
  const [flip, setFlip] = useState(false);

  useEffect(() => {
    // Smart reposition logic (avoid going off-screen)
    const viewportHeight = window.innerHeight;
    const shouldFlip = y < 100; // near top
    setFlip(shouldFlip);
    setPosition({
      top: shouldFlip ? y + 60 : y - 60,
      left: x,
    });
  }, [x, y]);

  const variants = {
    hidden: { opacity: 0, y: flip ? -10 : 10 },
    visible: {
      opacity: 1,
      y: 0,
      transition: { duration: 0.25, ease: "easeOut" },
    },
  };

  return (
    <AnimatePresence>
      {visible && (
        <motion.div
          className="absolute bg-white/90 shadow-md rounded-xl p-2 flex gap-2 border border-gray-200"
          initial="hidden"
          animate="visible"
          exit="hidden"
          variants={variants}
          style={{
            top: position.top,
            left: position.left,
            transform: "translate(-50%, -50%)",
          }}
        >
          {type === "text" ? (
            <>
              <button className="px-2 py-1 hover:bg-gray-100 rounded">B</button>
              <button className="px-2 py-1 hover:bg-gray-100 rounded">I</button>
              <button className="px-2 py-1 hover:bg-gray-100 rounded">U</button>
            </>
          ) : (
            <>
              <button className="px-2 py-1 hover:bg-gray-100 rounded">Crop</button>
              <button className="px-2 py-1 hover:bg-gray-100 rounded">Mask</button>
              <button className="px-2 py-1 hover:bg-gray-100 rounded">Flip</button>
            </>
          )}
        </motion.div>
      )}
    </AnimatePresence>
  );
}
```

---

#### 2. Update `BottomToolbar.tsx`

We’ll sync its animation to appear **after** the floating toolbar (using a small delay).

```tsx
import { motion, AnimatePresence } from "framer-motion";

interface BottomToolbarProps {
  visible: boolean;
  type: "text" | "image" | null;
}

export default function BottomToolbar({ visible, type }: BottomToolbarProps) {
  const variants = {
    hidden: { opacity: 0, y: 50 },
    visible: {
      opacity: 1,
      y: 0,
      transition: { delay: 0.25, duration: 0.35, ease: "easeOut" },
    },
  };

  return (
    <AnimatePresence>
      {visible && (
        <motion.div
          className="fixed bottom-0 left-0 right-0 bg-gray-50 border-t border-gray-200 p-3 shadow-lg flex justify-center gap-3"
          initial="hidden"
          animate="visible"
          exit="hidden"
          variants={variants}
        >
          {type === "text" ? (
            <>
              <button className="px-3 py-1 bg-white border rounded hover:bg-gray-100">Font</button>
              <button className="px-3 py-1 bg-white border rounded hover:bg-gray-100">Color</button>
              <button className="px-3 py-1 bg-white border rounded hover:bg-gray-100">Align</button>
            </>
          ) : (
            <>
              <button className="px-3 py-1 bg-white border rounded hover:bg-gray-100">Resize</button>
              <button className="px-3 py-1 bg-white border rounded hover:bg-gray-100">Opacity</button>
              <button className="px-3 py-1 bg-white border rounded hover:bg-gray-100">Filter</button>
            </>
          )}
        </motion.div>
      )}
    </AnimatePresence>
  );
}
```

---

#### 3. Orchestrate the Two Components

When an element is selected on the canvas, trigger:

```ts
setSelectedElement({ x, y, type: "text" });
setShowToolbar(true);
```

✅ Floating toolbar appears first.
✅ After 250ms, bottom panel expands smoothly.
✅ Repositions dynamically to avoid overlap or clipping.

---

Would you like me to **add the integration snippet inside the main `DesignerPage.tsx`** (to show how both toolbars coordinate together when selecting text/image)?

Perfect 😎
Now let’s wire everything together inside the **`DesignerPage.tsx`** — this is where our canvas (using `react-konva`) will live, and where we’ll coordinate the **floating toolbar** and **bottom toolbar** behaviors.

---

## 🧠 DesignerPage Overview

* Canvas area → allows selecting elements (text/image)
* When selected → triggers both toolbars with animations
* Smart delay ensures coordinated appearance
* Future-ready → can easily extend for shapes, placeholders, etc.

---

### 🧩 `DesignerPage.tsx`

```tsx
import React, { useState, useRef } from "react";
import { Stage, Layer, Rect, Text, Image } from "react-konva";
import FloatingToolbar from "../components/FloatingToolbar";
import BottomToolbar from "../components/BottomToolbar";

interface SelectedElement {
  x: number;
  y: number;
  type: "text" | "image" | null;
}

export default function DesignerPage() {
  const [selected, setSelected] = useState<SelectedElement | null>(null);
  const [showToolbar, setShowToolbar] = useState(false);
  const stageRef = useRef<any>(null);

  const handleSelect = (e: any, type: "text" | "image") => {
    const shape = e.target;
    const stage = stageRef.current;
    const position = shape.getAbsolutePosition();
    const scale = stage.scaleX();

    setSelected({
      x: position.x * scale + shape.width() / 2,
      y: position.y * scale,
      type,
    });

    setShowToolbar(true);
  };

  const handleDeselect = (e: any) => {
    // Deselect if click empty area
    const clickedOnEmpty = e.target === e.target.getStage();
    if (clickedOnEmpty) {
      setShowToolbar(false);
      setSelected(null);
    }
  };

  return (
    <div className="w-full h-screen flex flex-col bg-gray-100 relative overflow-hidden">
      {/* Header */}
      <header className="h-12 flex items-center justify-between bg-white px-4 shadow-sm border-b">
        <h1 className="font-semibold text-gray-700">🎨 Designer</h1>
        <div className="flex gap-2 items-center">
          <label className="text-sm text-gray-600">Canvas Size:</label>
          <select className="border rounded px-2 py-1 text-sm">
            <option>A4 (210x297mm)</option>
            <option>Letter (8.5x11in)</option>
            <option>Custom</option>
          </select>
        </div>
      </header>

      {/* Canvas */}
      <div className="flex-1 relative">
        <Stage
          width={window.innerWidth}
          height={window.innerHeight - 100}
          onMouseDown={handleDeselect}
          ref={stageRef}
          className="bg-white shadow-inner border border-gray-200 mx-auto my-3 rounded-lg"
        >
          <Layer>
            {/* Example Image */}
            <Rect
              x={150}
              y={120}
              width={200}
              height={150}
              fill="#d0e6ff"
              shadowBlur={4}
              onClick={(e) => handleSelect(e, "image")}
            />
            {/* Example Text */}
            <Text
              x={200}
              y={300}
              text="Sample Text"
              fontSize={24}
              fontFamily="Arial"
              fill="black"
              draggable
              onClick={(e) => handleSelect(e, "text")}
            />
          </Layer>
        </Stage>

        {/* Floating Toolbar */}
        {selected && (
          <FloatingToolbar
            x={selected.x}
            y={selected.y}
            visible={showToolbar}
            type={selected.type}
          />
        )}

        {/* Bottom Toolbar */}
        <BottomToolbar visible={showToolbar} type={selected?.type || null} />
      </div>
    </div>
  );
}
```

---

### 🎬 Flow Summary

1. Click on **text or image** on the canvas → triggers selection.
2. Floating toolbar appears **instantly** above or below the selected item.
3. Bottom toolbar expands **after 250ms delay** for styling tools.
4. Click empty space → hides both toolbars.

---

### 💡 Next Steps (Suggestions)

We can now easily extend:

1. Add **keyboard shortcuts** (Delete, Ctrl+D, etc.)
2. Add **drag handles** for resizing text/images.
3. Connect **toolbar actions** to modify the Konva elements live.
4. Add **undo/redo** support using Redux Toolkit.

---

Would you like to proceed next with connecting the **toolbar actions (font size, bold, color, etc.)** to the selected text on the canvas — so that toolbar interactions actually update the design in real-time?

🔥 Awesome — this is where your **Designer** really starts to feel like a mini Canva.
We’ll now connect the **Floating Toolbar** and **Bottom Toolbar** so their actions directly modify the selected text or image on the Konva canvas — in real time.

Let’s begin with **Text editing features** (we’ll do image tools next).

---

## 🧠 Concept

When a text element is selected:

* The `FloatingToolbar` will let you toggle **bold**, **italic**, **underline**.
* The `BottomToolbar` will handle **font size**, **color**, and **alignment**.
* All updates will immediately reflect on the selected Konva `Text` node.

We’ll use **React state + refs** first, then later optimize via Redux Toolkit.

---

## 1️⃣ Extend `DesignerPage.tsx`

We’ll store a reference to the selected Konva node (text or image) and expose update functions.

```tsx
import React, { useState, useRef } from "react";
import { Stage, Layer, Rect, Text } from "react-konva";
import FloatingToolbar from "../components/FloatingToolbar";
import BottomToolbar from "../components/BottomToolbar";

interface SelectedElement {
  x: number;
  y: number;
  type: "text" | "image" | null;
  node: any; // Reference to the Konva node
}

export default function DesignerPage() {
  const [selected, setSelected] = useState<SelectedElement | null>(null);
  const [showToolbar, setShowToolbar] = useState(false);
  const stageRef = useRef<any>(null);

  const handleSelect = (e: any, type: "text" | "image") => {
    const shape = e.target;
    const stage = stageRef.current;
    const position = shape.getAbsolutePosition();
    const scale = stage.scaleX();

    setSelected({
      x: position.x * scale + shape.width() / 2,
      y: position.y * scale,
      type,
      node: shape,
    });
    setShowToolbar(true);
  };

  const handleDeselect = (e: any) => {
    if (e.target === e.target.getStage()) {
      setSelected(null);
      setShowToolbar(false);
    }
  };

  const handleUpdateText = (updates: any) => {
    if (selected?.node && selected.type === "text") {
      selected.node.setAttrs(updates);
      selected.node.getLayer().batchDraw();
    }
  };

  return (
    <div className="w-full h-screen flex flex-col bg-gray-100 relative overflow-hidden">
      {/* Header */}
      <header className="h-12 flex items-center justify-between bg-white px-4 shadow-sm border-b">
        <h1 className="font-semibold text-gray-700">🎨 Designer</h1>
        <div className="flex gap-2 items-center">
          <label className="text-sm text-gray-600">Canvas Size:</label>
          <select className="border rounded px-2 py-1 text-sm">
            <option>A4 (210x297mm)</option>
            <option>Letter (8.5x11in)</option>
          </select>
        </div>
      </header>

      {/* Canvas */}
      <div className="flex-1 relative">
        <Stage
          width={window.innerWidth}
          height={window.innerHeight - 100}
          onMouseDown={handleDeselect}
          ref={stageRef}
          className="bg-white shadow-inner border border-gray-200 mx-auto my-3 rounded-lg"
        >
          <Layer>
            {/* Example Image */}
            <Rect
              x={150}
              y={120}
              width={200}
              height={150}
              fill="#d0e6ff"
              shadowBlur={4}
              onClick={(e) => handleSelect(e, "image")}
            />
            {/* Example Text */}
            <Text
              x={200}
              y={300}
              text="Sample Text"
              fontSize={24}
              fontFamily="Arial"
              fill="black"
              draggable
              onClick={(e) => handleSelect(e, "text")}
            />
          </Layer>
        </Stage>

        {selected && (
          <>
            <FloatingToolbar
              x={selected.x}
              y={selected.y}
              visible={showToolbar}
              type={selected.type}
              onTextStyleChange={handleUpdateText}
            />
            <BottomToolbar
              visible={showToolbar}
              type={selected.type}
              onTextStyleChange={handleUpdateText}
            />
          </>
        )}
      </div>
    </div>
  );
}
```

---

## 2️⃣ Update `FloatingToolbar.tsx`

Now it will send style updates (bold, italic, underline) back to the parent.

```tsx
import { motion, AnimatePresence } from "framer-motion";
import { useEffect, useState } from "react";

interface FloatingToolbarProps {
  x: number;
  y: number;
  visible: boolean;
  type: "text" | "image" | null;
  onTextStyleChange?: (updates: any) => void;
}

export default function FloatingToolbar({
  x,
  y,
  visible,
  type,
  onTextStyleChange,
}: FloatingToolbarProps) {
  const [position, setPosition] = useState({ top: y, left: x });
  const [flip, setFlip] = useState(false);

  useEffect(() => {
    const shouldFlip = y < 100;
    setFlip(shouldFlip);
    setPosition({
      top: shouldFlip ? y + 60 : y - 60,
      left: x,
    });
  }, [x, y]);

  const variants = {
    hidden: { opacity: 0, y: flip ? -10 : 10 },
    visible: { opacity: 1, y: 0, transition: { duration: 0.25 } },
  };

  const handleStyle = (style: string) => {
    if (!onTextStyleChange) return;
    switch (style) {
      case "bold":
        onTextStyleChange({ fontStyle: "bold" });
        break;
      case "italic":
        onTextStyleChange({ fontStyle: "italic" });
        break;
      case "underline":
        onTextStyleChange({ textDecoration: "underline" });
        break;
    }
  };

  return (
    <AnimatePresence>
      {visible && (
        <motion.div
          className="absolute bg-white/90 shadow-md rounded-xl p-2 flex gap-2 border border-gray-200"
          initial="hidden"
          animate="visible"
          exit="hidden"
          variants={variants}
          style={{
            top: position.top,
            left: position.left,
            transform: "translate(-50%, -50%)",
          }}
        >
          {type === "text" && (
            <>
              <button onClick={() => handleStyle("bold")} className="px-2 py-1 hover:bg-gray-100 rounded font-bold">B</button>
              <button onClick={() => handleStyle("italic")} className="px-2 py-1 hover:bg-gray-100 rounded italic">I</button>
              <button onClick={() => handleStyle("underline")} className="px-2 py-1 hover:bg-gray-100 rounded underline">U</button>
            </>
          )}
        </motion.div>
      )}
    </AnimatePresence>
  );
}
```

---

## 3️⃣ Update `BottomToolbar.tsx`

This one will control **font size**, **color**, and **alignment**.

```tsx
import { motion, AnimatePresence } from "framer-motion";

interface BottomToolbarProps {
  visible: boolean;
  type: "text" | "image" | null;
  onTextStyleChange?: (updates: any) => void;
}

export default function BottomToolbar({ visible, type, onTextStyleChange }: BottomToolbarProps) {
  const variants = {
    hidden: { opacity: 0, y: 50 },
    visible: { opacity: 1, y: 0, transition: { delay: 0.25, duration: 0.35 } },
  };

  const handleFontSize = (size: number) => onTextStyleChange?.({ fontSize: size });
  const handleColor = (color: string) => onTextStyleChange?.({ fill: color });
  const handleAlign = (align: string) => onTextStyleChange?.({ align });

  return (
    <AnimatePresence>
      {visible && (
        <motion.div
          className="fixed bottom-0 left-0 right-0 bg-gray-50 border-t border-gray-200 p-3 shadow-lg flex justify-center gap-3"
          initial="hidden"
          animate="visible"
          exit="hidden"
          variants={variants}
        >
          {type === "text" && (
            <>
              <select onChange={(e) => handleFontSize(Number(e.target.value))} className="border rounded px-2 py-1">
                {[16, 20, 24, 28, 32, 40].map((s) => (
                  <option key={s} value={s}>{s}px</option>
                ))}
              </select>
              <input type="color" onChange={(e) => handleColor(e.target.value)} className="w-10 h-8 border rounded" />
              <select onChange={(e) => handleAlign(e.target.value)} className="border rounded px-2 py-1">
                <option value="left">Left</option>
                <option value="center">Center</option>
                <option value="right">Right</option>
              </select>
            </>
          )}
        </motion.div>
      )}
    </AnimatePresence>
  );
}
```

---

### ✅ Result

Now you can:

* Click text → Floating + Bottom toolbars appear
* Change font size, color, bold/italic/underline → updates instantly
* Future-proof for image editing, placeholders, etc.

---



