**USER**

Next , lets revalidate the Placeholder Functionality !!!

**RESPONSE**

Fantastic! Let's dive in and get this implemented.

Here are the step-by-step instructions.

---

### **Phase 1, Step_07: Placeholder Functionality**

### **Step 1: Update Redux State (`editorSlice.ts`)**

First, we'll modify the `TextElement` type to include the new placeholder fields and set their default values when a new element is created.

**File to edit: `src/features/editor/editorSlice.ts`**

```typescript
// src/features/editor/editorSlice.ts

import { createSlice, PayloadAction, nanoid } from '@reduxjs/toolkit';
import { RootState } from '../../app/store';

// Keep BaseElement as is
export interface BaseElement {
  id: string;
  x: number;
  y: number;
  width?: number;
  height?: number;
  rotation: number;
  isVisible: boolean;
  isLocked: boolean;
}

// UPDATE TextElement interface
export interface TextElement extends BaseElement {
  type: 'text';
  text: string;
  fontSize: number;
  fontFamily: string;
  fill: string;
  isPlaceholder: boolean; // ADD THIS
  placeholderKey: string;  // ADD THIS
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
    // UPDATE addElement reducer
    addElement: (state, action: PayloadAction<Omit<TextElement, 'id' | 'isVisible' | 'isLocked' | 'isPlaceholder' | 'placeholderKey'>>) => {
      const newElement: TextElement = {
        id: nanoid(),
        isVisible: true,
        isLocked: false,
        isPlaceholder: false, // Default to static
        placeholderKey: 'placeholder_key', // Default key name
        ...(action.payload as TextElement),
      };
      state.elements.push(newElement);
      state.selectedElementId = newElement.id;
    },
    // No other changes needed for the other reducers
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
    toggleElementVisibility: (state, action: PayloadAction<string>) => {
      const element = state.elements.find((el) => el.id === action.payload);
      if (element) {
        element.isVisible = !element.isVisible;
      }
    },
    toggleElementLock: (state, action: PayloadAction<string>) => {
      const element = state.elements.find((el) => el.id === action.payload);
      if (element) {
        element.isLocked = !element.isLocked;
      }
    },
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
  reorderElements,
} = editorSlice.actions;

export const selectElements = (state: RootState) => state.editor.elements;
export const selectSelectedElementId = (state: RootState) => state.editor.selectedElementId;

export default editorSlice.reducer;
```

---

### **Step 2: Create the Properties Tab UI Component**

This is a new file that will live alongside your other layout components.

**File to create: `src/features/editor/layout/PropertiesTab.tsx`**

```tsx
// src/features/editor/layout/PropertiesTab.tsx

import React from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { RootState } from '../../../app/store';
import { updateElement, TextElement } from '../editorSlice';

const PropertiesTab = () => {
  const dispatch = useDispatch();
  const { elements, selectedElementId } = useSelector((state: RootState) => state.editor);

  if (!selectedElementId) {
    return <div className="p-4 text-gray-500">Select an element to see its properties.</div>;
  }

  const selectedElement = elements.find(el => el.id === selectedElementId);

  if (!selectedElement) {
    return <div className="p-4 text-gray-500">Element not found.</div>;
  }

  // Helper function to dispatch updates
  const handleUpdate = (changes: Partial<TextElement>) => {
    dispatch(updateElement({ id: selectedElementId, changes }));
  };

  // Render specific properties for a Text Element
  if (selectedElement.type === 'text') {
    const textElement = selectedElement as TextElement;
    return (
      <div className="p-4 space-y-4">
        <h3 className="text-lg font-semibold border-b pb-2">Text Properties</h3>
        
        {/* Content Type Toggle */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Content Type</label>
          <div className="flex rounded-md shadow-sm">
            <button
              onClick={() => handleUpdate({ isPlaceholder: false })}
              className={`flex-1 p-2 text-sm rounded-l-md ${
                !textElement.isPlaceholder ? 'bg-blue-600 text-white' : 'bg-white text-gray-700 hover:bg-gray-50'
              }`}
            >
              Static
            </button>
            <button
              onClick={() => handleUpdate({ isPlaceholder: true })}
              className={`flex-1 p-2 text-sm rounded-r-md border-l border-gray-200 ${
                textElement.isPlaceholder ? 'bg-blue-600 text-white' : 'bg-white text-gray-700 hover:bg-gray-50'
              }`}
            >
              Placeholder
            </button>
          </div>
        </div>

        {/* Conditional Input Field */}
        {textElement.isPlaceholder ? (
          <div>
            <label htmlFor="placeholderKey" className="block text-sm font-medium text-gray-700">
              Parameter Name
            </label>
            <input
              type="text"
              id="placeholderKey"
              value={textElement.placeholderKey}
              onChange={(e) => handleUpdate({ placeholderKey: e.target.value })}
              className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 sm:text-sm"
            />
          </div>
        ) : (
          <div>
            <label htmlFor="staticText" className="block text-sm font-medium text-gray-700">
              Text
            </label>
            <textarea
              id="staticText"
              rows={3}
              value={textElement.text}
              onChange={(e) => handleUpdate({ text: e.target.value })}
              className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500 sm:text-sm"
            />
          </div>
        )}
      </div>
    );
  }

  return <div className="p-4">Properties for this element type are not yet implemented.</div>;
};

export default PropertiesTab;
```

