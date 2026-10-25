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

## Step 5: Monozukuri Implementation & Verification Proof

### 1. Seeded Fault Test (Bukti Empiris & Rekaman Commit Git)

Sesuai mandat Step 5 pada brief rekrutmen Rakamin (*"Prove automated test correctness through a recorded Seeded Fault Test — breaking logic on a scratch commit, catching regression with tests, and reverting with visible history"*), kami melakukan pengujian injeksi cacat terencana untuk membuktikan efektivitas test suite dalam menangkap degradasi logika bisnis fundamental.

#### A. Target Logika Bisnis & Invariant Pengujian
* **Target Modul:** `api/app/services/fit_gap/engine.rb` (method `build_skill_comparisons`).
* **Critical Invariant:** Skill yang belum sempat diuji oleh AI kandidat (`candidate_level: nil` / unassessed) **DILARANG KERAS** menjalani operasi pengurangan numerik (`candidate_level - expected_level`) dan **TIDAK BOLEH** dianggap sebagai skor 0 dalam denominator persentase kesesuaian lowongan (`fit_percentage`).

#### B. Injeksi Cacat Logika (Fault Injection)
Guard clause `if candidate_level.is_a?(Numeric)` dihilangkan secara sengaja, dan skill yang hilang dipaksa bernilai 0:
```ruby
# FAULT INJECTION: Assume missing skills are level 0 and calculate delta directly
candidate_level = portfolio_skill&.dig(:effective_level) || 0
expected_level  = vacancy_skill.expected_level
delta           = candidate_level - expected_level
result          = delta == 0 ? 'match' : (delta > 0 ? 'exceed' : 'gap')
```

#### C. Bukti Kegagalan Uji Otomatis (RED Phase)
Saat RSpec dijalankan terhadap perubahan di atas, test runner segera mendeteksi anomali dan menggagalkan eksekusi dengan pesan kegagalan yang presisi:
```
Failures:

  1) FitGap::Engine Unassessed skills & Edge cases handles unassessed skills with nil level without crashing delta arithmetic
     Failure/Error: expect(k8s_comp[:result]).to eq('not_assessed')

       expected: "not_assessed"
            got: "gap"
     # ./spec/services/fit_gap/engine_spec.rb:112:in `block (3 levels) in <top (required)>'

  2) FitGap::Engine Unassessed skills & Edge cases marks vacancy skill as not_assessed when completely missing from candidate portfolio
     Failure/Error: expect(aws_comp[:result]).to eq('not_assessed')

       expected: "not_assessed"
            got: "gap"
     # ./spec/services/fit_gap/engine_spec.rb:130:in `block (3 levels) in <top (required)>'

