---
type: outfit
id: ""
style: ""
season: []
tags: []
images: []

items: []
group1: []
group2: []
group3: []

notes: ""
---
```dataviewjs
const container = dv.container;

const input = container.createEl("input");
input.type = "text";
input.placeholder = "🔎 Find item by ID, brand, name...";
Object.assign(input.style, { padding: "6px", width: "100%", marginBottom: "0.75em" });

const resultsEl = container.createEl("div");

function normalize(v) {
    if (!v) return "";
    if (Array.isArray(v)) return v.join(" ").toLowerCase();
    return v.toString().toLowerCase();
}

function fuzzyMatch(text, query) {
    if (!text) return false;
    let t = 0, q = 0;
    while (t < text.length && q < query.length) {
        if (text[t] === query[q]) q++;
        t++;
    }
    return q === query.length;
}

function scoreWord(blobWord, q) {
    if (blobWord === q) return 100;
    if (blobWord.startsWith(q)) return 80;
    if (blobWord.includes(q)) return 60;
    if (q.length >= 3 && fuzzyMatch(blobWord, q)) return 20;
    return 0;
}

function scoreBlob(blob, query) {
    if (!query) return 1;
    const queryWords = query.split(/\s+/).filter(Boolean);
    const blobWords = blob.split(/\s+/).filter(Boolean);
    let total = 0;
    for (let q of queryWords) {
        let best = 0;
        for (let bw of blobWords) best = Math.max(best, scoreWord(bw, q));
        if (blob.includes(q)) best = Math.max(best, 60);
        if (best === 0) return 0;
        total += best;
    }
    return total;
}

function render(query) {
    resultsEl.empty();
    if (!query) return;

    const clothes = dv.pages('"Clothing"')
        .where(p => p.type == "clothing")
        .array();

    const scored = [];
    for (let item of clothes) {
        const blob = [
            item.name, item.brand, item.category,
            item.item_id, item.notes,
            ...(Array.isArray(item.color) ? item.color : [item.color])
        ].map(normalize).join(" ");

        const score = scoreBlob(blob, query);
        if (score > 0) scored.push({ item, score });
    }

    scored.sort((a, b) => b.score - a.score);

    if (scored.length === 0) {
        resultsEl.createEl("p", { text: "No items found." });
        return;
    }

    for (let { item } of scored.slice(0, 10)) {
        const row = resultsEl.createEl("div");
        Object.assign(row.style, {
            display: "flex",
            alignItems: "center",
            gap: "0.75em",
            padding: "0.4em 0",
            borderBottom: "1px solid var(--background-modifier-border)",
            cursor: "pointer"
        });

        // Thumbnail
        const img = Array.isArray(item.images) ? item.images[0] : item.images;
        if (img) {
            const thumb = row.createEl("img", {
                attr: { src: app.vault.adapter.getResourcePath(img) }
            });
            thumb.setAttribute("style", "width: 48px; height: 48px; object-fit: cover; border-radius: 4px; flex-shrink: 0;");
        }

        // Info
        const info = row.createEl("div");
        Object.assign(info.style, { flex: "1", fontSize: "0.9em" });

        const nameEl = info.createEl("div", { text: item.name ?? item.file.name });
        nameEl.style.fontWeight = "bold";

        const sub = [];
        if (item.item_id) sub.push("🆔 " + item.item_id);
        if (item.brand) sub.push(item.brand);
        if (item.category) sub.push(item.category);
        info.createEl("div", { text: sub.join("  •  ") });

        // Copy button
        const btn = row.createEl("button", { text: "Copy link" });
        Object.assign(btn.style, { flexShrink: "0", cursor: "pointer", padding: "4px 8px" });

        const wikilink = `[[${item.file.name}]]`;

        btn.addEventListener("click", () => {
            navigator.clipboard.writeText(wikilink).then(() => {
                btn.textContent = "✅ Copied!";
                setTimeout(() => btn.textContent = "Copy link", 1500);
            });
        });
    }
}

let debounceTimer;
input.addEventListener("input", () => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => render(input.value.toLowerCase().trim()), 200);
});
```
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

---

## 🖼 Outfit Images
```dataviewjs
const imgs = dv.current().images ?? [];
for (let img of imgs) dv.paragraph(`![[${img}|300]]`);
```

---

## 👕 Outfit Preview
```dataviewjs
const page = dv.current();

function asList(v){ if(!v) return []; return Array.isArray(v)?v:[v]; }

function renderItem(link){
    let p = dv.page(link);
    if (!p) return;
    const img = p.images?.[0];
    if (!img) return;

    dv.el("img","",{
        attr:{ src: app.vault.adapter.getResourcePath(img), width:"120px"}
    });
}

// GROUPS
const groups = [page.group1, page.group2, page.group3];
let groupIndex = 1;

for (let g of groups){
    const list = asList(g);
    if (!list.length) continue;

    dv.header(3, "Group " + groupIndex);
    let container = dv.el("div","",{cls:"outfit-grid"});

    for (let item of list) renderItem(item);
    groupIndex++;
}

// LOOSE ITEMS
const loose = asList(page.items);
if (loose.length){
    dv.header(3,"Items");
    let container = dv.el("div","",{cls:"outfit-grid"});
    for (let item of loose) renderItem(item);
}
```

---

## 🏷 Info
```dataview
TABLE id, style, season, tags
WHERE file.path = this.file.path
```

---

## 📝 Notes
```dataview
LIST notes
WHERE file.path = this.file.path
```