---
feature: CCA Registration Rules — Academic Year Override Bug
slug: cca-registration-rules-ay-override-bug
type: bug-fixing
status: draft
author: BBS Team
date: 2026-07-31
target_release: TBD
---

# CCA Registration Rules — Academic Year Override Bug

## Bug Description

Saat mengedit CCA Registration Rule yang sudah ada, field **Academic Year** di form selalu ter-override ke **current academic year**, bukan menampilkan AY dari rule yang sedang diedit.

## Impact

- **Create mode:** Tidak terpengaruh (default ke current AY — expected behavior)
- **Edit mode:** User mengedit rule AY 2025/2027, tapi form menampilkan dan menyimpan current AY (misal 2026/2028), sehingga data rule berubah AY-nya tanpa disadari
- **Data integrity:** Rule bisa tersimpan dengan AY yang salah karena user tidak sadar bahwa AY telah berubah

## Root Cause

**File:** `bbs/client/src/views/ccaRegistrationRules/CCARegistrationRulesForm.jsx`

Ada **dua useEffect yang conflict**:

### useEffect 1 (line 72-76) — Override AY ke current AY:
```jsx
useEffect(() => {
  if (currentAy?.id) {
    setValue("academicYearId", currentAy?.id);  // <-- selalu override
  }
}, [currentAy]);
```

### useEffect 2 (line 78-106) — Load data rule yang diedit, lalu reset form:
```jsx
useEffect(() => {
  if (id) {
    dispatch(fromApi.getCcaRegistrationRule(id))
      .then((response) => {
        const rules = response.data?.find(...)?.attributes || {};
        reset({
          academicYearId: rules.academicYearId || "",  // <-- di-reset ke AY asli
          ...
        });
      });
  }
}, [id, dispatch, reset]);
```

**Urutan eksekusi:**
1. `currentAy` sudah tersedia di Redux saat component mount
2. useEffect 1 berjalan → `setValue("academicYearId", currentAy.id)` (override ke current AY)
3. useEffect 2 berjalan → fetch data rule, `reset()` dengan AY asli
4. Tapi useEffect 1 **berjalan lagi** saat ada perubahan `currentAy` (atau re-render lain), sehingga **menimpa AY yang sudah di-reset**

### Bug yang Sama dengan LEAPS

Ini adalah pola bug yang identik dengan yang ditemukan di `leaps-academic-year-selectable`:
- `LeapsForm.jsx` line 74-78: `useEffect` yang selalu `setValue("academicYearId", currentAyear?.id)`
- `CCARegistrationRulesForm.jsx` line 72-76: `useEffect` yang selalu `setValue("academicYearId", currentAy?.id)`

## Steps to Reproduce

1. Buka halaman CCA Registration Rules di admin portal
2. Buat rule baru dengan Academic Year "2025/2027"
3. Klik **Edit** pada rule yang baru dibuat
4. **Terlihat:** Field Academic Year menampilkan current AY, bukan "2025/2027"
5. Klik **Update** tanpa mengubah apapun
6. Rule tersimpan dengan Academic Year yang berbeda (current AY)

## Scope

### In Scope
- Fix `useEffect` override AY di `CCARegistrationRulesForm.jsx` — hanya set default AY di **create mode**, jangan override di edit mode
- Verifikasi tidak ada regression di create mode (default AY tetap current AY)

### Out of Scope
- Perubahan di backend (`api_nest/`)
- Perubahan di halaman list (`CCARegistrationRules.jsx`) — sudah ok
- Perubahan di LEAPS form (sudah ada brief terpisah)

## Fix

### File: `bbs/client/src/views/ccaRegistrationRules/CCARegistrationRulesForm.jsx`

**Change 1 — Adjust useEffect AY override (line 72-76):**

```diff
 useEffect(() => {
-  if (currentAy?.id) {
-    setValue("academicYearId", currentAy?.id);
-  }
-}, [currentAy]);
+  // Set default ke current AY hanya di create mode (bukan edit mode)
+  if (currentAy?.id && !id) {
+    setValue("academicYearId", currentAy?.id);
+  }
+}, [currentAy, id]);
```

## API Changes

**Tidak ada.** API sudah benar — `GET /api/v1/ccaRegistrationRules/:id` mengembalikan `academicYearId` yang sesuai.

## Database Changes

**Tidak ada.** Data di database sudah benar — bug hanya di FE yang override field form.

## Testing Steps

1. **Create mode:** Buka `/cca-registration-rules/add` → AY default ke current AY → bisa diganti → submit sukses
2. **Edit mode:** Buka `/cca-registration-rules/:id/edit` → AY menampilkan AY dari rule yang diedit → bisa diganti → submit sukses
3. **Regression:** Pastikan tidak ada error di form lain yang menggunakan `currentAy`