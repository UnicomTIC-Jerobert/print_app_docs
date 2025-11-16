**USER**

sure im ready for the next step , lets proceed !!!

**RESPONSE**

Excellent! Let's add the next fundamental element to our editor: **Shapes**.

We'll start by adding a simple Rectangle. This process will beautifully demonstrate the power of our current architecture, as we'll be able to reuse a lot of the logic we've already built for text.

---

### **Phase 1, Step_08: Adding Shape Elements**

#### **1. Update the Redux Slice (`editorSlice.ts`)**

We need a new type for our shape elements and need to update our `CanvasElement` union type to include it.

**Update `src/features/editor/editorSlice.ts`:**

```typescript
// ... (imports remain the same)

// BaseElement and TextElement interfaces remain the same
export interface BaseElement { /* ... */ }
export interface TextElement extends BaseElement { /* ... */ }

// ADD the new ShapeElement interface
export interface ShapeElement extends BaseElement {
  type: 'shape';
  shapeType: 'rect'; // We'll start with just rectangles
  fill: string;
  stroke: string;
  strokeWidth: number;
}

// UPDATE the CanvasElement discriminated union
export type CanvasElement = TextElement | ShapeElement;

// ... (EditorState and initialState remain the same)

const editorSlice = createSlice({
  name: 'editor',
  initialState,
  reducers: {
    // UPDATE the addElement action's payload type to be more generic
    addElement: (state, action: PayloadAction<Omit<CanvasElement, 'id' | 'isVisible' | 'isLocked'>>) => {
      const newElement = {
        id: nanoid(),
        isVisible: true,
        isLocked: false,
        ...action.payload,
      };
      state.elements.push(newElement as CanvasElement);
      state.selectedElementId = newElement.id;
    },
    // No changes needed to our other reducers! They are already generic enough.
    // selectElement, updateElement, toggle..., reorder...
  },
});

// ... (actions and selectors remain the same)
```

**Key Changes:**
*   We defined a `ShapeElement` interface with shape-specific properties.
*   We updated `CanvasElement` to be a union of `TextElement | ShapeElement`.
*   We made our `addElement` action's payload more generic to accept any new element type. This is a great sign of a well-designed state structure!

#### **2. Update the `LeftToolbar` to Add Shapes**

Let's add a button to the toolbar to create a new rectangle.

**First, create a new icon: `src/assets/icons/SquareIcon.tsx`**

```tsx
const SquareIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" {...props}><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect></svg>
);
export default SquareIcon;
```

**Now, update `src/features/editor/layout/LeftToolbar.tsx`:**

```tsx
import { useDispatch } from 'react-redux';
import { addElement, TextElement, ShapeElement } from '../editorSlice';
import TextIcon from '../../../assets/icons/TextIcon';
import SquareIcon from '../../../assets/icons/SquareIcon'; // Import the new icon

const LeftToolbar = () => {
  const dispatch = useDispatch();

  const handleAddText = () => { /* ... (no changes here) */ };

  const handleAddRectangle = () => {
    // Define the properties of the default rectangle element
    const defaultRectElement: Omit<ShapeElement, 'id' | 'isVisible' | 'isLocked'> = {
      type: 'shape',
      shapeType: 'rect',
      x: 200,
      y: 200,
      width: 100,
      height: 100,
      rotation: 0,
      fill: '#cccccc',
      stroke: '#000000',
      strokeWidth: 1,
    };
    dispatch(addElement(defaultRectElement));
  };

  return (
    <aside className="w-20 bg-white border-r border-gray-300 p-2 flex flex-col items-center space-y-4">
      <button
        onClick={handleAddText}
        className="flex flex-col items-center p-2 rounded-lg hover:bg-gray-200 transition-colors"
        title="Add Text"
      >
        <TextIcon className="w-8 h-8 text-gray-700" />
        <span className="text-xs mt-1">Text</span>
      </button>
      
      {/* Add the new button for shapes */}
      <button
        onClick={handleAddRectangle}
        className="flex flex-col items-center p-2 rounded-lg hover:bg-gray-200 transition-colors"
        title="Add Rectangle"
      >
        <SquareIcon className="w-8 h-8 text-gray-700" />
        <span className="text-xs mt-1">Shape</span>
      </button>
    </aside>
  );
};

export default LeftToolbar;
```

#### **3. Render Shapes on the Canvas**

