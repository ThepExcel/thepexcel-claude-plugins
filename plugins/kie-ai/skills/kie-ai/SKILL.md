---
name: kie-ai
description: Execution skill สำหรับ Kie.ai API (Grok Imagine ผ่าน kie.ai proxy) — รับ prompt จาก /art-tech-engineer แล้ว gen ภาพ/วีดีโอผ่าน Kie.ai REST API รองรับ text-to-image, image-to-image, image-to-video, และ upscale ใช้เมื่อถูกระบุให้ใช้ Kie.ai โดยตรง หรือเมื่อ orchestrator delegate มา
model: claude-sonnet-4-20250514
---

# kie-ai — Grok Imagine ผ่าน Kie.ai API

Gen image/video ผ่าน Kie.ai REST API — proxy สำหรับ Grok Imagine models

## Path Shortcuts

```
OUT    = D:/ClaudeMediaGen/output-fal   # ใช้ร่วมกับ fal output
BASE   = https://api.kie.ai/api/v1/jobs
```

## Environment Variables

| Var | ที่อยู่ | วิธีอ่าน |
|-----|--------|---------|
| `KIE_API_KEY` | Windows User Env | `os.environ["KIE_API_KEY"]` (set via `setx` หรือ `[Environment]::SetEnvironmentVariable`) |

## Model Selection

| Model ID | Type | Best For | Output |
|----------|------|----------|--------|
| `grok-imagine/text-to-image` | text2img | Aesthetic/cinematic images, Thai prompt ได้ดี | 3 รูป/request |
| `grok-imagine/image-to-image` | img2img | แก้ไข/ปรับ style จากรูปต้นแบบ | TBD |
| `grok-imagine/image-to-video` | img2vid | สร้างวีดีโอจากรูปนิ่ง | TBD |
| `grok-imagine/upscale` | upscale | เพิ่ม resolution วีดีโอ | TBD |
| `veo3_fast` | video | Video fast, รองรับ i2v (REFERENCE_2_VIDEO) | video + audio |
| `veo3_quality` | video | Video quality สูงสุด | video + audio |
| `veo3_lite` | video | Video ประหยัดสุด | video + audio |

> **หมายเหตุ:** text-to-image + veo3_fast ทดสอบแล้ว ใช้งานได้จริง / model อื่นๆ มีใน docs แต่ยังไม่ได้ทดสอบ
> **Veo3 ใช้คนละ endpoint** — ดู section "Veo3 Video Generation" และ "API Endpoints Summary" ด้านล่าง

## Aspect Ratios

| Value | ใช้เมื่อ |
|-------|---------|
| `2:3` | Portrait (default สำหรับรูปคน) |
| `3:2` | Landscape |
| `1:1` | Square |
| `16:9` | Wide cinematic |
| `9:16` | Story/Reels format |

## API Flow — 3 ขั้นตอน

### 1. Create Task

```
POST {BASE}/createTask
Authorization: Bearer <KIE_API_KEY>
Content-Type: application/json

Body:
{
  "model": "grok-imagine/text-to-image",
  "input": {
    "prompt": "...",
    "aspect_ratio": "2:3"
  }
}

Response: { "code": 200, "data": { "taskId": "..." } }
```

- ตรวจ `code == 200` ก่อน ถ้าไม่ใช่ = error
- เก็บ `taskId` ไว้ poll

### 2. Poll Task Status

```
GET {BASE}/recordInfo?taskId=<taskId>
Authorization: Bearer <KIE_API_KEY>
```

**Task States:**

| State | ความหมาย | Action |
|-------|---------|--------|
| `waiting` | อยู่ในคิว | รอต่อ |
| `queuing` | อยู่ในคิว | รอต่อ |
| `generating` | กำลัง gen | รอต่อ |
| `success` | เสร็จแล้ว | ดึง URL |
| `fail` | ล้มเหลว | อ่าน `failMsg` + `failCode` |

**Poll strategy:** ทุก 5 วินาที, timeout 5 นาที (60 รอบ)

**เมื่อ success:**
- `data.resultJson` เป็น JSON string → ต้อง parse อีกรอบ
- ภายในมี `resultUrls: [url1, url2, url3]` — ได้ 3 รูปต่อ 1 request

### 3. Download Images

**สำคัญ:** URL ต้องใส่ headers พิเศษ ไม่งั้นจะโดน block

```
Headers สำหรับ download:
  User-Agent: Mozilla/5.0
  Referer: https://kie.ai/
```

**Naming convention:**
```
kie_grok_{taskId_last8}_{index}.jpg
```

**Output directory:** `D:/ClaudeMediaGen/output-fal/`

---

## Prompt Rules

> Grok prompt syntax rules have moved to the shared model-card.
> See [grok-image.md](../art-engine/references/model-cards/grok-image.md) for:
> - Hybrid prompt strategy (English + Thai)
> - Forbidden words
> - Subject short / Scene long rule
> - ห้ามบอก body size

## Implementation Guide

### สร้าง gen script ใน Python

1. **อ่าน API key** จาก `os.environ["KIE_API_KEY"]` (Windows User Env — set ครั้งเดียวทั่วเครื่อง)
2. **Create task** → POST `/createTask` → เก็บ `taskId`
3. **Poll** → GET `/recordInfo?taskId=` → วน loop ทุก 5s จนได้ `success` หรือ `fail`
4. **Parse result** → `data.resultJson` เป็น string → `json.loads()` → `resultUrls`
5. **Download** → ใส่ `User-Agent` + `Referer` headers → save เป็น `.jpg`
6. **สร้าง .txt คู่** → save prompt text ข้างรูปทุกครั้ง (เพื่อ traceability)

