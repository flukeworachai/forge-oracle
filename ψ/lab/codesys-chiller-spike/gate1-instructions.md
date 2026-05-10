# Gate 1 — Instructions for fluke

> Live-guided execution. Forge ส่งทีละ batch — fluke ทำ + paste output → Forge verify + ส่ง batch ถัดไป.
> Mirror ของ comment ที่ posted ใน [issue #3](https://github.com/flukeworachai/atlas-oracle/issues/3).

## Batch 1 — Pre-flight + CODESYS Store + downloads

### A. Pre-flight (Windows host)

ตรวจ 4 ข้อก่อนเริ่ม:

1. **Free disk** ≥ 5 GB on `C:` (Dev System ~2 GB + Control Win ~500 MB + project + buffer)
2. **Admin rights** บน Windows (จำเป็นสำหรับ install runtime service)
3. **Port 8080 ว่าง** (WebVisu default) — `netstat -ano | findstr :8080` ใน PowerShell, ถ้าไม่มี output = ว่าง
4. **Antivirus** — ถ้ามี Defender ให้ exclude `C:\Program Files\CODESYS*\` (CODESYS รัน .exe เป็น runtime service, AV จะมา scan + ทำ scan cycle วูบ)

### B. CODESYS Store account

1. ไป https://store.codesys.com
2. คลิก **Register** (มุมขวาบน)
3. กรอก email + password — รับ verification email → ยืนยัน
4. Login เข้าระบบ

### C. Downloads (3 packages)

Login แล้ว search + add to cart (ทุกตัว price = 0 €):

| Package | ใช้ทำอะไร |
|---------|----------|
| **CODESYS Development System V3** | IDE (Windows) |
| **CODESYS Control Win SL** | soft-PLC runtime (30-day trial — เพียงพอสำหรับ spike) |
| **CODESYS Visualization** | HTML5/WebVisu support (ถ้าไม่ bundle ใน Dev System) |

Checkout (zero charge) → Download links ที่ "My Account → My Downloads" → save .exe ทั้งหมดไว้ที่ `C:\Users\flukw\Downloads\codesys-spike\`

### D. Paste back ให้ Forge

ทำเสร็จแล้ว paste 4 อย่างนี้ใน issue #3:

```
[ ] disk free: <X GB>
[ ] admin rights: yes/no
[ ] port 8080: free/used (paste netstat line if used)
[ ] AV exclusion: done/skipped/N-A

Downloads saved (filenames):
- <filename1>
- <filename2>
- <filename3>

Time taken (signup → all downloaded): <minutes>
```

---

**Forge รอตรงนี้** — paste กลับมาที่ issue #3, Forge verify + ส่ง Batch 2 (install + runtime start) ต่อ.

หาก Store login ติด / download ไม่ขึ้น / package หาไม่เจอ → comment พร้อม screenshot, Forge ปรับ path ให้.

— Forge 🔥
