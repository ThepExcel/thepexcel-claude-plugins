---
name: skill-compatibility-test
description: ทดสอบว่า skill ใช้งานได้ครบแค่ไหนในแต่ละ Claude product (Chat / Cowork / Code)
---

# Skill Compatibility Test

ทดสอบ tools ทีละตัวตามลำดับ แล้วสรุปผลเป็นตาราง

## ขั้นตอน

ทำทีละข้อ ถ้าข้อไหนทำได้ → ✅ ถ้าทำไม่ได้ (ไม่มี tool / error) → ❌ ถ้าทำได้บางส่วน → ⚠️

### Test 1: Bash
รันคำสั่งนี้ด้วย Bash tool:
```
echo "COMPATIBILITY_TEST: Bash works"
```

### Test 2: Read
อ่านไฟล์ `$CLAUDE_SKILL_DIR/test-data.txt` ด้วย Read tool

### Test 3: Write
สร้างไฟล์ `/tmp/skill-compat-test-output.txt` ด้วย Write tool เขียนว่า "Test passed"

### Test 4: Grep
ค้นหาคำว่า "COMPATIBILITY_TEST" ในไฟล์ `$CLAUDE_SKILL_DIR/test-data.txt` ด้วย Grep tool

### Test 5: Glob
ค้นหาไฟล์ `*.txt` ใน `$CLAUDE_SKILL_DIR/` ด้วย Glob tool

### Test 6: Agent (subagent)
Spawn subagent ด้วย Agent tool ให้ตอบว่า "Subagent works" กลับมา

### Test 7: WebSearch
ค้นหาคำว่า "Anthropic Claude" ด้วย WebSearch tool

### Test 8: $ARGUMENTS
แสดงค่า `$ARGUMENTS` ที่ได้รับ (ถ้ามี)

### Test 9: $CLAUDE_SKILL_DIR
แสดง path ของ `$CLAUDE_SKILL_DIR`

## สรุปผล

หลังทดสอบครบ ให้สรุปเป็นตารางนี้:

```
| # | Test              | ผลลัพธ์ | หมายเหตุ |
|---|-------------------|---------|---------|
| 1 | Bash              |         |         |
| 2 | Read              |         |         |
| 3 | Write             |         |         |
| 4 | Grep              |         |         |
| 5 | Glob              |         |         |
| 6 | Agent (subagent)  |         |         |
| 7 | WebSearch          |         |         |
| 8 | $ARGUMENTS         |         |         |
| 9 | $CLAUDE_SKILL_DIR  |         |         |
```

แล้วบอกว่า:
- กำลังรันอยู่บน product อะไร (Chat / Cowork / Code Desktop / Code Terminal)
- ถ้าเป็น Code → tools ทั้งหมดควรผ่าน
- ถ้าเป็น Chat/Cowork → หลายตัวจะใช้ไม่ได้ ซึ่งเป็นเรื่องปกติ
