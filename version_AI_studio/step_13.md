**USER**

Sure lets implement the STEP 1 which is setting up "JSON server"

**RESPONSE**

Perfect. Let's start with the foundation: setting up our mock backend (`json-server`) and getting it to run smoothly with our Vite frontend.

---

### **Step 1: Set Up `json-server` and Run Concurrently**

This step will give us a working API that we can start building against immediately.

#### **Action 1: Install Dependencies**

Open your project's terminal and run this command:

```bash
npm install --save-dev json-server concurrently
```

*   `--save-dev` installs these as development dependencies, which is appropriate since they are for our local development environment, not the final production build.

#### **Action 2: Create the Database File**

In the **root directory** of your project (the same folder that contains `package.json`, `vite.config.ts`, etc.), create a new file named `db.json`.

Copy and paste the following JSON content into `db.json`. This gives us one sample template to work with.

**`db.json`**
```json
{
  "templates": [
    {
      "id": "annual-report-template",
      "name": "Annual Report Concept",
      "thumbnailUrl": "/thumbnails/annual-report.png",
      "editorState": {
        "elements": [
          {
            "id": "eJ1op5_a-",
            "type": "text",
            "x": 60,
            "y": 60,
            "width": 300,
            "height": 55,
            "rotation": 0,
            "isVisible": true,
            "isLocked": false,
            "text": "Annual Report",
            "fontSize": 48,
            "fontFamily": "Arial",
            "fill": "#333333",
            "isPlaceholder": false,
            "placeholderKey": "title"
          },
          {
            "id": "fgH2pL-b_",
            "type": "text",
            "x": 60,
            "y": 150,
            "rotation": 0,
            "isVisible": true,
            "isLocked": false,
            "text": "Default Name",
            "fontSize": 24,
            "fontFamily": "Arial",
            "fill": "#555555",
            "isPlaceholder": true,
            "placeholderKey": "recipient_name"
          },
          {
            "id": "kL9rT_8cM",
            "type": "shape",
            "shapeType": "rect",
            "x": 350,
            "y": 200,
            "width": 150,
            "height": 150,
            "rotation": 0,
            "isVisible": true,
            "isLocked": true,
            "fill": "#e0e0e0",
            "stroke": "#aaaaaa",
            "strokeWidth": 2
          }
        ],
        "selectedElementId": null
      }
    }
  ]
}
```

#### **Action 3: Create the Thumbnail Image**

For the demo to work, the `thumbnailUrl` needs to point to a real image.

1.  In your `public` directory, create a new folder named `thumbnails`.
2.  Save any small image inside this new folder and name it `annual-report.png`. You can take a screenshot of your current editor canvas or just use a placeholder image for now. The path should be `public/thumbnails/annual-report.png`.

#### **Action 4: Update `package.json` Scripts**

Now, let's wire everything up so it runs with a single command. Open your `package.json` file and find the `"scripts"` section.

Replace the entire `"scripts"` block with this new configuration:

```json
"scripts": {
  "dev": "concurrently \"npm run server\" \"npm run vite\"",
  "vite": "vite",
  "server": "json-server --watch db.json --port 3001",
  "build": "tsc && vite build",
  "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
  "preview": "vite preview"
},
```

**Explanation:**
*   `server`: This new script starts `json-server`, tells it to watch our `db.json` file for changes, and runs it on port `3001`.
*   `vite`: This is the original `dev` script for starting the React app.
*   `dev`: This is our new main command. It uses `concurrently` to run both the `server` and `vite` scripts at the same time.

---

### **Your Turn: Verification**

1.  **Stop** your current development server if it's running (Ctrl+C).
2.  Run the new command:
    ```bash
    npm run dev
    ```
3.  Your terminal should now show output from **both** `concurrently`, the `json-server`, and `vite`. It will look something like this:

    ```
    [server]   Resources
    [server]   http://localhost:3001/templates
    [server]
    [vite]     VITE v5.x.x  ready in xxx ms
    [vite]     ➜  Local:   http://localhost:5173/
    ```

4.  **Test the API:** Open a new browser tab and navigate to `http://localhost:3001/templates`. You should see the JSON data from your `db.json` file.
5.  **Test the App:** Your React application should still be running as normal at `http://localhost:5173/`.

Once you have confirmed that both servers are running correctly, let me know, and we will proceed to **Step 2: Implement Routing and the Dashboard Page.**
