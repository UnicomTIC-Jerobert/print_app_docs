**USER**

**Step 3: Connecting the Editor to Load and Save Data.**

**RESPONSE**

Excellent. Let's wire up the editor. This is a crucial step where the application will start to feel whole. We'll make it so that when you click "Edit" on a design, its actual content loads into the editor, and any changes you make are automatically saved back to our mock backend.

---

### **Step 3: Connect the Editor to Load and Save Data**

#### **Action 1: Add a New "Load" Reducer to the Redux Slice**

We need an action that can completely replace the editor's state with data we fetch from the server.

**File to edit: `src/features/editor/editorSlice.ts`**

1.  First, we need to export the `EditorState` type because we'll use it in our new reducer. Find the `EditorState` interface and add the `export` keyword.
    ```typescript
    // Add the `export` keyword here
    export interface EditorState {
      elements: CanvasElement[];
      selectedElementId: string | null;
    }
    ```
2.  Next, add the new reducer inside the `reducers: { ... }` block.
    ```typescript
    // ...inside the reducers object of createSlice
    reducers: {
      // ... (addElement, selectElement, etc. remain the same)
      
      // ADD THIS NEW REDUCER
      setEditorState: (state, action: PayloadAction<EditorState>) => {
        // This will replace the entire current state with the one from the server
        state.elements = action.payload.elements;
        state.selectedElementId = action.payload.selectedElementId;
      },
    },
    ```
3.  Finally, export the new action creator at the bottom of the file.
    ```typescript
    export const {
      // ... (existing actions)
      reorderElements,
      setEditorState, // ADD THIS
    } = editorSlice.actions;
    ```

#### **Action 2: Load the Template Data When the Editor Mounts**

We'll use the `templateId` from the URL to fetch the correct design and dispatch our new `setEditorState` action.

**File to edit: `src/features/editor/layout/EditorLayout.tsx`**

```tsx
import React, { useEffect } from 'react';
import { useParams } from 'react-router-dom';
import { useDispatch } from 'react-redux';
import { setEditorState, EditorState } from '../editorSlice'; // Import setEditorState and EditorState
import LeftToolbar from './LeftToolbar';
import TopHeader from './TopHeader';
import PropertiesPanel from './PropertiesPanel';
import CanvasArea from '../canvas/CanvasArea';
import ContextualToolbar from './ContextualToolbar';

const EditorLayout = () => {
  // Get the templateId from the URL (e.g., "annual-report-template")
  const { templateId } = useParams<{ templateId: string }>();
  const dispatch = useDispatch();

  useEffect(() => {
    // This effect runs when the component mounts or when templateId changes
    if (templateId) {
      console.log(`Fetching data for template: ${templateId}`);
      
      // Fetch the specific template data from our json-server
      fetch(`http://localhost:3001/templates/${templateId}`)
        .then(res => res.json())
        .then(data => {
          // The template object has an `editorState` property inside it
          if (data.editorState) {
            // Dispatch the action to load this state into Redux
            dispatch(setEditorState(data.editorState as EditorState));
          }
        })
        .catch(error => console.error("Failed to load template:", error));
    }
  }, [templateId, dispatch]);

  return (
    <div className="flex flex-col h-screen w-screen bg-gray-100 font-sans">
      <TopHeader />
      <div className="flex flex-1 overflow-hidden">
        <LeftToolbar />
        <main className="flex-1 bg-gray-200 p-8 overflow-auto relative">
          <ContextualToolbar />
          <CanvasArea />
        </main>
        <PropertiesPanel />
      </div>
    </div>
  );
};

