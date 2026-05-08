```dataviewjs
const container = dv.container;
let filter = "";
let seasonFilter = "";
let groupBySeason = true;

// --- Static UI ---
const searchInput = container.createEl("input");
searchInput.type = "text";
searchInput.placeholder = "🔎 Fuzzy search outfits & items...";
Object.assign(searchInput.style, { marginBottom: "0.5em", padding: "6px", width: "100%", display: "block" });

const controlsRow = container.createEl("div");
Object.assign(controlsRow.style, { display: "flex", gap: "0.5em", marginBottom: "1em" });

const seasonSelect = controlsRow.createEl("select");
Object.assign(seasonSelect.style, { flex: "1", padding: "6px" });

const groupToggle = controlsRow.createEl("button");
groupToggle.textContent = "📂 Grouped";
Object.assign(groupToggle.style, { padding: "6px 10px", cursor: "pointer", flexShrink: "0" });

const resultsEl = container.createEl("div", { cls: "results-root" });

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

function makeLink(parent, path, displayName) {
    parent.createEl("a", {
        text: displayName || path,
        cls: "internal-link",
        attr: { href: path, "data-href": path }
    });
}

// --- Populate season dropdown ---
const allOutfits = dv.pages('"Outfits"').where(p => p.type == "outfit").array();

const seasons = [...new Set(
    allOutfits.flatMap(o => asList(o.season)).map(s => s.toString().trim()).filter(Boolean)
)].sort();

seasonSelect.createEl("option", { text: "🌦 All seasons", attr: { value: "" } });
for (let s of seasons) {
    seasonSelect.createEl("option", { text: s, attr: { value: s.toLowerCase() } });
}

// --- Render outfit block ---
function renderOutfitBlock(parent, outfit) {
    const block = parent.createEl("div", { cls: "outfit-block" });

    const h2 = block.createEl("h2");
    makeLink(h2, outfit.file.path, outfit.file.name);

    if (outfit.id) block.createEl("div", { text: "🆔 " + outfit.id });

    const grid = block.createEl("div");
    Object.assign(grid.style, {
        display: "flex",
        flexDirection: "column",
        gap: "1em",
        marginTop: "0.5em"
    });

    function renderItemCard(itemLink) {
        const page = dv.page(itemLink);
        if (!page) return;
        const img = asList(page.images)[0];
        if (!img) return;

        const card = grid.createEl("div", { cls: "card" });
        Object.assign(card.style, { width: "100%", marginBottom: "0.5em" });

        const meta = card.createEl("div", { cls: "meta" });
        makeLink(meta, page.file.path, page.file.name);

        if (page.item_id) card.createEl("div", { text: "🆔 " + page.item_id, cls: "item-id" });

        const imgEl = card.createEl("img", { attr: { src: app.vault.adapter.getResourcePath(img) } });
        imgEl.setAttribute("style", "width: 100% !important; height: auto !important; border-radius: 6px; display: block;");

        if (page.notes) card.createEl("div", { text: page.notes, cls: "note" });
    }

    for (let img of asList(outfit.images)) {
        const el = grid.createEl("img", { attr: { src: app.vault.adapter.getResourcePath(img) } });
        el.setAttribute("style", "width: 100% !important; height: auto !important; border-radius: 6px; display: block;");
    }

    function renderGroup(groupItems, label) {
        const list = asList(groupItems);
        if (!list.length) return;
        grid.createEl("div", { text: "🧩 " + label, cls: "group-label" });
        for (let item of list) renderItemCard(item);
    }

    renderGroup(outfit.group1, "Group 1");
    renderGroup(outfit.group2, "Group 2");
    renderGroup(outfit.group3, "Group 3");

    if (asList(outfit.items).length) {
        grid.createEl("div", { text: "👕 Items", cls: "group-label" });
        for (let item of asList(outfit.items)) renderItemCard(item);
    }

    const tags = asList(outfit.tags);
    if (tags.length) block.createEl("p", { text: "🏷 " + tags.join(", ") });

    const outfitSeasons = asList(outfit.season);
    if (outfitSeasons.length) block.createEl("p", { text: "🌦 " + outfitSeasons.join(", ") });

    const footer = block.createEl("p");
    makeLink(footer, outfit.file.path, outfit.file.name);

    block.createEl("hr");
}

// --- Main render ---
function render() {
    resultsEl.empty();

    // Grouping only makes sense when not searching
    const isSearching = filter.length > 0;
    const shouldGroup = groupBySeason && !isSearching;

    // Update toggle button appearance
    groupToggle.textContent = shouldGroup ? "📂 Grouped" : "☰ Flat";
    groupToggle.style.opacity = isSearching ? "0.4" : "1";

    const outfits = dv.pages('"Outfits"')
        .where(p => p.type == "outfit")
        .sort(p => p.file.name)
        .array();

    const scored = [];

    for (let outfit of outfits) {
        // Season filter
        if (seasonFilter) {
            const outfitSeasons = asList(outfit.season).map(s => s.toString().toLowerCase());
            if (!outfitSeasons.includes(seasonFilter)) continue;
        }

        let score = 0;

        if (!filter) {
            score = 1;
        } else {
            const outfitBlob = [outfit.id, ...asList(outfit.season), outfit.file.name]
                .map(normalize).join(" ");
            score = scoreBlob(outfitBlob, filter);

            if (score === 0) {
                const allItems = [
                    ...asList(outfit.items),
                    ...asList(outfit.group1),
                    ...asList(outfit.group2),
                    ...asList(outfit.group3)
                ];
                for (let itemLink of allItems) {
                    const page = dv.page(itemLink);
                    if (!page) continue;
                    const itemBlob = [page.name, page.brand, page.category, page.notes, page.item_id]
                        .map(normalize).join(" ");
                    const s = scoreBlob(itemBlob, filter);
                    if (s > score) score = s;
                }
            }
        }

        if (score > 0) scored.push({ outfit, score });
    }

    scored.sort((a, b) => b.score - a.score || a.outfit.file.name.localeCompare(b.outfit.file.name));

    if (scored.length === 0) {
        resultsEl.createEl("p", { text: "No outfits found." });
        return;
    }

    if (shouldGroup) {
        // Group by season
        const groups = new Map();
        const noSeason = [];

        
        for (let { outfit } of scored) {
		    const outfitSeasons = asList(outfit.season).map(s => s.toString().trim());
		    if (outfitSeasons.length === 0) {
		        noSeason.push(outfit);
		    } else {
		        const primary = outfitSeasons[0];
		        if (!groups.has(primary)) groups.set(primary, []);
		        groups.get(primary).push(outfit);
		    }
		}

        // Render each season group
        for (let [season, outfitsInGroup] of [...groups.entries()].sort()) {
            const section = resultsEl.createEl("div");

            const header = section.createEl("h1", { text: "🌦 " + season });
            Object.assign(header.style, {
                borderBottom: "2px solid var(--background-modifier-border)",
                paddingBottom: "0.25em",
                marginBottom: "0.5em"
            });

            for (let outfit of outfitsInGroup) renderOutfitBlock(section, outfit);
        }

        if (noSeason.length) {
            const section = resultsEl.createEl("div");
            const header = section.createEl("h1", { text: "🌦 No season" });
            Object.assign(header.style, {
                borderBottom: "2px solid var(--background-modifier-border)",
                paddingBottom: "0.25em",
                marginBottom: "0.5em"
            });
            for (let outfit of noSeason) renderOutfitBlock(section, outfit);
        }

    } else {
        // Flat list
        for (let { outfit } of scored) renderOutfitBlock(resultsEl, outfit);
    }
}

// --- Events ---
let debounceTimer;

searchInput.addEventListener("input", () => {
    filter = searchInput.value.toLowerCase().trim();
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(render, 200);
});

seasonSelect.addEventListener("change", () => {
    seasonFilter = seasonSelect.value.toLowerCase().trim();
    render();
});

groupToggle.addEventListener("click", () => {
    groupBySeason = !groupBySeason;
    render();
});

render();
```