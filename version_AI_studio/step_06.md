**USER**

**RESPONSE**

Excellent. Building the Layers Panel is a huge step towards making this a professional-grade tool. It gives users fine-grained control over the composition of their design.

We will build the right-hand sidebar with a tabbed interface. The first tab will be our new "Layers" panel.

---

### **Phase 1, Step 5: Building the Layers Panel**

#### **1. Update the Redux Slice (`editorSlice.ts`)**

First, our elements need new properties to handle visibility and locking. We also need new reducers to manage these properties and the layer order.

**Update `src/features/editor/editorSlice.ts`:**

```typescript
// ... (imports and other interfaces remain the same)

// Add isVisible and isLocked to the base element
export interface BaseElement {
  id: string;
  x: number;
  y: number;
  width?: number; // Make width/height optional on base
  height?: number;
  rotation: number;
  isVisible: boolean;
  isLocked: boolean;
}

export interface TextElement extends BaseElement {
  type: 'text';
  text: string;
  fontSize: number;
  fontFamily: string;
  fill: string;
}

export type CanvasElement = TextElement;

interface EditorState {
  elements: CanvasElement[];
  selectedElementId: string | null;
}

const initialState: EditorState = {
  elements: [],
  selectedElementId: null,
};

const editorSlice = createSlice({
  name: 'editor',
  initialState,
  reducers: {
    addElement: (state, action: PayloadAction<Omit<CanvasElement, 'id' | 'isVisible' | 'isLocked'>>) => {
      const newElement = {
        id: nanoid(),
        isVisible: true, // Default to visible
        isLocked: false,  // Default to unlocked
        ...action.payload,
      };
      // Add the new element to the end of the array (top layer)
      state.elements.push(newElement as CanvasElement);
      state.selectedElementId = newElement.id;
    },
    selectElement: (state, action: PayloadAction<string | null>) => {
      state.selectedElementId = action.payload;
    },
    updateElement: (
      state,
      action: PayloadAction<{ id: string; changes: Partial<CanvasElement> }>
    ) => {
      const { id, changes } = action.payload;
      const element = state.elements.find((el) => el.id === id);
      if (element) {
        Object.assign(element, changes);
      }
    },
    // New reducer to toggle visibility
    toggleElementVisibility: (state, action: PayloadAction<string>) => {
      const element = state.elements.find((el) => el.id === action.payload);
      if (element) {
        element.isVisible = !element.isVisible;
      }
    },
    // New reducer to toggle lock state
    toggleElementLock: (state, action: PayloadAction<string>) => {
      const element = state.elements.find((el) => el.id === action.payload);
      if (element) {
        element.isLocked = !element.isLocked;
      }
    },
    // New reducer to reorder elements
    reorderElements: (state, action: PayloadAction<{ dragIndex: number; hoverIndex: number }>) => {
      const { dragIndex, hoverIndex } = action.payload;
      const [removed] = state.elements.splice(dragIndex, 1);
      state.elements.splice(hoverIndex, 0, removed);
    },
  },
});

export const {
  addElement,
  selectElement,
  updateElement,
  toggleElementVisibility,
  toggleElementLock,
  reorderElements, // Export the new action
} = editorSlice.actions;

// ... (selectors remain the same)
export const selectElements = (state: RootState) => state.editor.elements;
export const selectSelectedElementId = (state: RootState) => state.editor.selectedElementId;

export default editorSlice.reducer;
```

**Key Changes:**
*   Added `isVisible` and `isLocked` to `BaseElement`.
*   Updated `addElement` to set default values for these new properties.
*   Added `toggleElementVisibility` and `toggleElementLock` reducers.
*   Added `reorderElements` reducer, which will be used for our drag-and-drop functionality.

#### **2. Build the UI Components**

We'll replace the placeholder `PropertiesPanel` with a new version that has tabs and contains our `LayersPanel`.

**Create some icons first in `src/assets/icons/`:**

**`src/assets/icons/EyeIcon.tsx`:**
```tsx
const EyeIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" {...props}><path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"></path><circle cx="12" cy="12" r="3"></circle></svg>
);
export default EyeIcon;
```

