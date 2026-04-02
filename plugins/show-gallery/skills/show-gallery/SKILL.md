---
name: show-gallery
description: "Universal media gallery — browse images/videos from any local folder with copy-path, enlarge, and video playback. Reusable across all gen projects."
model: claude-sonnet-4-20250514
---

# /show-gallery — Universal Media Browser

Browse generated images & videos from any local folder. Click to enlarge, copy path to clipboard.

## Quick Flow

```
พี่ระ: "โชว์รูปใน output-fal/" หรือ "show gallery D:/path/to/folder"
→ สร้าง gallery.html ใน target folder
→ preview_start server ที่ serve folder นั้น
→ navigate to /gallery.html
→ screenshot ให้พี่ระดู
```

## เมื่อถูกเรียก

### Step 1: ระบุ folder

- ถ้าพี่ระระบุ path → ใช้เลย
- ถ้าไม่ระบุ → ถามว่า folder ไหน หรือใช้ folder ล่าสุดที่ gen

### Step 2: สร้าง gallery.html

ใช้ **Gallery HTML Template** ด้านล่าง — Write ไฟล์ `gallery.html` ลงใน target folder
**ต้องแก้ `GALLERY_ROOT`** ให้เป็น absolute path ของ folder (ใช้ `/` ไม่ใช่ `\`)

### Step 3: เปิด Preview

1. เช็ค `.claude/launch.json` ว่ามี server ที่ serve folder นี้อยู่หรือยัง
2. ถ้าไม่มี → เพิ่ม config ใหม่ (port ถัดไป เช่น 8125, 8126...)
3. `preview_start` → `preview_eval: window.location.href = '/gallery.html'`
4. รอ 1-2 วิ → `preview_screenshot` ให้พี่ระดู

### Step 4: Interaction

พี่ระจะ copy path จากหน้า gallery แล้ว paste มาใน chat บอกว่าจะทำอะไรกับรูป/วิดีโอนั้น

---

## Gallery HTML Template

> **Copy ทั้ง block ด้านล่าง** แล้ว Write เป็น `gallery.html` ใน target folder
> แก้ `GALLERY_ROOT` บรรทัดเดียว

```html
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="UTF-8">
<title>Gallery</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
body{background:#1a1a1a;color:#fff;font-family:system-ui,sans-serif;padding:12px}
.header{display:flex;align-items:center;justify-content:space-between;margin-bottom:12px;flex-wrap:wrap;gap:8px}
.header h1{font-size:14px;color:#aaa;font-weight:400}
.header .count{font-size:13px;color:#666}
.controls{display:flex;gap:6px;align-items:center}
.controls button{background:#333;border:1px solid #555;color:#ccc;padding:4px 10px;border-radius:4px;cursor:pointer;font-size:12px}
.controls button:hover{background:#444}
.controls button.active{background:#555;color:#fff;border-color:#888}
.grid{display:grid;gap:8px}
.grid.cols-2{grid-template-columns:repeat(2,1fr)}
.grid.cols-3{grid-template-columns:repeat(3,1fr)}
.grid.cols-4{grid-template-columns:repeat(4,1fr)}
.card{position:relative;border-radius:6px;overflow:hidden;background:#222;cursor:pointer}
.card:hover{outline:2px solid #ffd700}
.card img,.card video{width:100%;display:block;aspect-ratio:2/3;object-fit:cover}
.card .info{position:absolute;bottom:0;left:0;right:0;background:linear-gradient(transparent,rgba(0,0,0,.85));padding:6px 8px}
.card .fname{font-size:11px;color:#ddd;word-break:break-all}
.card .meta{font-size:10px;color:#888;margin-top:2px}
.card .copy-btn{position:absolute;top:6px;right:6px;background:rgba(0,0,0,.7);border:1px solid #555;color:#ccc;padding:3px 8px;border-radius:4px;font-size:11px;cursor:pointer;opacity:0;transition:opacity .2s}
.card:hover .copy-btn{opacity:1}
.card .copy-btn.copied{background:#2a5a2a;border-color:#4a4;color:#8f8}
.card .badge{position:absolute;top:6px;left:6px;background:rgba(0,0,0,.7);color:#ffd700;padding:2px 6px;border-radius:3px;font-size:10px;font-weight:700}
.modal{display:none;position:fixed;top:0;left:0;right:0;bottom:0;background:rgba(0,0,0,.95);z-index:100;justify-content:center;align-items:center;flex-direction:column;cursor:pointer}
.modal.active{display:flex}
.modal img,.modal video{max-height:85vh;max-width:95vw;object-fit:contain;border-radius:6px}
.modal .path-bar{margin-top:8px;padding:6px 14px;background:#333;border-radius:4px;font-size:12px;color:#aaa;cursor:pointer;user-select:all}
.modal .path-bar:hover{background:#444;color:#fff}
.modal .caption{margin-top:6px;font-size:11px;color:#888;max-width:80vw;text-align:center;max-height:60px;overflow:auto}
.toast{position:fixed;bottom:20px;left:50%;transform:translateX(-50%);background:#2a5a2a;color:#8f8;padding:8px 20px;border-radius:6px;font-size:13px;opacity:0;transition:opacity .3s;z-index:200;pointer-events:none}
.toast.show{opacity:1}
</style>
</head>
<body>

<div class="header">
  <h1 id="title">Gallery</h1>
  <span class="count" id="count"></span>
  <div class="controls">
    <button onclick="setCols(2)">2</button>
    <button onclick="setCols(3)" class="active">3</button>
    <button onclick="setCols(4)">4</button>
    <span style="color:#555">|</span>
    <button onclick="filterType('all')" class="active" id="btn-all">All</button>
    <button onclick="filterType('image')" id="btn-image">IMG</button>
    <button onclick="filterType('video')" id="btn-video">VID</button>
  </div>
</div>
<div class="grid cols-3" id="grid"></div>

<div class="modal" id="modal">
  <img id="modal-img" style="display:none">
  <video id="modal-vid" controls style="display:none"></video>
  <div class="path-bar" id="modal-path" onclick="event.stopPropagation();copyPath(this.dataset.path)"></div>
  <div class="caption" id="modal-caption"></div>
</div>

<div class="toast" id="toast">Copied!</div>

<script>
// ====== CONFIG — แก้ตรงนี้ ======
const GALLERY_ROOT = 'D:/ClaudeMediaGen/output-fal';
// =================================

const IMG_EXT = /\.(png|jpg|jpeg|webp|gif)$/i;
const VID_EXT = /\.(mp4|webm|mov)$/i;
let allItems = [];
let currentFilter = 'all';
const captions = {};

async function loadGallery() {
  const resp = await fetch('./');
  const html = await resp.text();
  const links = [...html.matchAll(/href="([^"]+)"/g)].map(m => decodeURIComponent(m[1]));
  const mediaFiles = links.filter(f => IMG_EXT.test(f) || VID_EXT.test(f)).sort();

  // Load .txt captions
  const txtFiles = links.filter(f => f.endsWith('.txt'));
  await Promise.all(txtFiles.map(async t => {
    try {
      const r = await fetch(t);
      captions[t.replace('.txt','')] = (await r.text()).slice(0, 200);
    } catch(e) {}
  }));

  allItems = mediaFiles.map(f => {
    const isVid = VID_EXT.test(f);
    const baseName = f.replace(/\.[^.]+$/, '');
    return { file: f, isVideo: isVid, caption: captions[baseName] || '' };
  });

  renderGrid();
}

function renderGrid() {
  const grid = document.getElementById('grid');
  const filtered = currentFilter === 'all' ? allItems
    : currentFilter === 'image' ? allItems.filter(i => !i.isVideo)
    : allItems.filter(i => i.isVideo);

  document.getElementById('count').textContent =
    `${filtered.length}/${allItems.length} items (${allItems.filter(i=>!i.isVideo).length} img, ${allItems.filter(i=>i.isVideo).length} vid)`;

  grid.innerHTML = '';
  filtered.forEach((item, i) => {
    const card = document.createElement('div');
    card.className = 'card';
    const absPath = GALLERY_ROOT + '/' + item.file;
    const sizeLabel = item.isVideo ? 'VIDEO' : '';

    if (item.isVideo) {
      card.innerHTML = `
        <video src="${item.file}" muted preload="metadata"></video>
        ${sizeLabel ? `<span class="badge">${sizeLabel}</span>` : ''}
        <button class="copy-btn" onclick="event.stopPropagation();copyPath('${absPath}',this)">Copy Path</button>
        <div class="info"><div class="fname">${item.file}</div></div>`;
    } else {
      card.innerHTML = `
        <img src="${item.file}" loading="lazy">
        <button class="copy-btn" onclick="event.stopPropagation();copyPath('${absPath}',this)">Copy Path</button>
        <div class="info"><div class="fname">${item.file}</div></div>`;
    }

    card.onclick = () => openModal(item, absPath);
    grid.appendChild(card);
  });

  if (filtered.length === 0) {
    grid.innerHTML = '<p style="grid-column:1/-1;text-align:center;color:#666;padding:40px">No media files found</p>';
  }
}

function openModal(item, absPath) {
  const modal = document.getElementById('modal');
  const mImg = document.getElementById('modal-img');
  const mVid = document.getElementById('modal-vid');
  const mPath = document.getElementById('modal-path');
  const mCap = document.getElementById('modal-caption');

  if (item.isVideo) {
    mImg.style.display = 'none';
    mVid.style.display = 'block';
    mVid.src = item.file;
    mVid.play();
  } else {
    mVid.style.display = 'none';
    mVid.pause();
    mImg.style.display = 'block';
    mImg.src = item.file;
  }

  mPath.textContent = absPath;
  mPath.dataset.path = absPath;
  mCap.textContent = item.caption;
  modal.classList.add('active');
  modal.onclick = (e) => {
    if (e.target === modal) { modal.classList.remove('active'); mVid.pause(); }
  };
}

function copyPath(path, btn) {
  navigator.clipboard.writeText(path).then(() => {
    showToast('Copied: ' + path);
    if (btn) { btn.textContent = 'Copied!'; btn.classList.add('copied'); setTimeout(() => { btn.textContent = 'Copy Path'; btn.classList.remove('copied'); }, 1500); }
  });
}

function showToast(msg) {
  const t = document.getElementById('toast');
  t.textContent = msg;
  t.classList.add('show');
  setTimeout(() => t.classList.remove('show'), 2000);
}

function setCols(n) {
  const grid = document.getElementById('grid');
  grid.className = 'grid cols-' + n;
  document.querySelectorAll('.controls button').forEach(b => {
    if (['2','3','4'].includes(b.textContent)) b.classList.toggle('active', b.textContent == n);
  });
}

function filterType(type) {
  currentFilter = type;
  ['all','image','video'].forEach(t => {
    document.getElementById('btn-'+t)?.classList.toggle('active', t === type);
  });
  renderGrid();
}

loadGallery();
</script>
</body>
</html>
```

---

## Prompt Rules (สำหรับ Claude เมื่อ invoke skill นี้)

1. **สร้าง gallery.html** ใน target folder — copy template จากด้านบน
2. **แก้ `GALLERY_ROOT`** ให้ตรงกับ absolute path ของ folder (forward slashes)
3. **แก้ `<title>`** ให้สื่อกับเนื้อหา
4. **เช็ค launch.json** — ถ้าไม่มี server ที่ serve folder นี้ → เพิ่ม config ใหม่
5. **`preview_start`** → navigate `/gallery.html` → **`preview_screenshot`** ให้พี่ระดู
6. ถ้า screenshot timeout → ลอง `preview_snapshot` แทน แล้วบอกพี่ระเปิดดูใน Preview panel

## Browse Folder Mode

เมื่อพี่ระบอก "browse folder" หรือ "ดู folder" โดยไม่ระบุ path:
- แสดง list ของ output folders ที่มี media files
- ให้พี่ระเลือก folder
- แล้วสร้าง gallery ตาม flow ปกติ

## Related Skills

- `/gen-character-image` — ใช้ gallery หลัง gen character
- `/kie-ai` — ใช้ gallery หลัง gen จาก Kie.ai
- `/fal-ai` — ใช้ gallery หลัง gen จาก fal.ai
- `/comfyui-user` — ใช้ gallery หลัง gen จาก ComfyUI
- `/image-analysis` — วิเคราะห์รูปที่เลือกจาก gallery
