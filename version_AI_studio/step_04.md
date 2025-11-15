**USER**

our next step will be to make the text element on the canvas selectable and movable.

**RESPONSE**

Of course. This is where the application starts to feel like a real editor. Let's make it happen.

We'll need to update our Redux state to track which element is selected, and then use that information in our canvas to show a transformer and handle dragging.

---

### **Phase 1, Step 3: Making Elements Selectable & Movable**

#### **1. Update the Redux Slice (`editorSlice.ts`)**

We need actions to:
a) Set the `selectedElementId` when an element is clicked.
b) Update an element's properties (like its `x` and `y` position) after it's been moved.

**Update `src/features/editor/editorSlice.ts`:**

```typescript
import { createSlice, PayloadAction, nanoid } from '@reduxjs/toolkit';
import { RootState } from '../../app/store';

// ... (Keep the interface definitions from the previous step)
export interface BaseElement {
  id: string;
  x: number;
  y: number;
  width: number;
  height: number;
  rotation: number;
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
    addElement: (state, action: PayloadAction<Omit<CanvasElement, 'id'>>) => {
      const newElement = {
        id: nanoid(),
        ...action.payload,
      };
      state.elements.push(newElement as CanvasElement);
      // Automatically select the new element
      state.selectedElementId = newElement.id;
    },
    // Action to set the selected element
    selectElement: (state, action: PayloadAction<string | null>) => {
      state.selectedElementId = action.payload;
    },
    // Action to update an element's properties
    updateElement: (
      state,
      action: PayloadAction<{ id: string; changes: Partial<CanvasElement> }>
    ) => {
      const { id, changes } = action.payload;
      const element = state.elements.find((el) => el.id === id);
      if (element) {
        // Apply the changes to the element
        Object.assign(element, changes);
      }
    },
  },
});

export const { addElement, selectElement, updateElement } = editorSlice.actions;

export const selectElements = (state: RootState) => state.editor.elements;
// New selector for the selected element ID
export const selectSelectedElementId = (state: RootState) => state.editor.selectedElementId;

export default editorSlice.reducer;
```

**Key Changes:**
*   **`selectElement`**: A simple reducer to update `selectedElementId`. We'll call this when we click an element. Passing `null` will deselect everything.
*   **`updateElement`**: A flexible reducer that finds an element by its ID and merges in new properties. We'll use this to save the new `x` and `y` coordinates after a drag.
*   **Modified `addElement`**: When a new element is created, it's now automatically selected. This is much better UX.
*   **`selectSelectedElementId`**: A new selector to easily get the ID of the selected element.

#### **2. Update the Canvas (`CanvasArea.tsx`)**

This is where we'll use our new actions and add the Konva `Transformer`.

**Update `src/features/editor/canvas/CanvasArea.tsx`:**

```tsx
import React, { useEffect, useRef } from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { Stage, Layer, Text, Transformer } from 'react-konva';
import Konva from 'konva';
import {
  selectElements,
  selectSelectedElementId,
  selectElement,
  updateElement,
  CanvasElement,
} from '../editorSlice';

const A4_WIDTH = 595;
const A4_HEIGHT = 842;

const CanvasArea = () => {
  const dispatch = useDispatch();
  const elements = useSelector(selectElements);
  const selectedElementId = useSelector(selectSelectedElementId);
  
  // Refs for Konva nodes
  const stageRef = useRef<Konva.Stage>(null);
  const transformerRef = useRef<Konva.Transformer>(null);

  useEffect(() => {
    // This effect handles attaching the transformer to the selected node
    const transformer = transformerRef.current;
    const stage = stageRef.current;
    
    if (transformer && stage) {
      if (selectedElementId) {
        // Find the Konva node corresponding to the selected element
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
  }, [selectedElementId]); // Rerun this effect when the selection changes

  const handleStageClick = (e: Konva.KonvaEventObject<MouseEvent>) => {
    // Deselect if the click is on the stage itself (not a shape)
    if (e.target === e.target.getStage()) {
      dispatch(selectElement(null));
    }
  };

  return (
    <div className="w-full h-full flex items-center justify-center">
      <div className="bg-white shadow-lg">
        <Stage
          ref={stageRef}
          width={A4_WIDTH}
          height={A4_HEIGHT}
          onClick={handleStageClick}
        >
          <Layer>
            {elements.map((element) => {
              if (element.type === 'text') {
                return (
                  <Text
                    key={element.id}
                    id={element.id} // Important for finding the node
                    {...element}
                    draggable
                    onClick={() => dispatch(selectElement(element.id))}
                    onTap={() => dispatch(selectElement(element.id))}
                    onDragEnd={(e) => {
                      dispatch(
                        updateElement({
                          id: element.id,
                          changes: { x: e.target.x(), y: e.target.y() },
                        })
                      );
                    }}
                  />
                );
              }
              return null;
            })}
            {/* The Transformer is rendered here, but attached via the effect */}
            <Transformer
              ref={transformerRef}
              boundBoxFunc={(oldBox, newBox) => {
                // Limit the size of the transformer for text
                if (newBox.width < 5 || newBox.height < 5) {
                  return oldBox;
                }
                return newBox;
              }}
            />
          </Layer>
        </Stage>
      </div>
    </div>
  );
};

export default CanvasArea;
```

**Key Changes & Explanation:**
1.  **Refs:** We create refs (`stageRef`, `transformerRef`) to get direct access to the Konva `Stage` and `Transformer` instances.
2.  **`useEffect` for Transformer:** This is the core logic. When `selectedElementId` changes, this effect runs. It finds the actual Konva node on the stage using its `id` (`stage.findOne('#' + selectedElementId)`) and tells the transformer to attach to it. If nothing is selected, it detaches from everything.
3.  **Deselecting:** The `handleStageClick` function detects a click on the blank area of the stage and dispatches `selectElement(null)` to clear the selection.
4.  **Updated `<Text>` Component Props:**
    *   **`id={element.id}`**: This is crucial. It gives the Konva node an ID that we can use to find it from the stage.
    *   **`draggable`**: We enable dragging on all text elements.
    *   **`onClick` / `onTap`**: When a text element is clicked, it dispatches `selectElement` with its own ID, setting it as the selected element in our Redux store.
    *   **`onDragEnd`**: This event fires *after* the user finishes dragging. We get the final `x` and `y` from the event object (`e.target`) and dispatch `updateElement` to save this new position to our Redux store. **This is what makes the new position "stick" permanently.**
5.  **`<Transformer />`**: We add the transformer component to our layer. It will be invisible until the `useEffect` attaches it to a node.

---

### **Your Turn**

1.  Update the two files (`editorSlice.ts` and `CanvasArea.tsx`).
2.  Your app will hot-reload.
3.  **Test the new functionality:**
    *   Add a new text element. It should appear with the transformer handles around it (because it's selected by default).
    *   Click on the blank canvas. The handles should disappear.
    *   Click on the text again. The handles should reappear.
    *   Click and drag the text to a new position. Let go. The text should stay in the new position.
    *   Add a second text element. You should be able to select between them, and only the selected one will have the handles.

You now have a truly interactive canvas! Once this is working, our next step will be to make the `PropertiesPanel` on the right side come to life, allowing you to edit the text content and style of the selected element.