---

### **Step 3: Integrate the New Tab into the Sidebar**

We'll update the main sidebar component to import and render our new `PropertiesTab`.

**File to edit: `src/features/editor/layout/PropertiesPanel.tsx`**

```tsx
// src/features/editor/layout/PropertiesPanel.tsx

import React, { useState } from 'react';
import LayersPanel from './LayersPanel';
import PropertiesTab from './PropertiesTab'; // IMPORT THIS

type ActiveTab = 'properties' | 'layers';

const PropertiesPanel = () => {
  const [activeTab, setActiveTab] = useState<ActiveTab>('properties'); // UPDATE DEFAULT TAB

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
      <div className="flex-1 overflow-y-auto"> {/* ADD overflow-y-auto */}
        {activeTab === 'properties' && <PropertiesTab />} {/* UPDATE THIS LINE */}
        {activeTab === 'layers' && <LayersPanel />}
      </div>
    </aside>
  );
};

export default PropertiesPanel;
```

---

### **Step 4: Update the Canvas to Show Placeholders**

Finally, let's make the canvas visually represent the placeholders.

**File to edit: `src/features/editor/canvas/CanvasArea.tsx`**

```tsx
// src/features/editor/canvas/CanvasArea.tsx

// ... (imports and component setup as before)

// Find this section within the return statement of CanvasArea.tsx
// ...
          <Layer>
            {elements.map((element) => {
              // UPDATE THIS BLOCK
              if (element.type === 'text') {
                const displayText = element.isPlaceholder
                  ? `{{${element.placeholderKey}}}`
                  : element.text;

                return (
                  <Text
                    key={element.id}
                    id={element.id}
                    {...element}
                    text={displayText} // Use the new variable here
                    draggable={!element.isLocked}
                    visible={element.isVisible}
                    onClick={(e) => {
                      if (!element.isLocked) {
                        dispatch(selectElement(element.id));
                      }
                      e.cancelBubble = true;
                    }}
                    onTap={() => { if (!element.isLocked) dispatch(selectElement(element.id)) }}
                    onDragEnd={(e) => {
                      dispatch(
                        updateElement({
                          id: element.id,
                          changes: { x: e.target.x(), y: e.target.y() },
                        })
                      );
                    }}
                    onTransformEnd={(e) => {
                      const node = e.target;
                      const scaleX = node.scaleX();
                      const scaleY = node.scaleY();
                      node.scaleX(1);
                      node.scaleY(1);
                      dispatch(
                        updateElement({
                          id: element.id,
                          changes: {
                            x: node.x(),
                            y: node.y(),
                            width: Math.max(5, node.width() * scaleX),
                            height: Math.max(node.height() * scaleY),
                            rotation: node.rotation(),
                          },
                        })
                      );
                    }}
                  />
                );
              }
              // ...
              return null;
            })}
            <Transformer
              ref={transformerRef}
              // ... (rest of the transformer props)
            />
          </Layer>
// ... (rest of the component)
```

---

Once you've made these changes, your application should be fully functional with the new placeholder feature. Let me know when you're ready for the next step
