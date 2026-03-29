---
name: skilljar-scraper
description: Scrape Skilljar courses — extract all lesson text + images into organized folders. Works with any Skilljar-hosted course (e.g., Anthropic Academy). Requires Chrome MCP with an active logged-in session.
triggers:
  - "scrape skilljar"
  - "save course"
  - "download skilljar"
  - "save skilljar"
  - "extract course"
  - "scrape course"
  - "ดึงคอร์ส"
  - "save คอร์ส"
  - "ดาวน์โหลดคอร์ส"
---

# Skilljar Course Scraper

Scrape any Skilljar-hosted course into organized folders with text (content.md) and images per lesson.

## Prerequisites

- **Chrome MCP** connected with a tab open to the Skilljar course
- User must be **logged in** to Skilljar already
- The course overview page must be loaded (URL like `https://<org>.skilljar.com/<course-slug>`)

## Workflow

### Step 1: Navigate to course overview & extract lesson list

Use `read_page` on the course overview page to find all lesson links. Lesson URLs follow the pattern:
```
/<course-slug>/<lesson-id>
```

Extract from the sidebar nav:
- Section headings (group names)
- Lesson names + IDs
- Lesson types (Text, Quiz, Video)

### Step 2: Fetch all lessons in one JS call

Run this JavaScript in the browser tab. It fetches all lesson pages in parallel using the browser's session cookies, extracts `#lesson-body` text + image URLs, and downloads the result as a JSON file.

```javascript
(async () => {
  // REPLACE: lessons array with actual lesson IDs/names from Step 1
  const lessons = [
    {id: 'LESSON_ID', name: 'NN_Lesson_Name'},
    // ... one entry per lesson
  ];
  const base = '/COURSE-SLUG/';  // REPLACE with actual course slug

  async function extractLesson(lesson) {
    try {
      const resp = await fetch(base + lesson.id);
      const html = await resp.text();
      const parser = new DOMParser();
      const doc = parser.parseFromString(html, 'text/html');
      const body = doc.querySelector('#lesson-body');
      if (!body) return { ...lesson, error: 'no #lesson-body', text: '', images: [] };
      const text = body.innerText;
      const images = Array.from(body.querySelectorAll('img'))
        .map(img => ({ src: img.getAttribute('src'), alt: img.getAttribute('alt') || '' }))
        .filter(img => img.src && !img.src.includes('data:'));
      return { ...lesson, text, images, error: null };
    } catch (e) {
      return { ...lesson, error: e.message, text: '', images: [] };
    }
  }

  const results = await Promise.all(lessons.map(extractLesson));
  const blob = new Blob([JSON.stringify(results, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'course_data.json';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
  return `Done! ${results.length} lessons. Errors: ${results.filter(r=>r.error).map(r=>r.name+': '+r.error).join(', ') || 'none'}`;
})()
```

### Step 3: Process JSON into folders

Run the Python processor script to create organized output:

```bash
cd "<OUTPUT_DIR>" && uv run --with requests python process_course.py
```

The script (`process_course.py` template below):
- Reads the downloaded JSON
- Creates one subfolder per lesson (e.g., `01_Lesson_Name/`)
- Saves `content.md` with full lesson text
- Downloads all images with sequential filenames based on alt text

### Step 4: Verify

```bash
find "<OUTPUT_DIR>" -type f | sort
```

Check all lessons have `content.md` and expected images.

## Python Processor Script Template

Save this as `process_course.py` in the output directory. Adjust `JSON_FILE` path if needed.

```python
import json, os, re, sys, requests
from pathlib import Path

sys.stdout.reconfigure(encoding='utf-8')

BASE_DIR = Path(__file__).parent
JSON_FILE = Path.home() / "Downloads" / "course_data.json"

def sanitize(name):
    return re.sub(r'[^\w\s-]', '', name).strip().replace(' ', '_')

def download_image(url, filepath):
    try:
        resp = requests.get(url, timeout=30)
        resp.raise_for_status()
        filepath.write_bytes(resp.content)
        print(f"  ✓ {filepath.name} ({len(resp.content)//1024}KB)")
        return True
    except Exception as e:
        print(f"  ✗ {filepath.name}: {e}")
        return False

def main():
    with open(JSON_FILE, 'r', encoding='utf-8') as f:
        lessons = json.load(f)
    print(f"Processing {len(lessons)} lessons\n")
    for lesson in lessons:
        name = lesson['name']
        folder = BASE_DIR / name
        folder.mkdir(exist_ok=True)
        text = lesson.get('text', '')
        images = lesson.get('images', [])
        if not text:
            print(f"⚠ {name}: no text"); continue
        (folder / 'content.md').write_text(text, encoding='utf-8')
        print(f"{name}: content.md ({len(text)} chars)")
        for idx, img in enumerate(images):
            src = img['src']
            alt = img.get('alt', '')
            ext = os.path.splitext(src.split('?')[0])[-1] or '.png'
            fname = f"{idx+1:02d}_{sanitize(alt)[:60]}{ext}"
            fp = folder / fname
            if not fp.exists():
                download_image(src, fp)
            else:
                print(f"  - {fname} (exists)")
    print("\nDone!")

if __name__ == '__main__':
    main()
```

## Naming Convention

- Folders: `NN_Lesson_Name` (NN = 2-digit sequence, spaces → underscores)
- Text: `content.md` per lesson
- Images: `NN_Alt_Text_Truncated.png` (NN = order within lesson)

## Key Details

- `#lesson-body` is the Skilljar selector for lesson content area
- `fetch()` from within the browser inherits session cookies — no need to extract/pass cookies manually
- DOMParser runs in-browser so we get clean text extraction
- Quiz lessons may have different structure but `#lesson-body` still works
- Images are typically hosted on `everpath-course-content.s3-accelerate.amazonaws.com`

## Limitations

- Requires active browser session (can't run headless)
- Video content (embedded players) is not downloaded — only text + images
- Interactive elements (quizzes with submission) capture question text only
