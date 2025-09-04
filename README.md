<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Lanzelot Cipher — Web Demo</title>
  <style>
    :root{font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial;--card:#fff;--bg:#f3f6fb}
    body{margin:0;background:var(--bg);padding:28px}
    .wrap{max-width:980px;margin:0 auto}
    header{display:flex;align-items:center;gap:16px;margin-bottom:18px}
    h1{font-size:20px;margin:0}
    .card{background:var(--card);border-radius:12px;padding:18px;box-shadow:0 6px 18px rgba(20,30,60,0.06);margin-bottom:14px}
    label{display:block;font-size:13px;color:#333;margin-bottom:6px}
    textarea,input[type=text],input[type=password]{width:100%;padding:10px;border-radius:8px;border:1px solid #d6dbe6;font-size:14px}
    .grid{display:grid;grid-template-columns:1fr 1fr;gap:12px}
    .controls{display:flex;gap:8px;flex-wrap:wrap;margin-top:10px}
    button{background:#0b63ff;color:#fff;border:0;padding:8px 12px;border-radius:8px;cursor:pointer}
    button.ghost{background:#f4f6fb;color:#0b63ff;border:1px solid #d6e0ff}
    .row{display:flex;gap:8px;align-items:center}
    small{color:#556}
    .output{font-family:monospace;padding:10px;border-radius:8px;border:1px dashed #d6dbe6;background:#fcfdff}
    .footer{font-size:13px;color:#445;margin-top:6px}
    .muted{color:#6b7280}
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Lanzelot Cipher — Web Demo</h1>
      <div class="muted">A multi-layer experimental cipher</div>
    </header>

    <section class="card">
      <label>Plaintext (letters & numbers only will be used; all "1" are shown as "!")</label>
      <textarea id="plaintext" rows="4" placeholder="Type message, e.g. HELLO123">HELLO123</textarea>

      <div style="height:12px"></div>
      <div class="grid">
        <div>
          <label>Passphrase</label>
          <input id="passphrase" type="password" placeholder="Secret passphrase" value="Lanzelot42" />
        </div>
        <div>
          <label>Salt (optional)</label>
          <input id="salt" type="text" placeholder="Optional salt" value="01" />
        </div>
      </div>

      <div style="height:12px"></div>
      <div class="row">
        <button id="encrypt">Encrypt</button>
        <button id="decrypt" class="ghost">Decrypt</button>
        <div style="flex:1"></div>
      </div>

      <div style="height:12px"></div>
      <div>
        <label>Ciphertext (all "1" displayed as "!")</label>
        <div id="ciphertext" class="output"></div>
      </div>

      <div style="height:8px"></div>
      <div class="controls">
        <button id="copyCipher">Copy ciphertext</button>
        <button id="copyPlain" class="ghost">Copy plaintext</button>
        <button id="download" class="ghost">Download .html (this page)</button>
        <div style="flex:1"></div>
        <div class="muted">Tip: this demo displays "!" instead of "1". Decrypt accepts either "1" or "!" in the ciphertext input.</div>
      </div>
    </section>

    <section class="card">
      <h3 style="margin-top:0">About this demo</h3>
      <p class="muted">This page implements the LNZT cipher (seeded substitution, additive keystream, nibble mixing, block transposition and CBC-like diffusion). It uses the browser's <code>crypto.subtle</code> SHA-256. Encryption and decryption are deterministic for the same passphrase/salt (no per-message nonce). All displayed output replaces the character "1" with "!"; when decrypting the page will accept either "1" or "!" as input.</p>
      <div class="footer">⚠️ Not audited for production use. Use only for learning and experimentation.</div>
    </section>
  </div>

<script>
// LNZT cipher — JavaScript/Browser port (nonce removed; deterministic by passphrase+salt)
// UI: replaces displayed '1' with '!'; decrypt accepts both '1' and '!'

// Alphabet A-Z then 0-9
const ALPHABET = Array.from({length:26}, (_,i)=>String.fromCharCode(65+i)).concat(Array.from({length:10},(_,i)=>String(i)));
const M = ALPHABET.length; // 36

const enc = new TextEncoder();
const dec = new TextDecoder();

async function sha256Bytes(input) {
  const data = (typeof input === 'string') ? enc.encode(input) : input;
  const buf = await crypto.subtle.digest('SHA-256', data);
  return new Uint8Array(buf);
}

function toHex(bytes) {
  return Array.from(bytes).map(b=>b.toString(16).padStart(2,'0')).join('');
}

async function deriveSeedBytes(passphrase, salt) {
  return await sha256Bytes(passphrase + '::' + (salt||""));
}

async function seededShuffle(items, seedBytes) {
  const out = items.slice();
  let idx = 0;
  for (let i = out.length - 1; i > 0; --i) {
    const payload = new Uint8Array(seedBytes.length + 1);
    payload.set(seedBytes, 0);
    payload[seedBytes.length] = idx & 0xFF;
    const chunk = await sha256Bytes(payload);
    let val = 0n;
    for (let k=0;k<8 && k<chunk.length;k++) {
      val = (val << 8n) + BigInt(chunk[k]);
    }
    const j = Number(val % BigInt(i+1));
    const tmp = out[i]; out[i] = out[j]; out[j] = tmp;
    idx += 1;
  }
  return out;
}

async function deriveKeystream(passphrase, nonce, length) {
  const out = [];
  let counter = 0;
  while (out.length < length) {
    const block = await sha256Bytes(passphrase + '::' + nonce + '::' + counter);
    for (let b of block) {
      out.push(b % M);
      if (out.length >= length) break;
    }
    counter += 1;
  }
  return out;
}

function indexOfSym(ch) { return ALPHABET.indexOf(ch); }
function symOf(idx) { return ALPHABET[idx % M]; }

function displayReplaceOnes(s) {
  return s.replace(/1/g, '!');
}
function inputReplaceBang(s) {
  // accepts ! as 1 for decryption input
  return s.replace(/!/g, '1');
}

async function encryptLNZT(plaintext, passphrase, salt='') {
  let message = (plaintext || '') .toUpperCase().replace(/[^A-Z0-9]/g,'');
  const nonceHex = ''; // nonce removed — deterministic output based only on passphrase+salt
  const baseSeed = await deriveSeedBytes(passphrase, salt);
  const basePerm = await seededShuffle([...Array(M).keys()], baseSeed);
  const substituteMap = Array.from({length:M}, (_,i)=>basePerm[i]);
  // substitution
  let indices = Array.from(message).map(ch=>substituteMap[indexOfSym(ch)]);
  // additive keystream
  const ks = await deriveKeystream(passphrase, nonceHex, indices.length);
  indices = indices.map((v,i)=> (v + ks[i]) % M);
  // nibble mixing (base-6)
  const nibbleKs = await deriveKeystream(passphrase+':nibble', nonceHex, indices.length*2);
  const newIndices = [];
  for (let i=0;i<indices.length;i++) {
    let d0 = Math.floor(indices[i]/6);
    let d1 = indices[i] % 6;
    d0 = (d0 + (nibbleKs[2*i] % 6)) % 6;
    d1 = (d1 + (nibbleKs[2*i+1] % 6)) % 6;
    const h = await sha256Bytes(passphrase + '::swap' + '::' + nonceHex + '::' + i);
    if (h[0] & 1) { [d0,d1] = [d1,d0]; }
    newIndices.push(d0*6 + d1);
  }
  indices = newIndices;
  // block transposition per-block-length
  const blockSize = (baseSeed[0] % 5) + 4; // 4..8
  const outIndices = [];
  for (let bstart=0;bstart<indices.length;bstart+=blockSize) {
    const block = indices.slice(bstart, Math.min(indices.length, bstart+blockSize));
    const L = block.length;
    const permSeed = await deriveSeedBytes(passphrase, salt + '::perm::L' + L);
    const perm = await seededShuffle([...Array(L).keys()], permSeed);
    const transposed = Array.from({length:L}, (_,i)=>block[perm[i]]);
    outIndices.push(...transposed);
  }
  // CBC-like diffusion
  const final = outIndices.slice();
  let prev = baseSeed[0] % M;
  for (let i=0;i<final.length;i++) {
    final[i] = (final[i] + prev) % M;
    prev = final[i];
  }
  const ciphertext = final.map(symOf).join('');
  return {ciphertext};
}

async function decryptLNZT(ciphertext, passphrase, salt) {
  let messageRaw = (ciphertext || '').toUpperCase();
  // accept '!' as '1'
  messageRaw = inputReplaceBang(messageRaw);
  let message = messageRaw.replace(/[^A-Z0-9]/g,'');
  const nonceHex = '';
  const indices = Array.from(message).map(ch=>indexOfSym(ch));
  const baseSeed = await deriveSeedBytes(passphrase, salt);
  // reverse CBC-like diffusion
  const recovered = indices.slice();
  let prev = baseSeed[0] % M;
  for (let i=0;i<recovered.length;i++) {
    const c = recovered[i];
    const orig = (c - prev + M) % M;
    recovered[i] = orig;
    prev = c;
  }
  // reverse block transposition
  const blockSize = (baseSeed[0] % 5) + 4;
  let outIndices = [];
  for (let bstart=0;bstart<recovered.length;bstart+=blockSize) {
    const block = recovered.slice(bstart, Math.min(recovered.length,bstart+blockSize));
    const L = block.length;
    const permSeed = await deriveSeedBytes(passphrase, salt + '::perm::L' + L);
    const perm = await seededShuffle([...Array(L).keys()], permSeed);
    const inv = Array(L).fill(0);
    for (let i=0;i<L;i++) inv[perm[i]] = i;
    const untrans = Array.from({length:L}, (_,i)=>block[inv[i]]);
    outIndices.push(...untrans);
  }
  // reverse nibble mixing
  const nibbleKs = await deriveKeystream(passphrase+':nibble', nonceHex, outIndices.length*2);
  const afterNibble = [];
  for (let i=0;i<outIndices.length;i++) {
    let d0 = Math.floor(outIndices[i]/6);
    let d1 = outIndices[i] % 6;
    const h = await sha256Bytes(passphrase + '::swap' + '::' + nonceHex + '::' + i);
    if (h[0] & 1) { [d0,d1] = [d1,d0]; }
    d0 = (d0 - (nibbleKs[2*i] % 6) + 6) % 6;
    d1 = (d1 - (nibbleKs[2*i+1] % 6) + 6) % 6;
    afterNibble.push(d0*6 + d1);
  }
  // reverse additive keystream
  const ks = await deriveKeystream(passphrase, nonceHex, afterNibble.length);
  const afterKS = afterNibble.map((v,i)=> (v - ks[i] + M) % M);
  // reverse substitution
  const basePerm = await seededShuffle([...Array(M).keys()], baseSeed);
  const inverseSub = Array(M).fill(0);
  for (let i=0;i<M;i++) inverseSub[basePerm[i]] = i;
  const recoveredPlainIndices = afterKS.map(i=>inverseSub[i]);
  const plaintext = recoveredPlainIndices.map(symOf).join('');
  return plaintext;
}

// UI wiring
const $ = id => document.getElementById(id);

$('encrypt').addEventListener('click', async ()=>{
  const pt = $('plaintext').value;
  const pw = $('passphrase').value;
  const salt = $('salt').value || '';
  try {
    const {ciphertext} = await encryptLNZT(pt, pw, salt);
    // display with 1 -> !
    $('ciphertext').textContent = displayReplaceOnes(ciphertext);
    // also show plaintext area with replaced ones for visual consistency
    $('plaintext').value = displayReplaceOnes(pt);
  } catch (err) {
    $('ciphertext').textContent = 'Error: '+err.message;
  }
});

$('decrypt').addEventListener('click', async ()=>{
  const rawCt = $('ciphertext').textContent || '';
  const ctForInput = inputReplaceBang(rawCt); // convert ! -> 1 for processing
  const pw = $('passphrase').value;
  const salt = $('salt').value || '';
  try {
    const pt = await decryptLNZT(ctForInput, pw, salt);
    // display plaintext with 1 -> ! for UI
    $('plaintext').value = displayReplaceOnes(pt);
  } catch (err) {
    alert('Decryption error: '+err.message);
  }
});

$('copyCipher').addEventListener('click', ()=>{
  const text = $('ciphertext').textContent || '';
  navigator.clipboard.writeText(text);
});
$('copyPlain').addEventListener('click', ()=>{
  navigator.clipboard.writeText($('plaintext').value);
});
$('download').addEventListener('click', ()=>{
  const blob = new Blob([document.documentElement.outerHTML], {type:'text/html'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url; a.download = 'lnzt_cipher_demo_passphrase_only_1to!.html'; a.click(); URL.revokeObjectURL(url);
});

</script>
</body>
</html>