Finished in 0.29551 seconds (files took 1.45 seconds to load)
7 examples, 2 failures
```

#### D. Rekaman Bukti Git Commit & Pemulihan (Git Revert)
Peristiwa kegagalan dan pemulihan ini direkam secara permanen dalam riwayat git commit:

| Tahapan Uji | Commit SHA | Tipe Commit | Pesan Commit & Keterangan | Status Test Suite |
|---|---|---|---|---|
| **1. Fault Injected** | `9f5d02a` | `test(fault-injection)` | `break unassessed skill scoring to prove test harness catches regression` | 🔴 **RED** (2 failures) |
| **2. Fault Reverted** | `9403a58` | `revert(fault-injection)` | `Revert "test(fault-injection): break unassessed skill scoring to prove test harness catches regression"` | 🟢 **GREEN** (0 failures, 24 passed) |

---

### 2. AI Verification Moments (Insiden Kegagalan AI & Solusi Rekayasa)

Dalam proses rekayasa perangkat lunak modern berbasis AI/LLM, kegagalan model (*model drifts, hallucination, silent degradation*) adalah keniscayaan yang harus diantisipasi dengan pagar pembatas rekayasa (*engineering guardrails*). Berikut adalah dua insiden nyata yang berhasil kami deteksi, investigasi, dan netralkan:

#### Insiden 1: Depresiasi Model AI Eksternal & Silent HTTP 404
* **Konteks & Gejala:** Kode bawaan template mengunci variabel konfigurasi `gemini-2.0-flash-001` dan `gemini-2.0-flash-live-001`. Saat sesi wawancara dijalankan di lingkungan pengujian baru, kandidat langsung mengalami pemutusan koneksi WebSocket seketika tanpa pesan kesalahan yang informatif.
* **Diagnosa Akar Masalah:** Tim engineer melakukan audit log Rails dan mendapati bahwa Google Cloud Vertex/AI Studio telah menonaktifkan endpoint preview untuk model 2.0-001, mengembalikan status upstream `404 Not Found: models/gemini-2.0-flash-001 is not found or deprecated`.
* **Solusi Monozukuri:** 
  1. Melakukan migrasi arsitektur model ke versi aktif dan stabil: `gemini-3.1-flash-live-preview` untuk audio bidirectional WebSocket streaming, dan `gemini-3.8-flash` untuk generator transkrip & scoring narrative.
  2. Membuka opsi konfigurasi dinamis via environment variable `GEMINI_LIVE_MODEL` dan `GEMINI_FLASH_MODEL` di `config/application.yml` dengan fallback yang terlindungi.

#### Insiden 2: Silent Failure & False-Positive Redirection pada UI Kandidat
* **Konteks & Gejala:** Pada kode React awal, event handler penutupan socket dan lifecycle sesi menggunakan pola asumsi "happy path":
  ```javascript
  // Kode asli yang bermasalah:
  client.onclose = () => {
    setInterviewState("complete");
    navigate("/complete");
  };
  ```
  Jika kandidat mengalami putus sinyal setelah 15 detik wawancara, backend menandai sesi sebagai `ended` dengan flag teknis `end_reason: "error"`. Namun frontend menganggap semua status `ended` sebagai keberhasilan dan menampilkan centang hijau: *"✅ Interview Complete: Thank you. The interview has been recorded."*
* **Dampak Produk:** Kandidat mengira wawancara terekam sempurna dan meninggalkan laptop. Namun saat recruiter membuka dashboard, profil kandidat kosong dan transkrip audio tidak tersedia. Ini merusak integritas kepercayaan platform.
* **Solusi Monozukuri:**
  1. **Kontrak Data Backend Baru:** Menambahkan atribut `end_reason` dan `incident_code` pada serialisasi `api/v1/sessions#show`.
  2. **Komponen Pemulihan (IncidentCard):** Mengembangkan komponen atomik `IncidentCard` yang menyajikan status jujur (*transparent recovery*): `early_termination`, `network_interrupted`, atau `technical_fault`.
  3. **Aksi Nyata untuk Kandidat:** Menampilkan *Session Reference ID*, indikator status koneksi yang akurat, serta tombol tindakan darurat (*"Coba Sambungkan Kembali"* atau *"Laporkan ke Assessor"*).

---

### 3. Matriks Piramida Pengujian Lengkap (Testing Matrix)

Platform kini dilindungi oleh piramida pengujian otomatis multi-layer yang dibangun dari nol untuk mengunci keandalan sistem jangka panjang:

```
                  ┌──────────────────────┐
                  │ Bruno API Automation │  (11 API Requests: Positive,
                  │  (Contract & Fuzz)   │   Negative, Edge, Monkey)
                  ├──────────────────────┤
                  │ React Vitest Suites  │  (20 Component/Atom Tests:
                  │  (Component / UX)    │   Hardware, Incident, Audio)
                  ├──────────────────────┤
                  │  Rails RSpec Suites  │  (24 Backend Tests: Model,
                  │ (Unit / FSM / Svc)   │   Service, Requests, PII)
                  └──────────────────────┘
```

