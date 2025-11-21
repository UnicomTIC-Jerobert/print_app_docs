**USER**

lets build sendout page

**RESPONSE**

Of course. Let's build the final and most impactful page for your client demo: the Sendout Page.

This page will showcase the true power of your application. It will take a finished template, extract its placeholders, and provide a user-friendly interface for entering data, complete with a live preview.

---

### **Step 4: Build the Sendout Page**

#### **Action 1: Create the Main `SendoutPage` Component**

This component will be the container for this feature. It will fetch the template data and manage the user's input.

1.  Inside `src/features`, create a new folder named `sendouts`.
2.  Inside `src/features/sendouts`, create a new file named `SendoutPage.tsx`.

Paste the following code into `SendoutPage.tsx`. This is the skeleton of the page, including data fetching and state management.

**`src/features/sendouts/SendoutPage.tsx`**
```tsx
import React, { useEffect, useState, useMemo } from 'react';
import { useParams, Link } from 'react-router-dom';
import { CanvasElement } from '../editor/editorSlice';

// A simplified type for our template data on this page
interface TemplateData {
  id: string;
  name: string;
  editorState: {
    elements: CanvasElement[];
  };
}

// Represents a single placeholder found in the template
interface Placeholder {
  key: string;
  type: 'text' | 'image'; // We can extend this later
}

const SendoutPage = () => {
  const { templateId } = useParams<{ templateId: string }>();
  const [template, setTemplate] = useState<TemplateData | null>(null);
  const [loading, setLoading] = useState(true);
  
  // State to hold the user's data, starts with one empty row
  const [rows, setRows] = useState<Record<string, string>[]>([]);
  // State to track which row is selected for the live preview
  const [activeRowIndex, setActiveRowIndex] = useState(0);

  // 1. Fetch the template data when the component mounts
  useEffect(() => {
    if (templateId) {
      fetch(`http://localhost:3001/templates/${templateId}`)
        .then(res => res.json())
        .then(data => {
          setTemplate(data);
          setLoading(false);
        });
    }
  }, [templateId]);

  // 2. Extract placeholders from the template once it's loaded
  const placeholders = useMemo((): Placeholder[] => {
    if (!template) return [];
    
    const foundPlaceholders: Placeholder[] = [];
    template.editorState.elements.forEach(el => {
      if ('isPlaceholder' in el && el.isPlaceholder) {
        foundPlaceholders.push({
          key: ('placeholderKey' in el && el.placeholderKey) || 'unknown',
          type: el.type,
        });
      }
    });
    return foundPlaceholders;
  }, [template]);

  // 3. Initialize the data rows once placeholders are known
  useEffect(() => {
    if (placeholders.length > 0) {
      const initialRow = Object.fromEntries(placeholders.map(p => [p.key, '']));
      setRows([initialRow]);
    }
  }, [placeholders]);
  
  const handleInputChange = (rowIndex: number, key: string, value: string) => {
    const newRows = [...rows];
    newRows[rowIndex][key] = value;
    setRows(newRows);
  };

  const addRow = () => {
    const newRow = Object.fromEntries(placeholders.map(p => [p.key, '']));
    setRows([...rows, newRow]);
  };
  
  if (loading) return <div>Loading template...</div>;
  if (!template) return <div>Template not found.</div>;

  return (
    <div className="flex h-screen bg-gray-100">
      {/* Left Side: Data Entry */}
      <div className="w-1/2 p-6 overflow-y-auto">
        <Link to="/" className="text-sm font-medium text-blue-600 hover:underline mb-4 block">
          &larr; Back to Dashboard
        </Link>
        <h1 className="text-2xl font-bold mb-1">Sendout for: <span className="text-blue-700">{template.name}</span></h1>
        <p className="text-gray-600 mb-6">Enter the data for each recipient below.</p>
        
        <div className="space-y-4">
          {rows.map((row, rowIndex) => (
            <div 
              key={rowIndex} 
              onClick={() => setActiveRowIndex(rowIndex)}
              className={`p-4 rounded-lg cursor-pointer ${activeRowIndex === rowIndex ? 'bg-white shadow-lg ring-2 ring-blue-500' : 'bg-white shadow'}`}
            >
              <h3 className="font-semibold mb-2 text-gray-700">Recipient #{rowIndex + 1}</h3>
              <div className="grid grid-cols-2 gap-4">
                {placeholders.map(p => (
                  <div key={p.key}>
                    <label className="block text-sm font-medium text-gray-600 capitalize">{p.key.replace(/_/g, ' ')}</label>
                    <input
                      type="text"
                      value={row[p.key] || ''}
                      onChange={(e) => handleInputChange(rowIndex, p.key, e.target.value)}
                      className="mt-1 block w-full rounded-md border-gray-300 shadow-sm sm:text-sm"
                    />
                  </div>
                ))}
              </div>
            </div>
          ))}
        </div>
        
        <button onClick={addRow} className="mt-6 bg-gray-200 px-4 py-2 text-sm font-semibold rounded-md hover:bg-gray-300">
          + Add Recipient
        </button>
      </div>

      {/* Right Side: Live Preview */}
      <div className="w-1/2 bg-gray-200 p-6 flex flex-col items-center justify-center">
        <h2 className="text-lg font-bold mb-4 text-gray-800">Live Preview</h2>
        {/* The PreviewCanvas will go here */}
        <div className="bg-white shadow-2xl">
          <p>Preview coming soon...</p>
        </div>
      </div>
    </div>
  );
};

