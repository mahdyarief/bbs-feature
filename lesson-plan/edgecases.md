# Edge Cases — Lesson Plan

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Copy ke kombinasi target yang sudah ada

**Scenario:** Guru menyalin lesson plan ke `ayCopy` + `classCopy` yang ternyata sudah punya lesson plan untuk kombinasi `(teacher, class, subject, ay, term, week)` yang sama.

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 409 | Copy gagal, tampilkan pesan "Lesson plan already exists in target class/ay" — data target aman, guru harus edit manual. |
| (B) Overwrite | Timpa lesson plan target dengan konten yang disalin — berisiko kehilangan data. |
| (C) Duplikasi tetap dibuat | Buat lesson plan kedua meski kombinasi sama — melanggar unique constraint, butuh relaksasi constraint. |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 409** karena paling aman dan konsisten dengan aturan unik (Business Rules #1). Guru bisa pilih kelas/AY lain atau edit target secara manual.

---

## EC-02: Copy ke kelas yang bukan milik guru tsb (dari Library)

**Scenario:** Di Lesson Plan Library, guru melihat lesson plan milik guru lain dan ingin menyalinnya ke kelas sendiri (Copy to Class).

| Opsi | Behavior |
|------|----------|
| (A) Copy dengan teacherId = user yang menyalin | Hasil copy tercatat milik guru yang menyalin (bukan guru asal) — sesuai ekspektasi "copy untuk saya". |
| (B) Copy dengan teacherId = guru asal | Hasil copy tetap milik guru asal — tidak berguna untuk guru yang menyalin. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Saat copy dari library, `teacherId` pada lesson plan baru = `req.user.id` (pengguna yang menyalin), sehingga muncul di list lesson plan miliknya. `sourceLessonPlanId` mencatat asal.

---

## EC-03: Week tidak valid untuk term yang dipilih

**Scenario:** AY 2026/2027 term 1 hanya punya 10 minggu, tapi guru mengirim `week=15` (atau memilih term 1 lalu week dari term 2).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 400 | Validasi `week` terhadap `academic_year_week` — pesan "Week is not valid for the selected term". |
| (B) Terima asal 1-40 | Tidak validasi per term — data bisa masuk tapi tidak sinkron dengan kalender akademik. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Modul `academic-year` sudah punya `AcademicYearWeek` entity, jadi validasi ini mudah dan mencegah data kacau.

---

## EC-04: HOD / Principal mencoba komentar dengan role yang salah

**Scenario:** User ber-role HOD mengirim `commentType=PRINCIPAL` (atau sebaliknya).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 403 | Role mismatch → "You don't have permission to comment". |
| (B) Auto-map ke role user | Server memaksa `commentType` sesuai role user yang login — user tidak bisa salah isi. |

**Decision:** _TBD_ — rekomendasi: **(A)** tolak dengan 403 + validasi di backend. Opsi (B) bisa jadi fallback UX di frontend (dropdown comment type hanya menampilkan role user), tapi backend tetap harus validasi.

---

## EC-05: Halaman utama menampilkan filter tanpa AY aktif

**Scenario:** User membuka halaman Lesson Plan tanpa memilih Academic Year (default).

| Opsi | Behavior |
|------|----------|
| (A) Default ke AY aktif | Otomatis pilih AY paling baru (contoh: 2026/2027, id 27) seperti teacher web. |
| (B) Tampilkan list kosong + minta pilih | User harus pilih AY dulu baru list muncul. |

**Decision:** _TBD_ — rekomendasi: **(A)** default ke AY aktif, sesuai perilaku teacher web (`#ls_ay` sudah ter-select ke AY 27 saat load).

---

## EC-06: Guru tidak punya kelas diampu di AY terpilih

**Scenario:** Guru pindah campus / AY baru belum ada assignment kelas, dropdown Class kosong.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan empty state | List kosong + pesan "No class assigned for this academic year"; tombol Create dinonaktifkan. |
| (B) Tampilkan semua kelas | Tampilkan semua kelas (tidak dibatasi assignment) — melanggar business rule kelas terbatas. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Teacher web juga hanya menampilkan kelas yang diampu.

---

## EC-07: Komentar dihapus / diedit

**Scenario:** HOD/Principal sudah menulis komentar, lalu ingin mengubah atau menghapusnya.

| Opsi | Behavior |
|------|----------|
| (A) Tidak bisa dihapus/diedit | Teacher web tidak punya fitur edit/hapus comment — komentar permanen. |
| (B) Bisa dihapus oleh penulis | Tambah endpoint DELETE comment — scope tambahan di luar replicate. |

**Decision:** _TBD_ — rekomendasi: **(A)** untuk fase pertama (replicate persis teacher web). Opsi (B) bisa jadi enhancement di notes.md.

---

## EC-08: Edit header saat lesson plan sudah dikomentari

**Scenario:** Guru mengubah Topic/Term/Week/Class dari lesson plan yang sudah punya komentar HOD/Principal.

| Opsi | Behavior |
|------|----------|
| (A) Boleh edit, komentar tetap | Komentar tetap terhubung ke lesson plan id yang sama — riwayat komentar tidak hilang. |
| (B) Blokir edit jika ada komentar | Guru harus minta HOD hapus komentar dulu — terlalu restriktif. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Komentar melekat pada `lesson_plan_id`, bukan pada nilai header; edit header tidak memutus relasi.

---

## EC-09: Upload file materi melebihi ukuran atau ekstensi tidak didukung

**Scenario:** Guru mengunggah file materi pada bagian Material / Resources (Power Point, PDF, Video) dengan ukuran melebihi batas atau ekstensi di luar format yang diizinkan (misal `.exe`, `.zip`).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 400 Validation Error | Filter pada Multer & Service menolak file sebelum diproses, kembalikan pesan: "File size exceeds limit" atau "Unsupported file format. Allowed formats: PDF (.pdf), PPT (.ppt, .pptx, .key), Video (.mp4, .3gp, .avi, .flv)". |
| (B) Kompresi otomatis di backend | Mencoba kompres file secara asinkron — membutuhkan resource komputasi tinggi dan kompleksitas transcode video. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Konsisten dengan batasan teknis upload file BBS dan keamanan server.

---

## EC-10: Pengisian Video Material (File Upload vs Link URL)

**Scenario:** Di bagian Material / Resources untuk Video, form menyediakan opsi upload file (`video_material[]`) ATAU tautan URL (`videolink_material`). Guru mengisi keduanya atau hanya salah satunya.

| Opsi | Behavior |
|------|----------|
| (A) Izinkan keduanya tersimpan | URL tersimpan di field teks/serialized array dan file tersimpan sebagai lampiran materi — guru fleksibel melampirkan video lokal sekaligus tautan YouTube/eksternal. |
| (B) Paksa salah satu (mutually exclusive) | Jika file diunggah, input link dikosongkan/dinonaktifkan dan sebaliknya. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Mengikuti perilaku form legacy di mana input file dan input teks URL tidak saling mengunci.

---

## EC-11: Multiple files pada kategori material yang sama

**Scenario:** Guru mengunggah lebih dari 1 file pada field material yang sama (misal memilih 2 file PDF lembar kerja via input `pdf_material[]`).

| Opsi | Behavior |
|------|----------|
| (A) Izinkan multiple files | Form legacy menggunakan `<input name="...[]" type="file" multiple>`, sehingga mendukung pengunggahan banyak file sekaligus untuk kategori materi yang sama. |
| (B) Batasi hanya 1 file per kategori | Hanya menerima single file per kategori material — membatasi guru yang memiliki lebih dari satu slide/handout. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Sesuai langsung dengan markup legacy `create2.html` yang menggunakan input file array `multiple`.

---

## EC-12: Akses Sub-Menu Lesson Plan Viewer oleh Guru Biasa

**Scenario:** Guru biasa (non-HOD dan non-Principal) mencoba mengakses sub-menu Lesson Plan Viewer melalui URL langsung `/lesson-plan-viewer`.

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 403 Forbidden / Redirect 404 | Frontend menggunakan guard hook `usePrincipalOrHod`. Jika false, redirect ke `/dashboard` atau `/lesson-plan`. Backend endpoint `/v1/lesson-plans/viewer` memvalidasi role reviewer HOD/Principal dan mengembalikan error 403 jika user tidak berhak. |
| (B) Izinkan tapi hanya lihat lesson plan sendiri | Menu tetap terbuka tapi hanya menampilkan data guru tersebut — membingungkan karena fungsi menu viewer adalah supervisi tim. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Sub-menu Lesson Plan Viewer ditujukan khusus untuk supervisi kepala departemen dan kepala sekolah, sehingga harus diproteksi ketat baik di frontend UI/navigasi maupun di backend guard.

---

## EC-13: Penomoran Counter Auto-Rename File saat Upload Bertahap

**Scenario:** Guru mengunggah 2 file PPT pertama (Counter 01, 02), kemudian di lain waktu mengunggah 1 file PPT tambahan pada lesson plan yang sama.

| Opsi | Behavior |
|------|----------|
| (A) Hitung counter berdasarkan file eksisting | Server menghitung `MAX(counter_number)` untuk `(lesson_plan_id, category)` terkait, sehingga file baru otomatis mendapatkan counter berikutnya (`03`), menghasilkan `PPT - [Topic] - Term [Term] - Week [Week] - 03.pptx`. |
| (B) Reset counter ke 01 dan timpa file lama | Menimpa file lama dengan counter sama — menyebabkan risiko hilangnya dokumen terdahulu. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Konsisten dengan format penamaan multi-file `PPT/PDF/VIDEO - [Topic] - Term [Term] - Week [Week] - [Counter]` tanpa risiko overwrite file yang sudah tersimpan.
