# Fullstack Product Engineer Case Study: Monozukuri Platform Revamp
**Target Repository:** `github.com/rakamindev/ai-interview-platform`  
**Engineer:** Naufal Pratama  
**Submission Structure:** Option B — Umbrella PR with Focused Sub-PRs  
**Umbrella PR Branch:** `umbrella/monozukuri-product-revamp`

---

## Executive Summary & Vision

Platform *AI Interview* dirancang untuk memecahkan hambatan terbesar dalam proses rekrutmen talenta di Indonesia: **skalabilitas evaluasi wawancara awal tanpa mengorbankan objektivitas dan empati terhadap kandidat**.

Namun, hasil audit menyeluruh terhadap kode sumber dan uji coba alur end-to-end mengungkapkan bahwa platform saat ini beroperasi sebagai **"Fragile Black Box"**. Sistem memiliki titik kegagalan tunggal (*single point of failure*), dependensi eksternal yang rapuh, dan ketiadaan *test harness*. Masalah paling krusial terletak pada antarmuka kandidat yang menyamarkan kegagalan teknis fatal dengan pesan sukses palsu (*false-positive UI*), serta kalkulasi evaluasi yang belum melindungi kandidat saat terjadi *unassessed skills*.

Melalui pendekatan **Monozukuri** (*Craftsmanship Beyond Baseline*), proyek ini merombak irisan vertikal fullstack: **"Resilient Evaluation & Candidate Experience Pipeline"** yang menjamin:
1. **Integritas Penilaian yang Adil:** Menangani kasus ekstrem *unassessed skills* dengan prinsip *Human-in-the-Loop* (menghapus penalti nilai 0 sepihak).
2. **Resiliensi Sistem & Transparansi UI:** Penanganan kegagalan terprediksi (*designed failure paths*) pada koneksi AI streaming dan penggantian false-positive UI dengan status pemulihan yang jujur.
3. **Kepatuhan Privasi Data (UU PDP):** Sanitasi logging PII dan pengamanan alur data kandidat.
4. **Piramida Pengujian Komprehensif:** Membangun test harness dari nol menggunakan **RSpec** (Unit/Service), **Bruno** (API Contract), dan **Vitest** (Frontend Components), dilengkapi bukti *Seeded Fault Test*.

---

## Step 2: Deep Context & Domain Immersion

### 1. Lanskap Industri Rekrutmen di Indonesia
Industri rekrutmen di Indonesia menghadapi volume pelamar yang sangat tinggi untuk setiap lowongan (terutama level intern dan junior). Recruiter dan assessor sering menghabiskan ratusan jam menyaring kandidat secara manual, atau menggunakan tes tertulis pilihan ganda yang tidak mencerminkan kemampuan komunikasi dan penyelesaian masalah sebenarnya.

### 2. Kebutuhan Pengguna (User Personas)
* **Assessor / Recruiter:** Membutuhkan ringkasan terstruktur dan bukti nyata (*evidence-based scoring*) untuk menilai apakah kandidat memenuhi standar kompetensi B7. Mereka tidak membutuhkan AI yang menggantikan keputusan akhir mereka, melainkan AI yang meringankan beban pencatatan dan pemetaan awal.
* **Hiring Manager:** Membutuhkan laporan *Fit-Gap* yang akurat terhadap kriteria lowongan (*Vacancy*), dengan indikator jelas mana skill yang benar-benar teruji dan mana yang memerlukan pendalaman di wawancara tahap berikutnya.

### 3. Dampak terhadap Kandidat & Regulasi UU PDP
Kandidat yang diwawancarai oleh AI tidak memilih platform ini; mereka wajib menggunakannya sebagai syarat seleksi. Sebuah kegagalan teknis (seperti koneksi terputus atau salah kalkulasi nilai) dapat menggagalkan perjalanan karier seseorang.
* **Kepatuhan UU PDP (UU No. 27 Tahun 2022):** Rekaman audio mentah dan transkrip percakapan adalah data pribadi yang sensitif. Sistem wajib:
  * Membatasi pencatatan log (tidak mencetak transkrip/nama kandidat di terminal log publik).
  * Memiliki batas retensi yang jelas dan perlindungan enkripsi token akses.

---

## Step 3: Defining Problem & Gap to Ideal Condition

