**USER**

 What would you like to build next? Shall we add images?

 **RESPONSE**

 Absolutely. Adding images is a big one, as it introduces file handling. We'll stick to the flyout pattern for a consistent and professional user experience.

The process will involve:
1.  Creating a new `ImageElement` type in Redux.
2.  Building an `ImageFlyout` with an upload button.
3.  Handling the image upload in the browser and creating a temporary URL for it.
4.  Creating a special component to handle loading the image for Konva.
5.  Rendering the image on the canvas.
6.  Adding image-specific options (including placeholder logic) to the Properties panel.

Let's get this done.

---

### **Phase 1, Step 10: Adding Image Elements**

#### **1. Update the Redux Slice (`editorSlice.ts`)**

First, define the `ImageElement` type and add it to our `CanvasElement` union.

**Update `src/features/editor/editorSlice.ts`:**

```typescript
// ... (imports and other interfaces)
export interface BaseElement { /* ... */ }
export interface TextElement extends BaseElement { /* ... */ }
export interface ShapeElement extends BaseElement { /* ... */ }

// ADD the new ImageElement interface
export interface ImageElement extends BaseElement {
  type: 'image';
  src: string; // URL of the image
  isPlaceholder: boolean;
  placeholderKey: string;
}

// UPDATE the CanvasElement discriminated union
export type CanvasElement = TextElement | ShapeElement | ImageElement;

// ... (The rest of the slice: state, reducers, actions, etc. remain unchanged.
// Our generic `addElement` and `updateElement` will work perfectly.)
```
**Key Change:** We've defined the structure for our image elements, including the all-important placeholder fields.

#### **2. Create a Custom Hook for Loading Images (`useImage`)**

This is a critical helper. Konva's `<Image>` component doesn't take a URL string (`src`). It needs an actual HTML `Image` object. This hook will handle loading the image from a URL and providing that object to our component.

**Create a new file: `src/hooks/useImage.ts`:**

```typescript
import { useState, useEffect } from 'react';

export const useImage = (url: string) => {
  const [image, setImage] = useState<HTMLImageElement | null>(null);

  useEffect(() => {
    if (!url) return;

    const img = new window.Image();
    img.src = url;

    // We need to enable cross-origin loading for some image sources
    img.crossOrigin = 'Anonymous';

    const handleLoad = () => {
      setImage(img);
    };

    img.addEventListener('load', handleLoad);

    // Cleanup function to remove the event listener
    return () => {
      img.removeEventListener('load', handleLoad);
    };
  }, [url]); // Rerun effect if the URL changes

  return image;
};
```

#### **3. Update the `LeftToolbar` and Create the `ImageFlyout`**

We'll add an "Image" button that opens our new flyout panel.

**First, the icon: `src/assets/icons/ImageIcon.tsx`**

```tsx
const ImageIcon = (props: React.SVGProps<SVGSVGElement>) => (
  <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" {...props}><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect><circle cx="8.5" cy="8.5" r="1.5"></circle><polyline points="21 15 16 10 5 21"></polyline></svg>
);
export default ImageIcon;
```

**Next, create the flyout component: `src/features/editor/layout/ImageFlyout.tsx`**

```tsx
import React, { useRef } from 'react';
import { useDispatch } from 'react-redux';
import { addElement, ImageElement } from '../editorSlice';

interface ImageFlyoutProps {
  onClose: () => void;
}

const ImageFlyout = ({ onClose }: ImageFlyoutProps) => {
  const dispatch = useDispatch();
  const fileInputRef = useRef<HTMLInputElement>(null);

  const handleUploadClick = () => {
    fileInputRef.current?.click();
  };

  const handleFileChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      const src = e.target?.result as string;
      const img = new window.Image();
      img.src = src;
      img.onload = () => {
        const defaultImageElement: Omit<ImageElement, 'id' | 'isVisible' | 'isLocked'> = {
          type: 'image',
          x: 100, y: 100,
          width: img.width * 0.5, // Start at 50% of original size
          height: img.height * 0.5,
          rotation: 0,
          src: src,
          isPlaceholder: false,
          placeholderKey: 'image_placeholder',
        };
        dispatch(addElement(defaultImageElement));
        onClose(); // Close the flyout
      };
    };
    reader.readAsDataURL(file);

    // Reset the input value to allow uploading the same file again
    event.target.value = '';
  };

  return (
    <div className="absolute left-full top-0 ml-2 w-60 bg-white rounded-lg shadow-lg border border-gray-200 p-2">
      <h4 className="font-semibold text-gray-800 px-2 pb-2">Images</h4>
      <input
        type="file"
        ref={fileInputRef}
        onChange={handleFileChange}
        className="hidden"
        accept="image/png, image/jpeg, image/gif, image/svg+xml"
      />
      <button
        onClick={handleUploadClick}
        className="w-full bg-blue-600 text-white p-2 text-sm rounded-md hover:bg-blue-700"
      >
        Upload Image
      </button>
      {/* Asset library / Stock photos would go here later */}
    </div>
  );
};

export default ImageFlyout;
```

**Finally, update `src/features/editor/layout/LeftToolbar.tsx` to use it:**

