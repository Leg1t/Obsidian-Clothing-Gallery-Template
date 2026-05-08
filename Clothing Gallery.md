```dataviewjs
const container = dv.container;
let filter = "";
let brandFilter = "";

// --- Helpers ---
function asList(v) {
    if (!v) return [];
    return Array.isArray(v) ? v : [v];
}

function normalize(v) {
    if (!v) return "";
    if (Array.isArray(v)) return v.join(" ").toLowerCase();
    return v.toString().toLowerCase();
}

function fuzzyMatch(text, query) {
    if (!query) return true;
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

// --- Static UI (built once) ---
const searchInput = container.createEl("input");
searchInput.type = "text";
searchInput.placeholder = "🔎 Search by name, brand, ID, notes...";
Object.assign(searchInput.style, {
    marginBottom: "0.5em",
    padding: "6px",
    width: "100%",
    display: "block"
});

const brandSelect = container.createEl("select");
Object.assign(brandSelect.style, {
    marginBottom: "1em",
    padding: "6px",
    width: "100%",
    display: "block"
});

const resultsEl = container.createEl("div");
Object.assign(resultsEl.style, {
    width: "100%",
    display: "block"
});

// --- Populate brand dropdown ---
const allClothes = dv.pages('"Clothing"')
    .where(p => p.type == "clothing")
    .array();

const brands = [...new Set(
    allClothes
        .map(p => p.brand)
        .filter(Boolean)
        .map(b => b.toString().trim())
)].sort();

brandSelect.createEl("option", { text: "👕 All brands", attr: { value: "" } });

for (let brand of brands) {
    brandSelect.createEl("option", { text: brand, attr: { value: brand.toLowerCase() } });
}

// --- Render ---
function render() {
    resultsEl.empty();

    const clothes = dv.pages('"Clothing"')
        .where(p => p.type == "clothing")
        .array();

    const scored = [];

    for (let item of clothes) {

        if (brandFilter) {
            const itemBrand = normalize(item.brand);
            if (itemBrand !== brandFilter) continue;
        }

        let score = 1;
        if (filter) {
            const blob = [
                item.name,
                item.brand,
                item.category,
                item.notes,
                item.item_id,
                item.color,
                item.material,
                item.pattern,
                item.fit,
                item.occasion,
                item.layer,
                ...asList(item.season),
                ...asList(item.tags)
            ].map(normalize).join(" ");

            score = scoreBlob(blob, filter);
        }

        if (score > 0) scored.push({ item, score });
    }

    scored.sort((a, b) => b.score - a.score || normalize(a.item.name ?? a.item.file.name).localeCompare(normalize(b.item.name ?? b.item.file.name)));

    if (scored.length === 0) {
        resultsEl.createEl("p", { text: "No items found." });
        return;
    }

    for (let { item } of scored) {

        const card = resultsEl.createEl("div", { cls: "card" });
        Object.assign(card.style, {
            width: "100%",
            display: "block",
            marginBottom: "1.5em",
            paddingBottom: "1em",
            borderBottom: "1px solid var(--background-modifier-border)"
        });

        // Title link
        const title = card.createEl("div", { cls: "title" });
        title.createEl("a", {
            text: item.name ?? item.file.name,
            cls: "internal-link",
            attr: { href: item.file.path, "data-href": item.file.path }
        });

        // ID
        if (item.item_id) {
            card.createEl("div", {
                text: "🆔 " + item.item_id,
                cls: "item-id"
            });
        }

        // Image
        const img = asList(item.images)[0];
        if (img) {
            const imgEl = card.createEl("img", {
                attr: { src: app.vault.adapter.getResourcePath(img) }
            });
            imgEl.setAttribute("style", "width: 100% !important; height: auto !important; border-radius: 6px; display: block; margin-top: 0.5em;");
        }

        // Meta row
        const meta = [];
        if (item.brand) meta.push("🏷 " + item.brand);
        if (item.category) meta.push("👔 " + item.category);
        if (item.color) meta.push("🎨 " + asList(item.color).join(", "));
        if (item.season) meta.push("🌦 " + asList(item.season).join(", "));

        if (meta.length) {
            const metaEl = card.createEl("div", {
                text: meta.join("  •  "),
                cls: "meta"
            });
            Object.assign(metaEl.style, {
                marginTop: "0.4em",
                fontSize: "0.85em",
                opacity: "0.8"
            });
        }

        // Notes
        if (item.notes) {
            const noteEl = card.createEl("div", {
                text: item.notes,
                cls: "note"
            });
            Object.assign(noteEl.style, {
                marginTop: "0.4em",
                fontSize: "0.85em",
                fontStyle: "italic",
                opacity: "0.75"
            });
        }
    }
}

// --- Events ---
let debounceTimer;

searchInput.addEventListener("input", () => {
    filter = searchInput.value.toLowerCase().trim();
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(render, 200);
});

brandSelect.addEventListener("change", () => {
    brandFilter = brandSelect.value.toLowerCase().trim();
    render();
});

render();
```