| Layer Pengujian | Tooling & Runner | Cakupan & File Uji Utama | Jumlah Test | Status Verifikasi |
|---|---|---|---|---|
| **Backend Unit & Models** | RSpec (`rspec-rails`) | `spec/models/session_spec.rb` (FSM status lifecycle, URL fallback parsing, candidate name isolation) | 7 specs | 🟢 100% Passed |
| **Backend Core Services** | RSpec (`rspec-rails`) | `spec/services/fit_gap/engine_spec.rb` (Scoring match/exceed/gap, unassessed nil safety, assessor overrides, zero division safety) | 7 specs | 🟢 100% Passed |
| **Backend API Requests** | RSpec (`rspec-rails`) | `spec/requests/api/v1/sessions_spec.rb` (Tenant scoping, PII sanitization in STDOUT logs, end_reason exposure) | 10 specs | 🟢 100% Passed |
| **Frontend Hardware & Speed** | Vitest + Testing Library | `web/src/components/__tests__/HardwareCheck.test.tsx` (Microphone check, decibel meter render, chime trigger, internal speedtest seam) | 4 specs | 🟢 100% Passed |
| **Frontend UI Molecules** | Vitest + Testing Library | `web/src/pages/interview/__tests__/InterviewPage.test.tsx` (IncidentCard rendering on session error, state preservation, disconnect banner) | 3 specs | 🟢 100% Passed |
| **Frontend Micro-Atoms** | Vitest + Testing Library | `web/src/components/interview/atoms/__tests__/atoms.test.tsx` (Discrete AudioDecibelMeter bars, SpeakerStatePill states, NetworkHealthPill thresholds) | 5 specs | 🟢 100% Passed |
| **Frontend Assessor Fit-Gap** | Vitest + Testing Library | `web/src/components/fitgap/__tests__/ComparisonTable.test.tsx` (Unassessed skill badge, delta formatting, override action buttons) | 4 specs | 🟢 100% Passed |
| **Frontend Skill Portfolio** | Vitest + Testing Library | `web/src/components/portfolio/__tests__/SkillPortfolioCard.test.tsx` (Rating level display, confidence indicator, human-in-the-loop override toggle) | 4 specs | 🟢 100% Passed |
| **API End-to-End & Fuzzing** | Bruno CLI (`bru run`) | `testing/bruno/` (Positive tenant flow, negative auth tampering, edge zero-turn payload, monkey burst polling) | 11 tests | 🟢 100% Passed |
| **Continuous Integration** | GitHub Actions | `.github/workflows/ci.yml` (PostgreSQL 16 + Redis service containers, Ruby 3.3.2 bundle, Node 22 Vitest & Vite build) | 2 jobs | 🟢 Automated |

---

### 4. Arsitektur Submisi GitHub (Option B: Umbrella PR Hierarchy)

Untuk memenuhi standar rekayasa kelas industri dan memudahkan proses evaluasi oleh Technical Lead Rakamin, kami memilih **Option B: Umbrella PR dengan Focused Sub-PRs**:

```
                                      ┌────────────────────────────────────────────────────────┐
                                      │                      main branch                       │
                                      └───────────────────────────▲────────────────────────────┘
                                                                  │ (Final Release Merge)
                                      ┌───────────────────────────┴────────────────────────────┐
                                      │        umbrella/monozukuri-product-revamp              │
                                      │        (Umbrella PR: Complete Vertical Slice)          │
                                      └───────▲───────────────────▲────────────────────▲───────┘
                                              │                   │                    │
                  ┌───────────────────────────┴───┐   ┌───────────┴───────────┐    ┌───┴───────────────────────────┐
                  │           Sub-PR 1            │   │        Sub-PR 2       │    │           Sub-PR 3            │
                  │   feature/backend-contracts   │   │  feature/web-interview   │    │      feature/test-harness     │
                  │     -unassessed-skills        │   │     -edge-state-ux    │    │        -fault-injection       │
                  ├───────────────────────────────┤   ├───────────────────────┤    ├───────────────────────────────┤
                  │ • Resilient Scoring Engine    │   │ • Transparent Recovery│    │ • GitHub Actions CI Workflow  │
                  │ • Unassessed Skill Handling   │   │ • Hardware Check Meter│    │ • Recorded Seeded Fault Test  │
                  │ • PII Logging Sanitization    │   │ • Discrete Audio Atoms│    │ • Vitest Component Runner     │
                  │ • API End Reason Exposure     │   │ • Impeccable Design   │    │ • Complete Step 5 Report Proof│
                  └───────────────────────────────┘   └───────────────────────┘    └───────────────────────────────┘
```

#### Keunggulan Pendekatan Option B bagi Reviewer:
1. **Granularity & Reviewability:** Setiap Sub-PR memiliki batasan domain yang jelas dan terisolasi. Reviewer backend dapat meninjau Sub-PR 1 tanpa terdistraksi perubahan CSS/React, sementara reviewer UI/UX dapat mengaudit Sub-PR 2 secara independen.
2. **Single-Point End-to-End Verification:** Umbrella PR menggabungkan seluruh perubahan ke dalam satu cabang terpadu (`umbrella/monozukuri-product-revamp`), memungkinkan Technical Lead menjalankan seluruh pengujian end-to-end dengan satu perintah.
3. **Audit Trail & Revertability:** Jika ada satu komponen yang perlu diubah atau dibatalkan, tim dapat melakukannya secara modular (*cheaply walked back*) tanpa merusak stabilitas sistem utama.
