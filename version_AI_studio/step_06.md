**USER**

What would you like to tackle next? We could add Shapes, Images, or build out the "Properties" tab for the placeholder logic.

**RESPONSE**

Excellent. Let's build out the "Properties" tab and implement the core "Static vs. Placeholder" functionality. This is a critical feature that turns our design tool into a powerful templating engine.

Our goal is to create the UI in the right-hand panel that allows a designer to switch a text element between being static content and a dynamic placeholder.

---

### **Phase 1, Step_06: Implementing the Placeholder Properties Panel**

#### **1. Update the Redux Slice (`editorSlice.ts`)**

We need to add the placeholder properties to our `TextElement` type.

**Update `src/features/editor/editorSlice.ts`:**

```typescript
// ... (imports and BaseElement interface remain the same)

export interface TextElement extends BaseElement {
  type: 'text';
  text: string; // This will hold the static text OR the placeholder key
  fontSize: number;
  fontFamily: string;
  fill: string;
  isPlaceholder: boolean; // Is this element a placeholder?
  placeholderKey: string; // The key for the placeholder, e.g., "user_name"
}

// ... (CanvasElement, EditorState, initialState remain the same)

const editorSlice = createSlice({
  name: 'editor',
  initialState,
  reducers: {
    addElement: (state, action: PayloadAction<Omit<TextElement, 'id' | 'isVisible' | 'isLocked' | 'isPlaceholder' | 'placeholderKey'>>) => {
      const newElement: TextElement = {
        id: nanoid(),
        isVisible: true,
        isLocked: false,
        isPlaceholder: false, // Default to static content
        placeholderKey: 'placeholder_key', // Default key
        ...(action.payload as TextElement),
      };
      state.elements.push(newElement);
      state.selectedElementId = newElement.id;
    },
    // ... (selectElement, updateElement, toggles, reorder all remain the same)
    // Our generic updateElement reducer is already perfect for this!
  },
});

// ... (actions and selectors remain the same)
```

**Key Changes:**
*   We added `isPlaceholder` (boolean) and `placeholderKey` (string) to the `TextElement` interface.
*   In `addElement`, we now set the default state for a new text element to be `isPlaceholder: false`.

#### **2. Create the Properties Tab UI**

This new component will contain the logic for editing the properties of the selected element.

**Create `src/features/editor/layout/PropertiesTab.tsx`:**

```tsx
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

#### **3. Integrate the New Tab**

Now, let's plug this component into our main `PropertiesPanel`.

**Update `src/features/editor/layout/PropertiesPanel.tsx`:**

```tsx
import React, { useState } from 'react';
import LayersPanel from './LayersPanel';
import PropertiesTab from './PropertiesTab'; // Import the new component

type ActiveTab = 'properties' | 'layers';

const PropertiesPanel = () => {
  const [activeTab, setActiveTab] = useState<ActiveTab>('properties'); // Default to properties now

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
      <div className="flex-1 overflow-y-auto"> {/* Add overflow-y-auto here */}
        {activeTab === 'properties' && <PropertiesTab />}
        {activeTab === 'layers' && <LayersPanel />}
      </div>
    </aside>
  );
};

export default PropertiesPanel;
```

#### **4. Visually Update the Canvas**

To make it obvious to the designer which elements are placeholders, let's change what the canvas displays.

**Update `src/features/editor/canvas/CanvasArea.tsx`:**

```tsx
// ... inside the .map() function in CanvasArea's return statement

if (element.type === 'text') {
  // Determine what text to display on the canvas
  const displayText = element.isPlaceholder
    ? `{{${element.placeholderKey}}}` // e.g., "{{user_name}}"
    : element.text;

  return (
    <Text
      key={element.id}
      id={element.id}
      {...element}
      text={displayText} // Use the displayText variable here
      // ... rest of the props (draggable, visible, onClick, etc.) remain the same
    />
  );
}
```

**Key Change:**
*   We've added a simple logic check. If the element `isPlaceholder`, we render its `placeholderKey` wrapped in curly braces. Otherwise, we render its normal `text` property. This provides immediate visual feedback on the canvas.

---

### **Your Turn**

1.  Update the four files as described above.
2.  **Test the placeholder workflow:**
    *   Add a text element. It should be selected, and the "Properties" tab should be active by default.
    *   The "Content Type" should be "Static". Change the text in the textarea and see it update on the canvas.
    *   Click the "Placeholder" button. The UI should switch to the "Parameter Name" input.
    *   Type a key like `customer_name` into the input. The text on the canvas should immediately change to `{{customer_name}}`.
    *   Switch back to "Static". The canvas should revert to showing the static text.
    *   The functionality of the contextual toolbar (changing font size, color) should still work in both modes.

You have now successfully implemented the core data-templating logic of your application! This is a massive step. 