### Batch Generation

- ส่ง request หลายตัวพร้อมกันได้ (ไม่มี rate limit ที่ทดสอบเจอ)
- แต่ละ request ได้ 3 รูป → 10 requests = 30 รูป
- ใช้ `run_in_background: true` สำหรับ parallel gen ใน Claude Code

### Error Handling

- `code != 200` จาก createTask → แสดง error ทั้ง response
- `state == "fail"` → อ่าน `data.failMsg` + `data.failCode`
- Network error ขณะ poll → retry ได้เลย ไม่ต้อง re-create task
- Download fail → retry พร้อม headers

## Cost

- ยังไม่มีข้อมูล cost per request จาก Kie.ai — ใช้ preview ก่อน batch เสมอ

## Rules

- **Preview ก่อน batch** — gen 1 request ก่อน (ได้ 3 รูป) ดู quality ก่อน gen เพิ่ม
- **ห้าม Read รูปที่ gen เสร็จ** — เปลือง context มาก แจ้งแค่ path ไฟล์พอ
- **.txt คู่เสมอ** — save prompt text คู่กับรูปทุกครั้ง
- **ห้าม Read output file ของ background task** — สั่ง gen แล้วจบเลย

## File Upload API (ทดสอบแล้ว)

ใช้ upload ไฟล์ (เช่น รูปสำหรับ image-to-video) ก่อนส่ง gen

```
POST https://kieai.redpandaai.co/api/file-stream-upload
Authorization: Bearer <KIE_API_KEY>
Content-Type: multipart/form-data

Parameters:
  file: (binary)
  uploadPath: (string)
  fileName: (string)

Response:
{
  "success": true,
  "data": {
    "downloadUrl": "https://tempfile.redpandaai.co/..."
  }
}
```

**หมายเหตุ:**
- ใช้ curl multipart เท่านั้น (base64 upload ได้ 403)
- ไฟล์ถูกลบอัตโนมัติหลัง 3 วัน
- ต้อง upload ก่อนใช้ image-to-video (Veo3 REFERENCE_2_VIDEO)

## Veo3 Video Generation (ทดสอบแล้ว)

> **Veo3 ใช้คนละ endpoint กับ Market API (Grok Imagine)** — ทั้ง generate และ poll

### Create Task

```
POST https://api.kie.ai/api/v1/veo/generate
Authorization: Bearer <KIE_API_KEY>
Content-Type: application/json

Body:
{
  "prompt": "...",
  "imageUrls": ["https://uploaded-url"],
  "model": "veo3_fast",
  "generationType": "REFERENCE_2_VIDEO",
  "aspect_ratio": "9:16"
}
```

| Parameter | ค่า | หมายเหตุ |
|-----------|-----|---------|
| `model` | `veo3_fast`, `veo3_quality`, `veo3_lite` | fast รองรับ i2v |
| `generationType` | `TEXT_2_VIDEO`, `REFERENCE_2_VIDEO` | REFERENCE_2_VIDEO ใช้ได้เฉพาะ veo3_fast |
| `aspect_ratio` | `16:9`, `9:16`, `Auto` | — |
| `imageUrls` | array of URLs | ต้อง upload ก่อนผ่าน file-stream-upload |

Optional: `enableTranslation`, `enableFallback`, `watermark`, `callBackUrl`, `seeds`

### Poll Status

```
GET https://api.kie.ai/api/v1/veo/record-info?taskId=<taskId>
Authorization: Bearer <KIE_API_KEY>
```

| Field | ค่า | ความหมาย |
|-------|-----|---------|
| `successFlag` | `0` | กำลัง generate |
| `successFlag` | `1` | สำเร็จ |
| `successFlag` | `2` | ล้มเหลว |
| `successFlag` | `3` | ล้มเหลว |

- Video URL อยู่ใน `data.response.resultUrls[]`
- มี audio (`hasAudioList: [true]`)
- ใช้เวลาประมาณ 60-120 วินาที

### Veo3 Flow: Upload → Generate → Poll

```
1. Upload image  → POST file-stream-upload → ได้ downloadUrl
2. Generate      → POST /veo/generate (ใส่ downloadUrl ใน imageUrls) → ได้ taskId
3. Poll          → GET /veo/record-info?taskId= → วน loop จน successFlag != 0
4. Download      → ดึง video จาก resultUrls[]
```

## API Endpoints Summary

| Feature | Market API (Grok Imagine) | Veo3 API |
|---------|--------------------------|----------|
| Create task | `POST /api/v1/jobs/createTask` | `POST /api/v1/veo/generate` |
| Poll status | `GET /api/v1/jobs/recordInfo` | `GET /api/v1/veo/record-info` |
| File upload | `POST kieai.redpandaai.co/api/file-stream-upload` | (same) |
| Base URL | `https://api.kie.ai` | `https://api.kie.ai` |

## Related Skills

- `/art-engine` - upstream: model selection and prompt syntax translation
- `/fal-ai` - alternative execution path for Grok (same model, different API - fal.ai direct vs Kie.ai proxy)
- `/art-director` - upstream: creative brief crafting
- `/sira-image-prefer` - taste DNA for prompt preferences
