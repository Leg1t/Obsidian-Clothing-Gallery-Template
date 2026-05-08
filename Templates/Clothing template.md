---
type: clothing
name: ""
category: ""
brand: ""
color: []
material: []
pattern: []
tags: []
season: []
occasion: []
layer: ""
fit: ""
rating:
item_id: ""
images: []
notes: ""
---
```dataviewjs
const container = dv.container;
const currentFile = app.workspace.getActiveFile();

container.createEl("p", { text: "📸 Upload image to vault:" });

const input = container.createEl("input");
input.type = "file";
input.accept = "image/*";
Object.assign(input.style, { marginBottom: "0.5em", display: "block" });

const status = container.createEl("p", { text: "" });

input.addEventListener("change", async () => {
    const file = input.files?.[0];
    if (!file) return;

    status.textContent = "⏳ Uploading...";

    try {
        // Read file as binary
        const arrayBuffer = await file.arrayBuffer();

        // Build a unique filename to avoid collisions
        const ext = file.name.split(".").pop();
        const baseName = file.name.replace(/\.[^.]+$/, "").replace(/[^a-zA-Z0-9_-]/g, "_");
        const timestamp = Date.now();
        const fileName = `Images/${baseName}_${timestamp}.${ext}`;

        // Check if Images folder exists, create if not
        const folder = app.vault.getAbstractFileByPath("Images");
        if (!folder) await app.vault.createFolder("Images");

        // Write file to vault
        await app.vault.createBinary(fileName, arrayBuffer);

        // Set images property in frontmatter
        await app.fileManager.processFrontMatter(currentFile, fm => {
            fm.images = [fileName];
        });


        status.textContent = `✅ Saved as ${fileName} and linked!`;

        // Show preview
        const preview = container.createEl("img");
        preview.setAttribute("style", "width: 100% !important; height: auto !important; border-radius: 6px; display: block; margin-top: 0.5em;");
        preview.src = app.vault.adapter.getResourcePath(fileName);

    } catch (err) {
        status.textContent = "❌ Error: " + err.message;
    }
});
```
# {{title}}

## 🖼 Images
```dataviewjs
const imgs = dv.current().images ?? [];
for (let img of imgs) {
    dv.paragraph(`![[${img}]]`);
}
```

---

## 📦 Info
```dataview
TABLE brand, category, color, material, pattern, season, occasion, layer, fit, rating
WHERE file.path = this.file.path
```

---

## 🏷 Tags
```dataview
TABLE tags
WHERE file.path = this.file.path
```

---

## 📝 Notes
```dataview
LIST notes
WHERE file.path = this.file.path
```

---


## 👗 Worn in Outfits
```dataviewjs
const currentPath = dv.current().file.path;
const outfits = dv.pages('"Outfits"').where(o => o.type == "outfit");

function asList(v){ if(!v) return []; return Array.isArray(v)?v:[v]; }

let results = [];

for (let o of outfits){

    let links = [
        ...asList(o.items),
        ...asList(o.group1),
        ...asList(o.group2),
        ...asList(o.group3)
    ];

    for (let l of links){
        if (dv.page(l)?.file.path === currentPath){
            results.push(o.file.link);
            break;
        }
    }
}

dv.list(results);
```