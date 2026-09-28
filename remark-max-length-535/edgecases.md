# Edge Cases — Remark Max Length 535

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

Konteks: perubahan limit remark 837 → **535** di frontend (Teacher Portal, `RemarkDetail.jsx`), dengan pengecekan max-length di frontend yang **tidak boleh** memicu/men-gate autosave (`POST /api/v1/remarks/bulk`, debounce 800ms).

---

## EC-01: Batas Boundary 535 vs 536 Karakter dan Perilaku Autosave

**Scenario:** Teacher mengetik remark tepat **535** karakter (valid) versus **536** karakter (invalid). Setelah debounce 800ms, apa yang dikirim ke backend? Saat ini validasi max-length dijalankan lewat `await trigger()` di dalam `autosaveStudentRef.current` sebelum `dispatch(fromApi.autosaveBulkRemark(...))` — jadi boundary menentukan apakah payload terkirim.

| Opsi | Behavior |
|------|----------|
| (A) 535 = valid → autosave terkirim; 536 = invalid → autosave **di-skip** (tidak ada request), error `Max. 535 characters` tampil. | Perilaku paling ketat: data > 535 tidak pernah persist. Namun autosave baris lain tetap jalan. |
| (B) 535 = valid → autosave terkirim; 536 = invalid → tetap kirim payload, frontend hanya menampilkan error (backend tetap terima jika EC-04 = frontend-only). | Data > 535 bisa persist — bertentangan dengan Business Rule #5. Tidak direkomendasikan. |
| (C) 535 = valid → autosave terkirim; 536 = invalid → autosave di-skip **dan** autosave global (semua baris) ditunda sampai diperbaiki. | Men-gate autosave seluruh form — bertentangan dengan AC-4 (tidak men-gate autosave baris valid). Tidak direkomendasikan. |

**Decision:** _TBD_

---

## EC-02: Satu Baris Invalid, Baris Lain Valid (Decoupling Validasi dari Autosave)

**Scenario:** Halaman remark menampilkan multiple siswa (loop `studentRemark.{i}`) masing-masing dengan beberapa remark (`remarks.{j}`). Siswa A remark-nya > 535 (invalid), Siswa B remark-nya ≤ 535 (valid) dan di-edit. Bagaimana autosave berperilaku agar validasi max-length tidak men-gate save baris valid?

| Opsi | Behavior |
|------|----------|
| (A) Autosave per-student dijalankan independen: baris valid (Siswa B) tetap terkirim setelah debounce; baris invalid (Siswa A) di-skip tanpa menahan baris lain. | Sesuai AC-4 & Business Rule #4. Pengecekan max-length terpisah dari siklus autosave. |
| (B) Autosave ditunda untuk seluruh form selama ada satu baris invalid (single `trigger()` global men-gate semua). | Melanggar AC-4: autosave baris valid tertahan oleh baris invalid. Tidak direkomendasikan. |
| (C) Autosave hanya memproses baris yang sedang diedit, tanpa memvalidasi baris lain — baris invalid lain diabaikan diam-diam. | Valid selama payload yang dikirim tidak invalid. Perlu dipastikan payload hanya berisi baris valid. |

**Decision:** _TBD_

---

## EC-03: Data Lama Tersimpan dengan Panjang 536–837 Karakter

**Scenario:** Sebelum limit diturunkan ke 535, sudah ada remark tersimpan di database dengan panjang 536–837 karakter. Saat teacher membuka halaman remark dan data lama di-load, apa yang terjadi?

| Opsi | Behavior |
|------|----------|
| (A) Data lama **tidak di-truncate/diubah otomatis**; ditampilkan apa adanya. Validasi max-length hanya berlaku saat edit baru. Jika teacher menyimpan tanpa mengubah field itu, tidak ada error paksa (field tidak di-touch). | Direkomendasikan — menghindari kehilangan data tanpa sepengetahuan user. Sesuai Business Rule #7. |
| (B) Data lama di-truncate otomatis ke 535 saat load. | Berisiko kehilangan data secara diam-diam. Tidak direkomendasikan. |
| (C) Data lama ditandai error (memaksa teacher memperbaiki sebelum bisa menyimpan form apa pun). | Teacher terpaksa memotong data lama agar form bisa disubmit — menambah friksi. Perlu keputusan stakeholder. |

**Decision:** _TBD_

---

## EC-04: Penegakan Batas 535 — Frontend-only vs Backend `@MaxLength`

**Scenario:** Saat ini `CreateRemarkDto.remark` hanya punya `@IsString()` + `@IsOptional()` — **tidak ada** `@MaxLength`. Batas 535 murni keputusan frontend. Bagaimana menjamin data > 535 tidak persist?