**`src/assets/icons/LockIcon.tsx`:**
```tsx
const LockIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" {...props}><rect x="3" y="11" width="18" height="11" rx="2" ry="2"></rect><path d="M7 11V7a5 5 0 0 1 10 0v4"></path></svg>
);
export default LockIcon;
```

**Create `src/features/editor/layout/LayersPanel.tsx`:**

```tsx
import React from 'react';
import { useSelector, useDispatch } from 'react-redux';
import {
  selectElements,
  selectSelectedElementId,
  selectElement,
  toggleElementVisibility,
  toggleElementLock,
  reorderElements,
} from '../editorSlice';
import EyeIcon from '../../../assets/icons/EyeIcon';
import LockIcon from '../../../assets/icons/LockIcon';

const LayersPanel = () => {
  const dispatch = useDispatch();
  const elements = useSelector(selectElements);
  const selectedElementId = useSelector(selectSelectedElementId);
  
  // We reverse the array for display so the top layer is at the top of the list
  const reversedElements = [...elements].reverse();

  const handleDragStart = (e: React.DragEvent, index: number) => {
    const originalIndex = elements.length - 1 - index;
    e.dataTransfer.setData('text/plain', originalIndex.toString());
  };

  const handleDrop = (e: React.DragEvent, index: number) => {
    e.preventDefault();
    const dragIndex = parseInt(e.dataTransfer.getData('text/plain'), 10);
    const hoverIndex = elements.length - 1 - index;
    if (dragIndex !== hoverIndex) {
      dispatch(reorderElements({ dragIndex, hoverIndex }));
    }
  };

  return (
    <div className="flex flex-col h-full">
      <h3 className="text-lg font-semibold p-4 border-b border-gray-200">Layers</h3>
      <ul className="flex-1 overflow-y-auto">
        {reversedElements.map((element, index) => (
          <li
            key={element.id}
            draggable
            onDragStart={(e) => handleDragStart(e, index)}
            onDragOver={(e) => e.preventDefault()}
            onDrop={(e) => handleDrop(e, index)}
            onClick={() => dispatch(selectElement(element.id))}
            className={`flex items-center p-2 border-b border-gray-200 cursor-pointer hover:bg-gray-100 ${
              selectedElementId === element.id ? 'bg-blue-100' : ''
            }`}
          >
            <span className="flex-1 truncate">{element.type === 'text' ? element.text : 'Shape'}</span>
            <div className="flex items-center space-x-2">
              <button
                onClick={(e) => {
                  e.stopPropagation();
                  dispatch(toggleElementLock(element.id));
                }}
                className={`p-1 rounded ${element.isLocked ? 'text-blue-600' : 'text-gray-400'}`}
              >
                <LockIcon />
              </button>
              <button
                onClick={(e) => {
                  e.stopPropagation();
                  dispatch(toggleElementVisibility(element.id));
                }}
                className={`p-1 rounded ${element.isVisible ? 'text-blue-600' : 'text-gray-400'}`}
              >
                <EyeIcon />
              </button>
            </div>
          </li>
        ))}
      </ul>
    </div>
  );
};

export default LayersPanel;
```

**Now, update `src/features/editor/layout/PropertiesPanel.tsx` to include tabs:**

```tsx
import React, { useState } from 'react';
import LayersPanel from './LayersPanel';

type ActiveTab = 'properties' | 'layers';

const PropertiesPanel = () => {
  const [activeTab, setActiveTab] = useState<ActiveTab>('layers');

  return (
    <aside className="w-72 bg-white border-l border-gray-300 flex flex-col">
      <div className="flex border-b border-gray-200">
        <button
          onClick={() => setActiveTab('properties')}
          className={`flex-1 p-2 text-sm font-medium ${
            activeTab === 'properties' ? 'bg-gray-200 text-gray-800' : 'text-gray-500 hover:bg-gray-100'
          }`}
        >
          Properties
        </button>
        <button
          onClick={() => setActiveTab('layers')}
          className={`flex-1 p-2 text-sm font-medium ${
            activeTab === 'layers' ? 'bg-gray-200 text-gray-800' : 'text-gray-500 hover:bg-gray-100'
          }`}
        >
          Layers
        </button>
      </div>
      <div className="flex-1 overflow-hidden">
        {activeTab === 'properties' && (
          <div className="p-4">Properties content will go here...</div>
        )}
        {activeTab === 'layers' && <LayersPanel />}
      </div>
    </aside>
  );
};

export default PropertiesPanel;
```