```tsx
// ... (add imports for ImageIcon and ImageFlyout)
import ImageIcon from '../../../assets/icons/ImageIcon';
import ImageFlyout from './ImageFlyout';

type ActiveFlyout = 'text' | 'shapes' | 'images' | null; // Add 'images'

const LeftToolbar = () => {
  // ... (state, ref, hook, and other handlers remain the same)
  
  return (
    <div ref={toolbarRef} className="relative">
      <aside className="w-20 bg-white border-r border-gray-300 h-full p-2 flex flex-col items-center space-y-4">
        {/* ... Text and Shapes buttons */}

        {/* Image Button (flyout toggle) */}
        <button
          onClick={() => handleToolbarClick('images')}
          className={`flex flex-col items-center p-2 rounded-lg transition-colors ${
            activeFlyout === 'images' ? 'bg-gray-200' : 'hover:bg-gray-200'
          }`}
          title="Add Image"
        >
          <ImageIcon className="w-8 h-8 text-gray-700" />
          <span className="text-xs mt-1">Images</span>
        </button>
      </aside>

      {/* Shapes Flyout Panel */}
      {activeFlyout === 'shapes' && ( /* ... */ )}

      {/* Image Flyout Panel */}
      {activeFlyout === 'images' && <ImageFlyout onClose={() => setActiveFlyout(null)} />}
    </div>
  );
};

export default LeftToolbar;
```

#### **4. Render Images on the Canvas**

We'll create a small wrapper component that uses our `useImage` hook, then use that in `CanvasArea`.

**Update `src/features/editor/canvas/CanvasArea.tsx`:**

```tsx
// Add Image to the Konva import
import { Stage, Layer, Text, Transformer, Rect, Image as KonvaImage } from 'react-konva';
// Import the hook and ImageElement type
import { useImage } from '../../../hooks/useImage';
import { ImageElement } from '../editorSlice';

// Create the new wrapper component INSIDE or ABOVE CanvasArea.tsx
const ImageComponent = ({ element }: { element: ImageElement }) => {
  const image = useImage(element.src);
  
  // Konva's Image component needs an actual image object, not a URL
  // The useImage hook provides this once it's loaded
  return <KonvaImage image={image} {...element} />;
};


const CanvasArea = () => {
  // ... (hooks, useEffect, and other handlers remain the same)

  return (
    // ...
        <Stage ...>
          <Layer>
            {elements.map((element) => {
              if (element.type === 'text') { /* ... */ }
              if (element.type === 'shape' && element.shapeType === 'rect') { /* ... */ }

              // ADD THIS BLOCK to render images
              if (element.type === 'image') {
                return (
                  <ImageComponent
                    key={element.id}
                    element={{
                      ...element,
                      id: element.id, // Ensure id is passed for transformer
                      draggable: !element.isLocked,
                      visible: element.isVisible,
                      onClick: (e) => {
                        if (!element.isLocked) dispatch(selectElement(element.id));
                        e.cancelBubble = true;
                      },
                      onTap: () => { if (!element.isLocked) dispatch(selectElement(element.id)) },
                      onDragEnd: (e) => { /* ... (same as Text/Rect) */ },
                      onTransformEnd: (e) => { /* ... (same as Text/Rect) */ },
                    }}
                  />
                );
              }
              return null;
            })}
            <Transformer /* ... */ />
          </Layer>
        </Stage>
    // ...
  );
};

export default CanvasArea;
```

#### **5. Add Image Properties to the Properties Panel**

Let's add the Static/Placeholder logic for images.

**Update `src/features/editor/layout/PropertiesTab.tsx`:**

```tsx
// ... (imports and component start)

  if (selectedElement.type === 'text') { /* ... */ }
  if (selectedElement.type === 'shape') { /* ... */ }

  // ADD THIS BLOCK for Image properties
  if (selectedElement.type === 'image') {
    const imageElement = selectedElement as ImageElement;
    return (
      <div className="p-4 space-y-4">
        <h3 className="text-lg font-semibold border-b pb-2">Image Properties</h3>
        
        {/* Content Type Toggle */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Content Type</label>
          <div className="flex rounded-md shadow-sm">
            <button
              onClick={() => handleUpdate({ isPlaceholder: false })}
              className={`flex-1 p-2 text-sm rounded-l-md ${!imageElement.isPlaceholder ? 'bg-blue-600 text-white' : 'hover:bg-gray-50'}`}
            >
              Static
            </button>
            <button
              onClick={() => handleUpdate({ isPlaceholder: true })}
              className={`flex-1 p-2 text-sm rounded-r-md border-l ${imageElement.isPlaceholder ? 'bg-blue-600 text-white' : 'hover:bg-gray-50'}`}
            >
              Placeholder
            </button>
          </div>
        </div>
        
        {imageElement.isPlaceholder ? (
          <div>
            <label htmlFor="placeholderKey" className="block text-sm font-medium text-gray-700">Parameter Name</label>
            <input
              type="text"
              id="placeholderKey"
              value={imageElement.placeholderKey}
              onChange={(e) => handleUpdate({ placeholderKey: e.target.value })}
              className="mt-1 block w-full rounded-md border-gray-300 shadow-sm sm:text-sm"
            />
          </div>
        ) : (
          <div>
            <label className="block text-sm font-medium text-gray-700">Image Preview</label>
            <img src={imageElement.src} alt="Preview" className="mt-1 w-full rounded-md border border-gray-300" />
          </div>
        )}
      </div>
    );
  }

  return <div className="p-4">Properties for this element type are not yet implemented.</div>;
```

---

### **Your Turn**

1.  Update and create all the necessary files as detailed above.
2.  **Test the image workflow:**
    *   Click the "Image" icon on the left to open the flyout.
    *   Click "Upload Image" and select an image from your computer.
    *   The image should appear on the canvas. You should be able to move, resize, and rotate it.
    *   Check the "Layers" panel—it should be there.
    *   Select the image and go to the "Properties" tab. You should see the static/placeholder toggle.
    *   Switch it to "Placeholder" and give it a parameter name.
    *   (Note: Visually, the image on the canvas won't change for a placeholder yet, but the state will be correct. We can add a visual overlay for placeholders later if needed).

You now have the three core element types—Text, Shapes, and Images—fully integrated into your editor. This is a massive milestone! The foundation is incredibly strong now.