Now we'll teach our `CanvasArea` how to draw rectangles.

**Update `src/features/editor/canvas/CanvasArea.tsx`:**

```tsx
// Make sure to import Rect from react-konva
import { Stage, Layer, Text, Transformer, Rect } from 'react-konva';
// ... other imports

// ... inside the CanvasArea component's return statement
          <Layer>
            {elements.map((element) => {
              if (element.type === 'text') {
                // ... (Text rendering logic remains the same)
              }
              
              // ADD THIS BLOCK to render shapes
              if (element.type === 'shape' && element.shapeType === 'rect') {
                return (
                  <Rect
                    key={element.id}
                    id={element.id}
                    {...element}
                    draggable={!element.isLocked}
                    visible={element.isVisible}
                    onClick={(e) => {
                      if (!element.isLocked) dispatch(selectElement(element.id));
                      e.cancelBubble = true;
                    }}
                    onTap={() => { if (!element.isLocked) dispatch(selectElement(element.id)) }}
                    onDragEnd={(e) => { /* ... (same as Text) */ }}
                    onTransformEnd={(e) => { /* ... (same as Text) */ }}
                  />
                );
              }

              return null;
            })}
            <Transformer /* ... */ />
          </Layer>
// ...
```
**Note:** You can simply copy the `onDragEnd` and `onTransformEnd` handlers from the `<Text>` component. They are completely reusable for our shape!

#### **4. Add Properties to the `PropertiesTab`**

Finally, let's allow the user to edit the shape's colors.

**Update `src/features/editor/layout/PropertiesTab.tsx`:**

```tsx
// ... (imports and component start)

  // ... (code for when no element is selected or found)
  
  // Render properties for Text Element
  if (selectedElement.type === 'text') {
    // ... (no changes here, this block remains the same)
  }

  // ADD THIS BLOCK for Shape properties
  if (selectedElement.type === 'shape') {
    const shapeElement = selectedElement as ShapeElement;
    return (
      <div className="p-4 space-y-4">
        <h3 className="text-lg font-semibold border-b pb-2">Shape Properties</h3>
        
        {/* Fill Color */}
        <div className="flex justify-between items-center">
          <label htmlFor="fillColor" className="text-sm font-medium text-gray-700">Fill</label>
          <input
            type="color"
            id="fillColor"
            value={shapeElement.fill}
            onChange={(e) => handleUpdate({ fill: e.target.value })}
            className="h-8 w-12 p-0 border-none rounded-md cursor-pointer"
          />
        </div>

        {/* Stroke Color */}
        <div className="flex justify-between items-center">
          <label htmlFor="strokeColor" className="text-sm font-medium text-gray-700">Stroke</label>
          <input
            type="color"
            id="strokeColor"
            value={shapeElement.stroke}
            onChange={(e) => handleUpdate({ stroke: e.target.value })}
            className="h-8 w-12 p-0 border-none rounded-md cursor-pointer"
          />
        </div>
        
        {/* Stroke Width */}
        <div>
          <label htmlFor="strokeWidth" className="block text-sm font-medium text-gray-700">Stroke Width</label>
          <input
            type="number"
            id="strokeWidth"
            min="0"
            value={shapeElement.strokeWidth}
            onChange={(e) => handleUpdate({ strokeWidth: parseInt(e.target.value, 10) || 0 })}
            className="mt-1 block w-full rounded-md border-gray-300 shadow-sm sm:text-sm"
          />
        </div>
      </div>
    );
  }

  return <div className="p-4">Properties for this element type are not yet implemented.</div>;
```

---

### **Your Turn**

1.  Update the four specified files and create the new `SquareIcon.tsx`.
2.  **Test the full shape workflow:**
    *   Click the new "Shape" button in the left toolbar. A grey rectangle should appear on the canvas.
    *   The rectangle should be selected, draggable, and resizable with the transformer, just like the text.
    *   Select the rectangle and go to the "Properties" tab on the right.
    *   Use the color pickers to change the `Fill` and `Stroke` color. See the canvas update live.
    *   Change the `Stroke Width`.
    *   Check the "Layers" panel. You should be able to hide, lock, and reorder the shape layer just like a text layer.

You have now successfully extended your editor to support a completely new type of element. The next logical step would be to add Images, which introduces the concept of asset management. Are you ready to proceed with adding images?