| Opsi | Behavior |
|------|----------|
| (A) Frontend-only: backend tidak diubah. Batas 535 hanya ditegakkan di frontend (yup + pengecekan inline). Request invalid di-skip frontend (EC-01 opsi A). | Perubahan minimal (scope Out of backend). Risiko: request di luar UI (curl/API langsung) tetap bisa mengirim > 535. |
| (B) Tambah `@MaxLength(535)` di `CreateRemarkDto` (dan berlaku untuk update via `PartialType`) sebagai defense-in-depth. Request invalid → backend balas **400** dengan pesan class-validator bawaan. | Menjamin invariant di backend. Menambah scope backend yang saat ini Out of Scope. Perlu disepakati karena mengubah perilaku API existing (data lama > 535 yang dikirim ulang akan ditolak). |
| (C) Kombinasi: frontend memblokir autosave invalid (A) **dan** backend `@MaxLength(535)` (B). | Paling aman; butuh perubahan frontend + backend. |

**Decision:** _TBD_

---

## EC-05: Character Counter `123 / 535` (Usulan Opsional)

**Scenario:** Spec mengusulkan penambahan counter karakter di bawah textarea agar teacher tahu sisa kuota. `BBSTextArea` di `bbs-client-common` belum memiliki fitur ini dan dipakai consumer lain.

| Opsi | Behavior |
|------|----------|
| (A) Tidak ada counter; hanya pesan error `Max. 535 characters` saat melebihi batas. | Perubahan paling minimal, tidak menyentuh `bbs-client-common`. Sesuai scope saat ini. |
| (B) Counter ditambahkan di `RemarkDetail.jsx` (lokal, di luar `BBSTextArea`) tanpa mengubah komponen common. | Menambah UI lokal; tidak mempengaruhi consumer lain. Perlu desain layout minor. |
| (C) Counter ditambahkan sebagai prop **opt-in** pada `BBSTextArea` (mis. `maxLength`/`showCounter`), default off agar tidak mengubah consumer lain. | Reusable untuk fitur lain, tetapi mengubah komponen shared — perlu persetujuan (Out of Scope saat ini). |

**Decision:** _TBD_

---

## EC-06: Error Muncul Hanya Setelah Blur (mode: "onTouched") vs Saat Mengetik

**Scenario:** `useForm` memakai `mode: "onTouched"` — error max-length baru muncul setelah field di-touch/blur. Bagaimana memastikan feedback muncul **tanpa memicu save** dan tetap sesuai AC-2 ("saat mengetik / onBlur")?

| Opsi | Behavior |
|------|----------|
| (A) Pertahankan `mode: "onTouched"`: error muncul setelah user blur/menyentuh field. Validasi tidak memicu autosave (dipisah dari handler autosave). | Sesuai perilaku existing, perubahan minimal. User melihat error setelah meninggalkan field. |
| (B) Ubah ke `mode: "onChange"` khusus agar error muncul langsung saat mengetik pada field remark. | Feedback lebih cepat, tetapi `onChange` menyentuh global form mode — perlu dipastikan tidak memicu autosave (AC-3). |
| (C) Validasi inline manual pada `onChange` handler `BBSTextArea` (set error sendiri tanpa `trigger()` autosave). | Kontrol penuh atas kapan error tampil & memisahkan dari save, tetapi menambah kode validasi manual di luar yup resolver. |

**Decision:** _TBD_

---

## EC-07: Manual Save (`handleSubmitRemark`) Saat Ada Baris > 535

**Scenario:** Teacher mengisi sebagian remark > 535 lalu menekan tombol **Save** manual (bukan autosave). `handleSubmitRemark` menjalankan `await trigger()` untuk seluruh form.

| Opsi | Behavior |
|------|----------|
| (A) `trigger()` gagal karena ada baris invalid → submit dibatalkan, error field tampil, tidak ada `createOrUpdateBulkRemark` terkirim. | Sesuai AC-7 (manual Save tetap memvalidasi seluruh form). Perilaku existing dipertahankan. |
| (B) Save tetap mengirim baris valid, baris invalid di-skip. | Berbeda dari perilaku existing (`trigger()` global) — menyimpang dari AC-7. Perlu keputusan eksplisit. |
| (C) Save memblokir seluruh form dan menampilkan toast ringkasan berapa baris invalid. | Lebih informatif; menambah UX pada `handleSubmitRemark`. |

**Decision:** _TBD_

---

## EC-08: Perilaku Saat Field Remark Di-lock (`isLocked && passDueDateInTerm`)

**Scenario:** Bila term sudah melewati due date (lihat `features/appraisal-lock/`), textarea `disabled` (`disabled={isLocked && passDueDateInTerm}`) dan tidak bisa diedit. Apakah validasi max-length masih relevan?

| Opsi | Behavior |
|------|----------|
| (A) Field disabled → tidak ada input baru, validasi max-length tidak berjalan, autosave tidak terpicu untuk field itu. | Field disabled tidak memicu `onChange`/autosave. Konsisten dengan perilaku existing. |
| (B) Validasi max-length tetap dievaluasi saat load untuk field berisi data lama > 535 (lihat EC-03), menandai error meski field disabled. | Menampilkan error pada field yang tidak bisa diedit — bisa membingungkan. Perlu keputusan. |

**Decision:** _TBD_