#### **3. Integrate New Properties into the Canvas**

Finally, let's make the canvas respect the `isVisible` and `isLocked` properties.

**Update `src/features/editor/canvas/CanvasArea.tsx`:**

```tsx
// ... (imports remain the same)

// ... (CanvasArea component starts)

  useEffect(() => {
    const transformer = transformerRef.current;
    const stage = stageRef.current;
    
    // Find the currently selected element from the state
    const selectedElement = elements.find(el => el.id === selectedElementId);

    if (transformer && stage) {
      // Attach to the selected node ONLY if it exists and is NOT locked
      if (selectedElementId && selectedElement && !selectedElement.isLocked) {
        const selectedNode = stage.findOne('#' + selectedElementId);
        if (selectedNode) {
          transformer.nodes([selectedNode]);
        } else {
          transformer.nodes([]);
        }
      } else {
        transformer.nodes([]);
      }
      transformer.getLayer()?.batchDraw();
    }
  }, [selectedElementId, elements]); // Add `elements` to the dependency array

  // ... (handleStageClick remains the same)

  return (
    // ...
        <Stage ...>
          <Layer>
            {elements.map((element) => {
              if (element.type === 'text') {
                return (
                  <Text
                    key={element.id}
                    id={element.id}
                    {...element}
                    draggable={!element.isLocked} // Draggable only if not locked
                    visible={element.isVisible}   // Use the isVisible prop
                    onClick={(e) => {
                      if (!element.isLocked) {
                        dispatch(selectElement(element.id));
                      }
                      e.cancelBubble = true; // Prevent stage click from firing
                    }}
                    onTap={() => { if (!element.isLocked) dispatch(selectElement(element.id)) }}
                    // ... (onDragEnd and onTransformEnd handlers remain the same)
                  />
                );
              }
              return null;
            })}
            <Transformer ... />
          </Layer>
        </Stage>
    // ...
  );
};

export default CanvasArea;
```

**Key Changes:**
*   **Draggable & Visible Props:** We pass `draggable={!element.isLocked}` and `visible={element.isVisible}` to the Konva `<Text>` component. Konva will now handle showing/hiding and enabling/disabling dragging for us.
*   **Transformer Logic:** The `useEffect` now checks if the selected element is locked. If it is, it will not attach the transformer, preventing resizing and rotation. We also added `elements` to the dependency array so the effect re-runs when a property like `isLocked` changes.
*   **Click Handling:** The `onClick` handler now checks `!element.isLocked` before selecting, so locked elements cannot be selected via the canvas.

---

### **Your Turn**

1.  Update the files as described: `editorSlice.ts`, `PropertiesPanel.tsx`, `CanvasArea.tsx`, and create `LayersPanel.tsx` and the two new icon files.
2.  **Test the full Layers workflow:**
    *   Add a few text elements. You should see them appear in the Layers panel on the right (top element in the list is the one on top on the canvas).
    *   Click on a layer in the panel. The corresponding element on the canvas should be selected.
    *   Click the "eye" icon to hide an element. It should disappear from the canvas but remain in the layer list. Click again to show it.
    *   Click the "lock" icon. You should no longer be able to select, move, or resize that element on the canvas. The transformer handles should disappear. Unlock it to restore functionality.
    *   Drag and drop a layer in the list to reorder it. The stacking order of the elements on the canvas should change accordingly.

You now have a powerful and essential feature for any design application. What would you like to tackle next? We could add Shapes, Images, or build out the "Properties" tab for the placeholder logic.
