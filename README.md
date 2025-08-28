<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>JSON Rarity Editor</title>
  <style>
    :root{--bg:#f7fafc;--card:#ffffff;--muted:#6b7280;--accent:#4f46e5;--surface:#ffffff;--text:#0f172a;--border:#e6edf3;--accent:#4f46e5;--accent-strong:#3730a3;--danger:#ff7777;--shadow:0 1px 2px rgba(2,6,23,0.04);}
    html[data-theme="dark"] {--bg: #0b1220;--surface: #0f1724;--card: #111827;--text: #e6eef6;--muted: #9aa6b2;--border: #1f2937;--accent: #82a3ff;--accent-strong: #4f6ef6;--danger: #4b1f1f;--shadow: 0 2px 8px rgba(2,6,23,0.6);}
    html[data-theme="contrast"] {--bg: #0b1220;--surface: #0f1724;--card: #111827;--text: #fffb00;--muted: #b0b29a;--border: #ff16d8;--accent: #48ff00;--accent-strong: #ff0062;--danger: #ff0000;--shadow: 0 2px 8px rgba(2,6,23,0.6);}
    html,body{height:100%;margin:0;font-family:Inter,ui-sans-serif,system-ui,-apple-system,"Segoe UI",Roboto,"Helvetica Neue",Arial}
    body{background:var(--bg);color:var(--text)}
    .container{max-width:1200px;margin:24px auto;padding:18px}
    .header{display:flex;gap:12px;align-items:center;justify-content:space-between}
    .btn{display:inline-block;background:var(--card);border:1px solid var(--border);padding:8px 12px;border-radius:8px;cursor:pointer;color:var(--text);}
    .primary{background:var(--accent);color:var(--border);border:none}
    .grid{display:grid;grid-template-columns:220px 420px 1fr;gap:16px;margin-top:16px}
    .card{background:var(--card);padding:12px;border-radius:10px;box-shadow:var(--shadow);border:1px solid var(--border)}
    h1{font-size:18px;margin:0}
    h2{font-size:14px;margin:0 0 8px 0}
    .rarity-btn{display:block;width:100%;text-align:left;padding:8px;border-radius:8px;border:none;background:transparent;cursor:pointer;color: var(--text);}
    .rarity-btn.active{background:var(--card);border-left:4px solid var(--accent)}
    .items-list{max-height:70vh;overflow:auto}
    .item-row{display:flex;justify-content:space-between;align-items:center;padding:8px;border-radius:6px}
    .item-row:hover{background:var(--bg)}
    input[type="text"],input[type="number"],textarea,select{width:100%;padding:8px;border-radius:6px;border:1px solid var(--border);background: var(--bg);color: var(--text);}
    .small{font-size:13px;color:var(--muted)}
    pre{white-space:pre-wrap;background:var(--surface);padding:10px;border-radius:8px;max-height:300px;overflow:auto;scrollbar-color: var(--surface) var(--bg);scrollbar-arrow-color: var(--surface);}
    .flex{display:flex;gap:8px;align-items:center}
    .col{display:flex;flex-direction:column;gap:8px}
    .muted{color:var(--muted)}
    .danger{background:var(--danger);border-color:var(--danger)}
    .controls{display:flex;gap:8px;align-items:center}
    .sample-area{margin-top:8px}
    label.file-label{cursor:pointer}
    .inline{display:inline-block}
    #filename{cursor: text;}
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <div>
        <h1>JSON Rarity Editor</h1>
        <div class="small muted">Mainly created to edit the affixes and suffixes.</div>
      </div>

      <div class="controls">
        <label class="btn file-label">
          Upload JSON
          <input id="fileInput" type="file" accept="application/json" style="display:none">
        </label>
        <button id="loadSampleBtn" class="btn">Load sample JSON</button>
        <button id="newBtn" class="btn">New Template</button>
        <input id="filename" type="text" class="btn" style="width:160px" value="items.json">
        <button id="downloadBtn" class="btn primary">Download JSON</button>
        <select id="themeSelect" class="btn">
          <option value="system">System</option>
          <option value="light">Light</option>
          <option value="dark">Dark</option>
          <option value="contrast">High Contrast</option>
        </select>
      </div>
    </div>

    <div class="grid">
      <aside class="card">
        <h2>Rarities</h2>
        <div id="rarities" style="display:flex;flex-direction:column;gap:6px;margin-top:8px"></div>
        <div style="margin-top:12px">
          <button id="addItemToRarity" class="btn primary" style="width:100%">Add item to selected rarity</button>
        </div>
      </aside>

      <section class="card">
        <h2>Items</h2>
        <div class="items-list" id="itemsList"></div>
      </section>

      <main class="card">
        <h2>Editor</h2>
        <div id="editorArea" class="col">
          <div class="muted">Select an item to edit or create a new one.</div>
        </div>

        <div style="margin-top:12px">
          <h3 class="small">Preview JSON</h3>
          <pre id="preview"></pre>
        </div>
      </main>
    </div>

  </div>

  <script>
    const ATTRIBUTE_LIST = [
      'precision','APCostReduction','weight','pierce','blunt','loot','HPGain','slash','range','damage',
      'armorBlunt','armorReduction','selfDamage','constantDamage','instantKillChance','missChance','gold',
      'damageBlocked','selfHitChance','HPDrain'
    ];

    const VALID_RARITIES = ['common','rare','epic','godly','magic','none'];

    function makeEmptyTemplate(){ return VALID_RARITIES.reduce((acc, r)=>{ acc[r] = {}; return acc; }, {}); }
    function deepClone(v){ return JSON.parse(JSON.stringify(v)); }

    let data = deepClone(makeEmptyTemplate());
    let selectedRarity = VALID_RARITIES[0];
    let selectedItem = null;

    const raritiesEl = document.getElementById('rarities');
    const itemsListEl = document.getElementById('itemsList');
    const editorArea = document.getElementById('editorArea');
    const previewEl = document.getElementById('preview');
    const fileInput = document.getElementById('fileInput');
    const downloadBtn = document.getElementById('downloadBtn');
    const newBtn = document.getElementById('newBtn');
    const addItemBtn = document.getElementById('addItemToRarity');
    const filenameInput = document.getElementById('filename');
    const loadSampleBtn = document.getElementById('loadSampleBtn');
    const themeSelect = document.getElementById('themeSelect');

    function normalizeImported(parsed){
      if (!parsed || typeof parsed !== 'object') return deepClone(makeEmptyTemplate());
      if (parsed.rarity && typeof parsed.rarity === 'object') parsed = parsed.rarity;
      const result = deepClone(makeEmptyTemplate());
      VALID_RARITIES.forEach(r => { if (parsed[r] && typeof parsed[r] === 'object') result[r] = deepClone(parsed[r]); });
      VALID_RARITIES.forEach(r => {
        const items = result[r];
        Object.keys(items).forEach(name => {
          let item = items[name];
          if (!Array.isArray(item) && typeof item === 'object') { item = ['', item]; }
          if (Array.isArray(item)) {
            if (item.length === 1) item.push({});
            if (item[1] && typeof item[1].stat === 'object') item[1] = deepClone(item[1].stat);
          }
          items[name] = item;
        });
      });
      return result;
    }

    function getStatType(statObj){
      if (!statObj) return 'fixed';
      if (statObj.prc !== undefined && statObj.prc !== null) return 'prc';
      if (statObj.rprc && (Number.isFinite(statObj.rprc.min) || Number.isFinite(statObj.rprc.max))) return 'rprc';
      return 'fixed';
    }

    function parseNullableNumber(str){ if (str === null || str === undefined || str === '') return null; const n = Number(str); return Number.isFinite(n) ? n : null; }

    function renderRarities(){ raritiesEl.innerHTML = ''; VALID_RARITIES.forEach(r => { const btn = document.createElement('button'); btn.className = 'rarity-btn' + (r === selectedRarity ? ' active' : ''); btn.textContent = r; btn.onclick = () => { selectedRarity = r; selectedItem = null; render(); }; raritiesEl.appendChild(btn); }); }

    function renderItems(){
      itemsListEl.innerHTML = '';
      const items = data[selectedRarity] || {};
      const names = Object.keys(items);
      if (names.length === 0) { const d = document.createElement('div'); d.className = 'small muted'; d.textContent = 'No items yet'; itemsListEl.appendChild(d); return; }
      names.forEach(name => {
        const row = document.createElement('div'); row.className = 'item-row';
        const left = document.createElement('div'); left.style.flex = '1';
        const title = document.createElement('div'); title.textContent = name; title.style.fontWeight = '600';
        const desc = document.createElement('div'); desc.className = 'small muted'; desc.style.maxWidth = '180px'; desc.style.overflow = 'hidden'; desc.style.textOverflow = 'ellipsis'; desc.textContent = (items[name] && items[name][0]) || '';
        left.appendChild(title); left.appendChild(desc); row.appendChild(left);
        const actions = document.createElement('div'); actions.style.display = 'flex'; actions.style.gap = '6px';
        const selectBtn = document.createElement('button'); selectBtn.className = 'btn'; selectBtn.textContent = 'Edit'; selectBtn.onclick = () => { selectedItem = name; render(); };
        const renameBtn = document.createElement('button'); renameBtn.className = 'btn'; renameBtn.textContent = 'Rename'; renameBtn.onclick = () => { const nn = prompt('Rename item', name); if (nn && nn !== name) renameItem(name, nn); };
        const delBtn = document.createElement('button'); delBtn.className = 'btn danger'; delBtn.textContent = 'Delete'; delBtn.onclick = () => { if (confirm(`Delete item "${name}" from ${selectedRarity}?`)) deleteItem(name); };
        actions.appendChild(selectBtn); actions.appendChild(renameBtn); actions.appendChild(delBtn); row.appendChild(actions); itemsListEl.appendChild(row);
      });
    }

    function renderEditor(){
      editorArea.innerHTML = '';
      if (!selectedItem) { const info = document.createElement('div'); info.className = 'muted'; info.textContent = 'Select or create an item to edit.'; editorArea.appendChild(info); return; }
      const items = data[selectedRarity] || {};
      const item = items[selectedItem] || ['', {}];

      const nameLabel = document.createElement('label'); nameLabel.textContent = 'Name';
      const nameInput = document.createElement('input'); nameInput.type = 'text'; nameInput.value = selectedItem; nameInput.onchange = () => { const v = nameInput.value.trim(); if (v && v !== selectedItem) renameItem(selectedItem, v); };
      const descLabel = document.createElement('label'); descLabel.textContent = 'Description';
      const descInput = document.createElement('textarea'); descInput.rows = 3; descInput.value = item[0] || '';
      descInput.onchange = () => { updateItemDescription(selectedItem, descInput.value); };

      editorArea.appendChild(nameLabel); editorArea.appendChild(nameInput);
      editorArea.appendChild(descLabel); editorArea.appendChild(descInput);

      const stats = (Array.isArray(item) ? (item[1] || {}) : {});
      const statsWrapper = document.createElement('div');
      const statsHeader = document.createElement('div'); statsHeader.className = 'flex'; statsHeader.style.justifyContent = 'space-between';
      const sh = document.createElement('div'); sh.textContent = 'Stats'; sh.style.fontWeight = '600';
      const addStatBtn = document.createElement('button'); addStatBtn.className = 'btn primary'; addStatBtn.textContent = 'Add stat'; addStatBtn.onclick = () => { addStat(selectedItem); };
      statsHeader.appendChild(sh); statsHeader.appendChild(addStatBtn); statsWrapper.appendChild(statsHeader);

      if (Object.keys(stats).length === 0) { const d = document.createElement('div'); d.className = 'small muted'; d.textContent = 'No stats yet'; statsWrapper.appendChild(d); }

      Object.entries(stats).forEach(([attr, statObj]) => {
        const card = document.createElement('div'); card.style.border = '1px solid var(--border)'; card.style.padding = '8px'; card.style.borderRadius = '8px'; card.style.marginTop = '8px';
        const row = document.createElement('div'); row.className = 'grid'; row.style.gridTemplateColumns = '220px 260px 180px'; row.style.gap = '8px';

        const attrDiv = document.createElement('div');
        const attrLbl = document.createElement('div'); attrLbl.className = 'small muted'; attrLbl.textContent = 'Attribute';
        const attrSelect = document.createElement('select');
        ATTRIBUTE_LIST.forEach(a => { const o = document.createElement('option'); o.value = a; o.textContent = a; attrSelect.appendChild(o); });
        const customOpt = document.createElement('option'); customOpt.value = '__custom__'; customOpt.textContent = 'Custom...'; attrSelect.appendChild(customOpt);
        if (!ATTRIBUTE_LIST.includes(attr)) { attrSelect.value = '__custom__'; } else { attrSelect.value = attr; }
        const customInput = document.createElement('input'); customInput.type = 'text'; customInput.placeholder = 'Custom attribute name'; customInput.style.display = ATTRIBUTE_LIST.includes(attr) ? 'none' : 'block'; customInput.value = ATTRIBUTE_LIST.includes(attr) ? '' : attr;
        attrSelect.onchange = () => { if (attrSelect.value === '__custom__') { customInput.style.display = 'block'; customInput.focus(); } else { customInput.style.display = 'none'; renameStat(attr, attrSelect.value); } };
        customInput.onchange = () => { const nv = customInput.value.trim(); if (nv) renameStat(attr, nv); else { customInput.value = attr; } };
        attrDiv.appendChild(attrLbl); attrDiv.appendChild(attrSelect); attrDiv.appendChild(customInput); row.appendChild(attrDiv);

        const center = document.createElement('div'); center.className = 'col';
        const typeLbl = document.createElement('div'); typeLbl.className = 'small muted'; typeLbl.textContent = 'Type';
        const typeSelect = document.createElement('select');
        ['fixed','prc','rprc'].forEach(t=>{const o=document.createElement('option');o.value=t;o.textContent=t;typeSelect.appendChild(o);});
        typeSelect.value = getStatType(statObj);
        typeSelect.onchange = () => { setStatType(attr, typeSelect.value); render(); };
        center.appendChild(typeLbl);
        center.appendChild(typeSelect);
        row.appendChild(attrDiv);
        row.appendChild(center);
        // make sure the selector is visible and sized
        try { typeSelect.style.display = 'block'; typeSelect.style.width = '120px'; typeSelect.style.minWidth = '80px'; typeSelect.style.marginBottom = '6px'; } catch(e) {}
        const currentType = getStatType(statObj);
        typeSelect.value = currentType;
        typeSelect.onchange = () => { setStatType(attr, typeSelect.value); render(); };

        const fixedLbl = document.createElement('div'); fixedLbl.className = 'small muted'; fixedLbl.textContent = 'Fixed';
        const fixedInput = document.createElement('input'); fixedInput.type = 'number'; fixedInput.step = 'any'; fixedInput.value = (statObj && statObj.fixed) != null ? statObj.fixed : '';
        fixedInput.onchange = () => { setStatField(attr, 'fixed', fixedInput.value); };

        const prcLbl = document.createElement('div'); prcLbl.className = 'small muted'; prcLbl.textContent = 'prc (optional)';
        const prcInput = document.createElement('input'); prcInput.type = 'number'; prcInput.step = 'any'; prcInput.value = (statObj && statObj.prc !== undefined && statObj.prc !== null) ? statObj.prc : '';
        prcInput.onchange = () => { setStatField(attr, 'prc', prcInput.value); render(); };

        const stackLbl = document.createElement('div'); stackLbl.className = 'small muted'; stackLbl.textContent = 'stack (optional)';
        const stackInput = document.createElement('input'); stackInput.type = 'number'; stackInput.step = '1'; stackInput.value = (statObj && statObj.stack !== undefined && statObj.stack !== null) ? statObj.stack : '';
        stackInput.onchange = () => { setStatField(attr, 'stack', stackInput.value); renderPreview(); };

        center.appendChild(typeLbl); center.appendChild(typeSelect);
        center.appendChild(fixedLbl); center.appendChild(fixedInput);
        center.appendChild(prcLbl); center.appendChild(prcInput);
        center.appendChild(stackLbl); center.appendChild(stackInput);

        const right = document.createElement('div'); right.className = 'col';
        const rprcLbl = document.createElement('div'); rprcLbl.className = 'small muted'; rprcLbl.textContent = 'rprc (min / max)';
        const rprcMin = document.createElement('input'); rprcMin.type = 'number'; rprcMin.step = 'any'; rprcMin.placeholder = 'min';
        const rprcMax = document.createElement('input'); rprcMax.type = 'number'; rprcMax.step = 'any'; rprcMax.placeholder = 'max';
        rprcMin.value = statObj.rprc && statObj.rprc.min!=null?statObj.rprc.min:'';
        rprcMax.value = statObj.rprc && statObj.rprc.max!=null?statObj.rprc.max:'';
        rprcMin.onchange = () => { setRprcField(attr,'min',rprcMin.value); };
        rprcMax.onchange = () => { setRprcField(attr,'max',rprcMax.value); };
        right.appendChild(rprcLbl);
        right.appendChild(rprcMin);
        right.appendChild(rprcMax);
        
        
        row.appendChild(right);
        card.appendChild(row);
        statsWrapper.appendChild(card);
        });
      editorArea.appendChild(statsWrapper);
      const hint = document.createElement('div'); hint.className = 'small muted'; hint.textContent = 'Changes are applied live. Use Download JSON to save to a file.'; editorArea.appendChild(hint);
    }

    function applyTheme(name){ if (name === 'system'){document.documentElement.removeAttribute('data-theme');} else {document.documentElement.setAttribute('data-theme', name);} localStorage.setItem('app-theme', name);}
    const savedTheme = localStorage.getItem('app-theme') || 'system';
    themeSelect.value = savedTheme;
    applyTheme(savedTheme);
    function createItem(){ const items = data[selectedRarity]; let base = 'New Item'; let i = 1; let name = base; while (items[name]) { name = base + ' ' + (i++); } items[name] = ['description...', {}]; selectedItem = name; render(); }
    function deleteItem(name){ delete data[selectedRarity][name]; selectedItem = null; render(); }
    function renameItem(oldName, newName){ if (!newName) return; if (data[selectedRarity][newName]) { alert('Name already exists'); return; } data[selectedRarity][newName] = data[selectedRarity][oldName]; delete data[selectedRarity][oldName]; selectedItem = newName; render(); }
    function updateItemDescription(name, desc){ if (!data[selectedRarity][name]) return; data[selectedRarity][name][0] = desc; renderPreview(); }

    function addStat(itemName){ const items = data[selectedRarity]; const it = items[itemName]; if (!it[1]) it[1] = {}; const stats = it[1]; let pick = ATTRIBUTE_LIST.find(a => !Object.prototype.hasOwnProperty.call(stats, a)); if (!pick) { let i = 1; pick = 'attribute_' + i; while (stats[pick]) { i++; pick = 'attribute_' + i; } } stats[pick] = { fixed: 0 }; render(); }
    function deleteStat(attr){ const stats = data[selectedRarity][selectedItem][1]; delete stats[attr]; render(); }
    function renameStat(oldAttr, newAttr){ if (!newAttr) return; const stats = data[selectedRarity][selectedItem][1]; if (oldAttr === newAttr) return; if (stats[newAttr]) { if (!confirm(`Attribute "${newAttr}" already exists. Overwrite?`)) return; } stats[newAttr] = stats[oldAttr]; delete stats[oldAttr]; render(); }

    function setStatType(attr, type){ const stat = data[selectedRarity][selectedItem][1][attr]; if (!stat) return; if (type === 'fixed') { delete stat.prc; if (!(stat.rprc && (Number.isFinite(stat.rprc.min) || Number.isFinite(stat.rprc.max)))) delete stat.rprc; } else if (type === 'prc') { stat.prc = (stat.prc != null) ? stat.prc : 0; stat.fixed = Number((stat.prc / 10).toFixed(1)); if (!(stat.rprc && (Number.isFinite(stat.rprc.min) || Number.isFinite(stat.rprc.max)))) delete stat.rprc; } else if (type === 'rprc') { stat.rprc = stat.rprc || { min: null, max: null }; stat.fixed = 0; delete stat.prc; } renderPreview(); }

    function setStatField(attr, field, value){ const stat = data[selectedRarity][selectedItem][1][attr]; if (!stat) return; if (field === 'fixed') { const n = parseNullableNumber(value); if (n == null) delete stat.fixed; else stat.fixed = n; } else if (field === 'prc') { const n = parseNullableNumber(value); if (n == null) delete stat.prc; else { stat.prc = n; stat.fixed = Number((stat.prc / 10).toFixed(1)); } if (!(stat.rprc && (Number.isFinite(stat.rprc.min) || Number.isFinite(stat.rprc.max)))) delete stat.rprc; } else if (field === 'stack') { const n = parseNullableNumber(value); if (n == null) delete stat.stack; else stat.stack = Math.floor(n); } else if (field === 'rprc_min') { const n = parseNullableNumber(value); stat.rprc = stat.rprc || { min: null, max: null }; stat.rprc.min = n; if (!(Number.isFinite(stat.rprc.min) || Number.isFinite(stat.rprc.max))) delete stat.rprc; else { stat.fixed = 0; delete stat.prc; } } else if (field === 'rprc_max') { const n = parseNullableNumber(value); stat.rprc = stat.rprc || { min: null, max: null }; stat.rprc.max = n; if (!(Number.isFinite(stat.rprc.min) || Number.isFinite(stat.rprc.max))) delete stat.rprc; else { stat.fixed = 0; delete stat.prc; } } renderPreview(); }

    fileInput.addEventListener('change', e => { const f = e.target.files[0]; if (!f) return; const reader = new FileReader(); reader.onload = ev => { try { let parsed = JSON.parse(ev.target.result); parsed = normalizeImported(parsed); data = deepClone(parsed); selectedRarity = VALID_RARITIES[0]; selectedItem = null; filenameInput.value = f.name.replace(/\.[^.]+$/,'') + '.json'; render(); } catch (err) { alert('Invalid JSON: ' + err.message); } }; reader.readAsText(f); e.target.value = null; });

    downloadBtn.addEventListener('click', () => { const json = JSON.stringify(data, null, 2); const blob = new Blob([json], { type: 'application/json' }); const url = URL.createObjectURL(blob); const a = document.createElement('a'); a.href = url; a.download = filenameInput.value || 'items.json'; a.click(); URL.revokeObjectURL(url); });

    newBtn.addEventListener('click', () => { data = deepClone(makeEmptyTemplate()); selectedRarity = VALID_RARITIES[0]; selectedItem = null; render(); });
    addItemBtn.addEventListener('click', () => { createItem(); });
    themeSelect.addEventListener('change', () => { applyTheme(themeSelect.value); });


    loadSampleBtn.addEventListener('click', () => {
      const sample = {
        "common":{
          "Rusty Sword":["A dull old blade", {"attack": {"fixed":1}}],
          "Old Shield":["Wooden shield", {"defense": {"fixed":2, "stack":2}}]
        },
        "epic":{
          "Flamebrand":["Burns enemies", {"damage": {"fixed":10, "prc":50}}]
        }
      };
      data = deepClone(normalizeImported(sample)); selectedRarity = 'common'; selectedItem = null; render();
    });

    function renderPreview(){ previewEl.textContent = JSON.stringify(data, null, 2); }
    function render(){ renderRarities(); renderItems(); renderEditor(); renderPreview(); }
    render();
  </script>
</body>
</html>
