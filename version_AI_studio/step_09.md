**USER**

yes , im ok but make this as a flyout element , so that we will stick the User experience across all Text , shape & images

**RESPONSE**

You're absolutely right. That's a much better UX design. Creating a flyout panel for Shapes not only looks more professional but also creates a scalable pattern that we can apply to Text (e.g., Heading, Subheading, Paragraph) and Images (e.g., Upload, Stock Photos) later on.

Excellent suggestion. Let's refactor and build it that way.

The core logic for adding a shape to Redux and rendering it on the canvas will be the same, but we'll wrap the trigger action inside a new UI component.

---

### **Phase 1, Step 9 (step_08 Revised): Adding Shapes via a Flyout Menu**

#### **1. Create a Reusable "Click Outside" Hook**

This is a classic UI problem: how to close a menu when the user clicks anywhere else. A custom hook is the perfect, reusable solution.

**Create a new file: `src/hooks/useOnClickOutside.ts`**
(You'll need to create the `src/hooks` directory).

```typescript
import { useEffect, RefObject } from 'react';

type Event = MouseEvent | TouchEvent;

export const useOnClickOutside = <T extends HTMLElement = HTMLElement>(
  ref: RefObject<T>,
  handler: (event: Event) => void
) => {
  useEffect(() => {
    const listener = (event: Event) => {
      const el = ref?.current;
      // Do nothing if clicking ref's element or descendent elements
      if (!el || el.contains((event?.target as Node) || null)) {
        return;
      }
      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);

    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]); // Re-run if ref or handler changes
};
```

#### **2. Refactor the `LeftToolbar` to Manage and Display the Flyout**

This is the biggest change. The toolbar will now manage which flyout (if any) is currently visible.

**First, create the new icon: `src/assets/icons/SquareIcon.tsx`**

```tsx
const SquareIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" {...props}><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect></svg>
);
export default SquareIcon;
```

**Now, update `src/features/editor/layout/LeftToolbar.tsx`:**

```tsx
import React, { useState, useRef } from 'react';
import { useDispatch } from 'react-redux';
import { addElement, ShapeElement } from '../editorSlice';
import { useOnClickOutside } from '../../../hooks/useOnClickOutside'; // Import the hook
import TextIcon from '../../../assets/icons/TextIcon';
import SquareIcon from '../../../assets/icons/SquareIcon';

type ActiveFlyout = 'text' | 'shapes' | null;

const LeftToolbar = () => {
  const dispatch = useDispatch();
  const [activeFlyout, setActiveFlyout] = useState<ActiveFlyout>(null);
  const toolbarRef = useRef<HTMLDivElement>(null);

  // Close flyout when clicking outside
  useOnClickOutside(toolbarRef, () => setActiveFlyout(null));

  const handleToolbarClick = (flyout: ActiveFlyout) => {
    setActiveFlyout(prev => (prev === flyout ? null : flyout));
  };

  const handleAddRectangle = () => {
    const defaultRectElement: Omit<ShapeElement, 'id' | 'isVisible' | 'isLocked'> = {
      type: 'shape', shapeType: 'rect', x: 200, y: 200, width: 100, height: 100,
      rotation: 0, fill: '#cccccc', stroke: '#000000', strokeWidth: 1,
    };
    dispatch(addElement(defaultRectElement));
    setActiveFlyout(null); // Close flyout after adding element
  };

  return (
    <div ref={toolbarRef} className="relative"> {/* Positioning context for the flyout */}
      <aside className="w-20 bg-white border-r border-gray-300 h-full p-2 flex flex-col items-center space-y-4">
        {/* Text Button (will be a flyout later) */}
        <button
          className="flex flex-col items-center p-2 rounded-lg hover:bg-gray-200 transition-colors"
          title="Add Text"
        >
          <TextIcon className="w-8 h-8 text-gray-700" />
          <span className="text-xs mt-1">Text</span>
        </button>

        {/* Shape Button (now a flyout toggle) */}
        <button
          onClick={() => handleToolbarClick('shapes')}
          className={`flex flex-col items-center p-2 rounded-lg transition-colors ${
            activeFlyout === 'shapes' ? 'bg-gray-200' : 'hover:bg-gray-200'
          }`}
          title="Add Shape"
        >
          <SquareIcon className="w-8 h-8 text-gray-700" />
          <span className="text-xs mt-1">Shapes</span>
        </button>
      </aside>

      {/* Shapes Flyout Panel */}
      {activeFlyout === 'shapes' && (
        <div className="absolute left-full top-0 ml-2 w-60 bg-white rounded-lg shadow-lg border border-gray-200 p-2">
          <h4 className="font-semibold text-gray-800 px-2 pb-2">Shapes</h4>
          <div className="grid grid-cols-3 gap-2">
            <button 
              onClick={handleAddRectangle}
              className="flex flex-col items-center justify-center p-2 rounded-md hover:bg-gray-100"
            >
              <div className="w-10 h-10 bg-gray-300 border border-gray-400"></div>
              <span className="text-xs mt-1">Rectangle</span>
            </button>
            {/* We will add Circle, Line, etc. here later */}
          </div>
        </div>
      )}
    </div>
  );
};

export default LeftToolbar;
```

#### **3. Redux, Canvas & Properties Logic (Same as Before)**

The good news is that the rest of our plan remains unchanged. We just changed *how* the `addElement` action is dispatched.

*   **Redux Slice:** Your `editorSlice.ts` should be updated exactly as described in the previous message (adding `ShapeElement`, updating the `CanvasElement` union).
*   **Canvas Rendering:** Your `CanvasArea.tsx` needs the new `if (element.type === 'shape')` block to render the `<Rect>` component.
*   **Properties Panel:** Your `PropertiesTab.tsx` needs the new `if (selectedElement.type === 'shape')` block to show the fill/stroke color pickers.

Please make sure you have also completed the changes from the previous response for those three files.

---

### **Your Turn**

1.  Create the `src/hooks` directory and the `useOnClickOutside.ts` hook file.
2.  Create the `SquareIcon.tsx` file.
3.  Completely replace your `LeftToolbar.tsx` with the new version above.
4.  Ensure `editorSlice.ts`, `CanvasArea.tsx`, and `PropertiesTab.tsx` are updated as per the previous instructions.
5.  **Test the new flyout workflow:**
    *   Click the "Shapes" icon in the left toolbar.
    *   A flyout panel should appear to the right of the toolbar.
    *   Click the "Rectangle" button inside the flyout.
    *   A rectangle should be added to the canvas, and the flyout should automatically close.
    *   Open the flyout again, then click anywhere else on the page (the canvas, the top header). The flyout should close.
    *   All previous functionality (selecting, resizing, changing colors in the Properties tab) should still work perfectly for the new shape.

This new pattern is much more robust. Now that we have this powerful flyout system, adding more shapes (like a circle) will be incredibly easy.
