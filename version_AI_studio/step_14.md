**USER**

**Step 2: Implement Routing and the Dashboard Page.**

**RESPONSE**

Great. Now that our mock backend is running, let's build the user-facing parts of the application: the navigation system (routing) and the main dashboard where users will see their designs.

---

### **Step 2: Implement Routing & The Dashboard Page**

#### **Action 1: Install React Router**

This library is the standard for handling navigation in React applications.

In your terminal, run:

```bash
npm install react-router-dom
```

#### **Action 2: Create the Dashboard Page Component**

We need a new home for this feature.

1.  Inside `src/features`, create a new folder named `dashboard`.
2.  Inside `src/features/dashboard`, create a new file named `DashboardPage.tsx`.

Copy and paste the following code into your new `DashboardPage.tsx` file.

**`src/features/dashboard/DashboardPage.tsx`**
```tsx
import React, { useEffect, useState } from 'react';
import { Link } from 'react-router-dom';

// This interface defines the shape of the data we expect from our API
interface Template {
  id: string;
  name: string;
  thumbnailUrl: string;
}

const DashboardPage = () => {
  const [templates, setTemplates] = useState<Template[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // This effect runs once when the component mounts
    const fetchTemplates = async () => {
      try {
        const response = await fetch('http://localhost:3001/templates');
        if (!response.ok) {
          throw new Error('Network response was not ok');
        }
        const data: Template[] = await response.json();
        setTemplates(data);
      } catch (e) {
        setError('Failed to fetch designs. Is the JSON server running?');
        console.error(e);
      } finally {
        setLoading(false);
      }
    };

    fetchTemplates();
  }, []); // The empty dependency array means this effect runs only once

  if (loading) {
    return <div className="p-8 text-center">Loading designs...</div>;
  }
  
  if (error) {
    return <div className="p-8 text-center text-red-500">{error}</div>;
  }

  return (
    <div className="p-8 bg-gray-100 min-h-screen">
      <h1 className="text-3xl font-bold mb-6 text-gray-800">My Designs</h1>
      <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
        {templates.map(template => (
          <div key={template.id} className="bg-white rounded-lg shadow-md overflow-hidden group transition-shadow hover:shadow-xl">
            <div className="w-full h-40 bg-gray-200">
              <img src={template.thumbnailUrl} alt={template.name} className="w-full h-full object-cover" />
            </div>
            <div className="p-4 flex justify-between items-center">
              <h3 className="font-semibold truncate text-gray-700">{template.name}</h3>
              <div className="flex space-x-2">
                <Link to={`/sendouts/${template.id}`} className="text-sm bg-gray-200 px-3 py-1 rounded hover:bg-gray-300 text-gray-800">
                  Use
                </Link>
                <Link to={`/editor/${template.id}`} className="text-sm bg-blue-600 text-white px-3 py-1 rounded hover:bg-blue-700">
                  Edit
                </Link>
              </div>
            </div>
          </div>
        ))}
        {/* We can add a "Create New" button here later */}
      </div>
    </div>
  );
};

export default DashboardPage;
```

#### **Action 3: Set Up the Application Router**

Now we'll tell our application which component to show for which URL. We'll replace the content of `App.tsx` with our new router configuration.

**Open `src/App.tsx` and replace its entire content with this:**

```tsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import EditorLayout from './features/editor/layout/EditorLayout';
import DashboardPage from './features/dashboard/DashboardPage';

// We'll create the SendoutPage component in a later step.
// For now, we can use a placeholder.
const SendoutPagePlaceholder = () => (
  <div className="p-8">
    <h1 className="text-2xl font-bold">Sendout Page</h1>
    <p>This page will be implemented soon.</p>
  </div>
);


function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<DashboardPage />} />
        <Route path="/editor/:templateId" element={<EditorLayout />} />
        <Route path="/sendouts/:templateId" element={<SendoutPagePlaceholder />} />
      </Routes>
    </Router>
  );
}

export default App;
```

---

### **Your Turn: Verification**

1.  Make sure your `npm run dev` process is still running. The Vite server should hot-reload automatically.
2.  Open your browser and navigate to the root URL: `http://localhost:5173/`.
3.  You should now see the **Dashboard Page**, not the editor. It should display a single card for "Annual Report Concept" with its thumbnail.
4.  **Test the links:**
    *   Click the **"Edit"** button. The URL should change to `/editor/annual-report-template`, and you should see the editor layout. (It won't load the correct data yet; that's the next step).
    *   Go back to the dashboard. Click the **"Use"** button. The URL should change to `/sendouts/annual-report-template`, and you should see our simple placeholder text.

Once you have successfully verified this new navigation and the dashboard is displaying correctly, let me know, and we'll move on to 