export default EditorLayout;
```

#### **Action 3: Implement the Auto-Save Manager**

Now we'll create the auto-save functionality we designed earlier.

1.  **Create the `useDebounce` hook.**
    *   Create the folder `src/hooks`.
    *   Create the file `src/hooks/useDebounce.ts`.
    *   Paste this code:
    ```typescript
    import { useState, useEffect } from 'react';

    export function useDebounce<T>(value: T, delay: number): T {
      const [debouncedValue, setDebouncedValue] = useState<T>(value);
      useEffect(() => {
        const handler = setTimeout(() => {
          setDebouncedValue(value);
        }, delay);
        return () => {
          clearTimeout(handler);
        };
      }, [value, delay]);
      return debouncedValue;
    }
    ```

2.  **Create the `AutoSaveManager` component.**
    *   Create the file `src/features/editor/AutoSaveManager.tsx`.
    *   Paste this code:
    ```tsx
    import { useEffect, useState, useRef } from 'react';
    import { useSelector } from 'react-redux';
    import { RootState } from '../../app/store';
    import { useDebounce } from '../../hooks/useDebounce';

    enum SaveStatus {
      SAVED = 'All changes saved',
      SAVING = 'Saving...',
      UNSAVED = 'Unsaved changes',
      ERROR = 'Error saving',
    }

    const AutoSaveManager = ({ documentId }: { documentId: string }) => {
      const [status, setStatus] = useState<SaveStatus>(SaveStatus.SAVED);
      const editorState = useSelector((state: RootState) => state.editor);
      const debouncedEditorState = useDebounce(editorState, 2000); // 2-second delay
      const isInitialMount = useRef(true);

      // Effect to detect unsaved changes
      useEffect(() => {
        if (!isInitialMount.current) {
          setStatus(SaveStatus.UNSAVED);
        }
      }, [editorState]);

      // Effect to trigger the save
      useEffect(() => {
        if (isInitialMount.current) {
          isInitialMount.current = false;
          return;
        }

        const saveData = async () => {
          setStatus(SaveStatus.SAVING);
          try {
            const response = await fetch(`http://localhost:3001/templates/${documentId}`, {
              method: 'PATCH',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ editorState: debouncedEditorState }),
            });
            if (!response.ok) throw new Error('Failed to save');
            setStatus(SaveStatus.SAVED);
          } catch (error) {
            setStatus(SaveStatus.ERROR);
            console.error('Save failed:', error);
          }
        };

        saveData();
      }, [debouncedEditorState, documentId]);

      return <div className="text-sm text-gray-500 mr-4">{status}</div>;
    };

    export default AutoSaveManager;
    ```
3.  **Add the `AutoSaveManager` to the `TopHeader`**.
    *   **File to edit: `src/features/editor/layout/TopHeader.tsx`**
    ```tsx
    import React from 'react';
    import { Link, useParams } from 'react-router-dom';
    import AutoSaveManager from '../AutoSaveManager';

    const TopHeader = () => {
      const { templateId } = useParams<{ templateId: string }>();

      return (
        <header className="h-16 bg-white border-b border-gray-300 flex items-center justify-between px-4 text-gray-800">
          <Link to="/" className="text-sm font-medium hover:text-blue-600">
            &larr; Back to Dashboard
          </Link>
          <div className="flex-1 text-center">
            {/* We could show the design name here later */}
          </div>
          {templateId && <AutoSaveManager documentId={templateId} />}
        </header>
      );
    };

    export default TopHeader;
    ```
---

### **Your Turn: Verification**

1.  Navigate to your dashboard at `http://localhost:5173/`.
2.  Click the "Edit" button on the "Annual Report Concept" card.
3.  **Verification 1 (Loading):** The editor should now load with the three elements defined in our `db.json` file. You should see the "Annual Report" text, the `{{recipient_name}}` placeholder, and the locked grey rectangle.
4.  **Verification 2 (Saving):**
    *   Make a change. For example, select the "Annual Report" text and change its color to blue.
    *   Watch the status in the top right. It should change to "Unsaved changes", then "Saving...", and finally "All changes saved".
    *   Open your `db.json` file in your code editor. You should see that the `fill` property for that text element has been updated to your new blue color hex code!
    *   Refresh the editor page. The text should load in as blue, proving that your save and load cycle is complete.

Once you have this working, the core functionality of your editor is essentially complete for the demo. Let me know when you're ready to build the final piece: the Sendout Page.