### Core Problem Statement
> *"Platform AI Interview saat ini beroperasi sebagai kotak hitam yang rapuh (Fragile Black Box), di mana kegagalan infrastruktur eksternal dan model AI ditutupi oleh false-positive UI state yang menyesatkan kandidat, tanpa adanya failure recovery path maupun test harness yang melindungi integritas evaluasi dan kepatuhan data (UU PDP)."*

### Severity-Ranked Findings (P0 to P3)

| Severity | Klasifikasi | Temuan Cacat Lapangan | One-Line Impact Statement |
|---|---|---|---|
| **P0** | *Defective Implementation* | **Silent Failure & False Positive UI:** Saat WebSocket Gemini AI terputus/gagal, backend menutup sesi darurat dengan `reason: 'error'`, namun frontend menganggap semua status `ended` sebagai sukses dan menampilkan centang hijau `✅ Interview Complete: Thank you. The interview has been recorded.` | **Fatal.** Kandidat tertipu mengira wawancara terekam sempurna, sementara recruiter menerima laporan kosong tanpa notifikasi error teknis. |
| **P1** | *Defective Implementation* | **Brittle Hardware Check Dependency:** Fitur speed test menguji upload 500 KB ke server pihak ketiga gratis di AS (`httpbin.org`) dengan batas wajib minimal 4 Mbps. | Kandidat di Indonesia dengan koneksi internet stabil terhalang masuk wawancara akibat latensi tinggi lintas benua ke AS. |
| **P1** | *Missing Specification* | **Ketiadaan Test Harness & CI Baseline:** Repositori bawaan memiliki **0 specs RSpec**, **0 runner tes frontend**, dan tanpa automasi CI. | Zero regression safety; tidak ada jaminan otomatis bahwa perubahan kode tidak merusak algoritma scoring atau flow autentikasi. |
| **P1** | *Missing Specification* | **Unassessed Skills Hallucination / Bias:** Jika durasi wawancara habis sebelum semua skill sempat digali AI, generator scoring menebak skor atau memberi nilai penalti yang merusak rata-rata. | Penilaian tidak adil terhadap kandidat kompeten yang kehabisan waktu pada sesi wawancara berdurasi pendek. |
| **P1** | *Data Privacy (UU PDP)* | **Plaintext PII Logging:** Backend mencetak transkrip dialog dan nama kandidat ke standard output / log file tanpa proses sanitasi/masking. | Pelanggaran prinsip perlindungan data pribadi kandidat di bawah regulasi UU PDP Indonesia. |
| **P2** | *Defective Configuration* | **Broken Invite URL Seam:** Endpoint backend menghasilkan `invite_url` menggunakan port API internal (`:3001`) alih-alih port frontend (`:5173`). | Recruiter membagikan tautan 404 yang rusak ke kandidat jika disalin langsung dari dashboard. |
| **P2** | *Defective Implementation* | **Deprecated AI Model Dependency:** Konfigurasi sample mengunci model `gemini-2.0-flash-001` yang telah dihentikan (404) oleh Google tanpa mekanisme resolusi model dinamis. | Sistem gagal total saat di-deploy di environment baru tanpa adanya fallback model yang valid. |
| **P3** | *UI/UX Polish* | **Rigid Layout & AI Slop Microcopy:** Tampilan hardware check dan ruang wawancara kaku, tidak memiliki visual audio meter mikrofon interaktif, dan status koneksi tidak informatif. | Mengurangi rasa percaya diri kandidat dan menurunkan persepsi profesionalitas platform di mata pengguna. |

### Pemisahan Missing Specification vs. Defective Implementation
* **Missing Specification (Belum Pernah Dirancang):**
  1. Status evaluasi `inconclusive / unassessed` untuk skill yang belum sempat ditanyakan.
  2. Kebijakan sanitasi logging PII kandidat.
  3. Komponen test suite otomatis (RSpec, Vitest, Bruno).
* **Defective Implementation (Ada Tapi Cacat/Rapuh):**
  1. Handler WebSocket di frontend yang menyamakan `ended (error)` dengan `complete (success)`.
  2. Pemilihan target endpoint speed test ke `httpbin.org` alih-alih internal `/api/v1/speed_test`.
  3. String template `invite_url` di model `Session` yang mengarah ke port API backend.

