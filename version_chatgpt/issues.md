**issue_1**

---

here you are deviating from the folder structure the preious structure is 
```
canva-designer/
├─ src/
│  ├─ app/
│  │  └─ store.ts
│  ├─ features/
│  │  ├─ canvas/
│  │  │  ├─ CanvasSlice.ts
│  │  │  └─ components/
│  │  │     ├─ CanvasStage.tsx
│  │  │     ├─ FloatingToolbar.tsx
│  │  │     ├─ BottomToolbar.tsx
│  │  │     ├─ LeftPanel.tsx
│  │  │     ├─ RightPanel.tsx
│  │  │     └─ TopBar.tsx
│  ├─ pages/
│  │  └─ DesignerPage.tsx
│  ├─ types/
│  │  └─ design.ts
│  ├─ utils/
│  ├─ main.tsx
│  └─ index.css
└─ package.json

```

but you gave another structure

```
src/
 ├── components/
 │    ├── CanvasDesigner.tsx
 │    ├── FloatingToolbar.tsx
 │    ├── BottomToolbar.tsx  
 │
 ├── store/
 │    ├── index.ts
 │    ├── designSlice.ts
 │
 ├── pages/
 │    └── DesignerPage.tsx
 │
 ├── App.tsx
 └── main.tsx
```

** What is DesignSlice , we already have CanvasSlice?

** problem in app/strore , then store/index.ts , store/designSlice !!!
---