export default SendoutPage;
```

#### **Action 2: Create the `PreviewCanvas` Component**

This is a **display-only** version of our canvas. It doesn't need Redux or any editing features. Its only job is to take template elements and a row of data, and merge them visually.

**Create a new file: `src/features/sendouts/PreviewCanvas.tsx`**
```tsx
import React from 'react';
import { Stage, Layer, Text, Rect, Image as KonvaImage } from 'react-konva';
import { CanvasElement } from '../editor/editorSlice';
import { useImage } from '../../hooks/useImage'; // We reuse our handy hook!

// A4 dimensions for consistency
const A4_WIDTH = 595;
const A4_HEIGHT = 842;

// The ImageComponent from CanvasArea.tsx is useful here too
const ImageComponent = ({ element }: { element: any }) => {
  const image = useImage(element.src);
  return <KonvaImage image={image} {...element} />;
};

interface PreviewCanvasProps {
  elements: CanvasElement[];
  dataRow: Record<string, string>; // The data for the currently selected recipient
}

const PreviewCanvas = ({ elements, dataRow }: PreviewCanvasProps) => {
  return (
    <Stage width={A4_WIDTH} height={A4_HEIGHT}>
      <Layer>
        {elements.map((element) => {
          // The core "merge" logic happens here
          if (element.type === 'text') {
            const displayText =
              element.isPlaceholder && dataRow[element.placeholderKey]
                ? dataRow[element.placeholderKey] // Use data from the row
                : element.isPlaceholder
                ? `{{${element.placeholderKey}}}` // Fallback if no data
                : element.text; // Static text
            return <Text key={element.id} {...element} text={displayText} />;
          }

          if (element.type === 'shape') {
            return <Rect key={element.id} {...element} />;
          }

          if (element.type === 'image') {
            // In a real app, you'd merge image URLs here too
            // For now, we just display the template's static/placeholder image
            return <ImageComponent key={element.id} element={element} />;
          }

          return null;
        })}
      </Layer>
    </Stage>
  );
};

export default PreviewCanvas;
```

#### **Action 3: Integrate the Preview into the `SendoutPage`**

Now, let's replace the "Preview coming soon..." placeholder with our new `PreviewCanvas` component.

**File to edit: `src/features/sendouts/SendoutPage.tsx`**

```tsx
// At the top, import the new component
import PreviewCanvas from './PreviewCanvas';

// ... inside the SendoutPage component's return statement, find the "Right Side" comment block
// and replace it with this:
      {/* Right Side: Live Preview */}
      <div className="w-1/2 bg-gray-200 p-6 flex flex-col items-center justify-center">
        <h2 className="text-lg font-bold mb-4 text-gray-800">Live Preview</h2>
        <div className="bg-white shadow-2xl">
          <PreviewCanvas
            elements={template.editorState.elements}
            dataRow={rows[activeRowIndex] || {}}
          />
        </div>
      </div>
// ...
```

#### **Action 4: Update the Main Router**

Finally, let's replace the placeholder route with our real component in the main `App.tsx` file.

**File to edit: `src/App.tsx`**
```tsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import EditorLayout from './features/editor/layout/EditorLayout';
import DashboardPage from './features/dashboard/DashboardPage';
import SendoutPage from './features/sendouts/SendoutPage'; // Import the real component

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<DashboardPage />} />
        <Route path="/editor/:templateId" element={<EditorLayout />} />
        {/* Replace the placeholder with the real component */}
        <Route path="/sendouts/:templateId" element={<SendoutPage />} />
      </Routes>
    </Router>
  );
}

export default App;
```

---

### **Your Turn: Verification**

This is the final step of the demo flow!

1.  Go to your dashboard at `http://localhost:5173/`.
2.  Click the **"Use"** button on the "Annual Report Concept" card.
3.  **Verification 1 (Layout):** You should be on the Sendout page. The left side should show an input field for "Recipient Name" (because that's the placeholder in our `db.json`). The right side should show the canvas preview.
4.  **Verification 2 (Live Preview):**
    *   In the "Recipient Name" input field, start typing your name.
    *   **As you type, the text on the canvas preview should update in real-time!** This is the key moment of the demo.
5.  **Verification 3 (Multiple Rows):**
    *   Click the "+ Add Recipient" button. A second form should appear.
    *   Type a different name into the second form. The preview won't change yet.
    *   Now, click on the first recipient's card. The preview should show the first name.
    *   Click on the second recipient's card. The preview should update to show the second name.

You have now successfully built a complete, end-to-end design templating and data-merge application flow. This is a fantastic and comprehensive demo for your client.