### Constraint Signals untuk Technical Lead
1. **Single Point of Failure (SPOF) pada Gemini Live BidiGenerateContent:**
   * *Risiko:* Ketergantungan langsung pada protokol WebSocket audio real-time dua arah rentan terhadap pemutusan mendadak akibat kuota API habis atau jitter jaringan kandidat.
   * *Rekomendasi Eskalasi:* Perlu arsitektur *Hybrid Fallback Mode*: jika streaming WebSocket terputus 3x berturut-turut, sistem otomatis mengalihkan kandidat ke mode *turn-based* (Speech-to-Text -> LLM text -> TTS audio playback).
2. **Ambiguity Kebijakan Retensi Multi-Tenant UU PDP:**
   * *Risiko:* Belum ada definisi jelas mengenai hak penghapusan data kandidat (*Right to be Forgotten*) dan isolasi data transkrip jika suatu tenant berhenti berlangganan.

---

## Step 4: Revamp Strategy, Acceptance Criteria & Trade-offs

### Evaluasi Solusi: Option A vs Option B

| Dimensi Evaluasi | Option A: Quick Patch (Solusi Tambal Sulam) | Option B: Monozukuri Resilient Architecture (Opsi Terpilih) |
|---|---|---|
| **Pendekatan Teknis** | Menambahkan timer reconnect di React, menurunkan threshold upload speed di frontend, dan membungkus `rescue` sederhana di controller. | Mengimplementasikan Finite State Machine (FSM) pada siklus `Session`, standardisasi status `unassessed` pada scoring, sanitasi log PII, dan multi-state error recovery UI. |
| **Product Impact vs Cost** | Biaya sangat rendah (< 2 jam kerja), tetapi dampak produk minim. Masalah integritas data dan kecemasan kandidat tetap terjadi. | Biaya implementasi moderat, namun mengubah produk menjadi platform enterprise-ready yang adil dan dapat dipercaya kandidat serta recruiter. |
| **Long-term Maintainability** | Buruk. Kode menjadi spaghetti karena logic error tercecer di komponen UI tanpa kontrak data backend yang jelas. | Sangat baik. State lifecycle terpusat, pengujian otomatis berlapis (RSpec + Bruno + Vitest), dan mudah diperluas (*walked back cheaply*). |
| **Failure Modes** | Masih rentan menghasilkan silent failure jika jaringan drop di level socket TCP; recruiter tetap menerima data setengah jadi. | Seluruh kegagalan dipetakan secara deterministik: jika koneksi putus, kandidat diberi tahu transparan dengan opsi rekoneksi atau reschedule. |
| **Contextual Fit** | Tidak mencerminkan standar peran *Product Engineer*. | **Sangat Sesuai.** Mewujudkan standar Monozukuri dan menjawab seluruh rubrik evaluasi brief Rakamin. |

### Self-Derived Acceptance Criteria (Handling Edge Cases)
1. **Unassessed Skills:**
   * *Given* kandidat menyelesaikan wawancara sebelum semua skill dari assessment sempat digali oleh AI.
   * *When* proses kalkulasi portfolio dijalankan oleh worker.
   * *Then* skill yang belum ter-cover (`not_yet`) diberi label `inconclusive / unassessed` dengan rating `null`, tidak dihitung sebagai angka 0 dalam rata-rata fit-gap score, dan diberi tag visual *"Needs Human Assessor Review"*.
2. **AI Connection Interruption:**
   * *Given* koneksi WebSocket antara backend dan Google Gemini terputus di tengah sesi.
   * *When* retry otomatis gagal mencapai batas maksimum (3x).
   * *Then* sesi bertransisi ke status `interrupted`, bukan `ended`. Frontend menampilkan layar status gangguan teknis lengkap dengan tombol *Reconnect*, Session ID, dan opsi menghubungi assessor (bukan centang hijau sukses).
3. **Hardware Speed Test Seam:**
   * *Given* kandidat membuka halaman hardware check dari jaringan mana pun di Indonesia.
   * *When* pengujian kecepatan internet dimulai.
   * *Then* sistem mengarahkan request upload ke endpoint internal `/api/v1/speed_test` secara deterministik dengan batas latency realistis.
4. **Data Protection (UU PDP):**
   * *Given* request transkrip atau sesi diproses oleh Rails server.
   * *When* log ditulis ke STDOUT atau log file.
   * *Then* nama lengkap kandidat, email, dan isi transkrip audio disanitasi/dimask, mencegah paparan PII pada sistem pemantauan logs.

---

*(Dokumen ini akan terus diperbarui seiring implementasi Step 5 dan bukti pengujian Sub-PR).*
