# Kontrak API: Student Dropout Classifier (Tabular)

Dokumen ini untuk rekan Web yang akan memanggil API. Berisi semua yang dibutuhkan untuk integrasi.

## Ringkasan

| | |
|---|---|
| **Base URL** | `https://izmuhammadra8-dropout-classifier.hf.space` |
| **Endpoint** | `POST /predict` |
| **Health check** | `GET /` → `{"status":"ok","message":"API siap menerima prediksi"}` |
| **Dokumentasi interaktif** | `GET /docs` (Swagger UI) |

## Request

- Metode: `POST`
- Path: `/predict`
- Body: `application/json`
- Kirim **data mentah**. Imputasi, one-hot encoding, dan scaling sudah ditangani di dalam pipeline model, jadi rekan Web tidak perlu mengolah apa pun.

### Field yang dikirim

| Field | Tipe | Contoh | Keterangan |
|---|---|---|---|
| `Internet_Access` | string | `"Yes"` | akses internet (kategori) |
| `Attendance_Rate` | number | `85.5` | persentase kehadiran |
| `Assignment_Delay_Days` | number | `2` | rata-rata keterlambatan tugas (hari) |
| `Travel_Time_Minutes` | number | `30` | waktu tempuh ke kampus (menit) |
| `Part_Time_Job` | string | `"No"` | status kerja paruh waktu (kategori) |
| `Stress_Index` | number | `6` | indeks stres |
| `GPA` | number | `3.2` | IPK |

### Contoh request body

```json
{
  "Internet_Access": "Yes",
  "Attendance_Rate": 85.5,
  "Assignment_Delay_Days": 2,
  "Travel_Time_Minutes": 30,
  "Part_Time_Job": "No",
  "Stress_Index": 6,
  "GPA": 3.2
}
```

## Response sukses (200)

```json
{
  "prediction": "Graduate",
  "confidence": 0.91
}
```

- `prediction` berupa **nama kelas yang bermakna** (bukan angka), sudah diterjemahkan di sisi API.
- `confidence` bernilai 0–1 (kalikan 100 untuk persen).

## Response error

**422 Unprocessable Entity** — field kurang atau tipe data salah. Ditangani otomatis oleh validasi skema (Pydantic). Contoh, jika `GPA` dikirim sebagai teks:

```json
{
  "detail": [
    {
      "loc": ["body", "GPA"],
      "msg": "Input should be a valid number",
      "type": "float_parsing"
    }
  ]
}
```

Perbedaan penting dari kode status: **4xx berarti kiriman klien yang perlu diperbaiki**, bukan server yang rusak (yang akan berupa 5xx). Jadi jika Anda menerima 422, periksa kembali field dan tipe data yang dikirim.

## Catatan operasional

- **Cold start:** layanan di free tier "tidur" saat idle. Request pertama setelah lama menganggur bisa lambat (beberapa detik) karena model dimuat ulang. Tangani dengan loading state di UI, dan pancing dengan `GET /` beberapa menit sebelum demo.
- **CORS:** saat ini mengizinkan semua origin (`*`), jadi pemanggilan dari domain mana pun tidak diblok. (Akan dipersempit ke origin produksi menjelang rilis.)
- **Urutan field tidak wajib di JSON.** API menyusun ulang kolom sesuai urutan training di sisi server, jadi rekan Web cukup mengirim pasangan nama–nilai apa adanya.

---

## Contoh pemanggilan (JavaScript `fetch`)

```javascript
const API_URL = "https://izmuhammadra8-dropout-classifier.hf.space/predict";

async function predictDropout(data) {
  const res = await fetch(API_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  });

  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.detail || `Request gagal (${res.status})`);
  }

  return res.json();   // { prediction, confidence }
}
```

### Contoh penggunaan dengan form sederhana di HTML

```html
<button id="cekBtn">Cek Prediksi</button>
<div id="result"></div>

<script>
document.getElementById("cekBtn").addEventListener("click", async () => {
  const result = document.getElementById("result");
  result.textContent = "Memproses...";   // loading state

  // Pada praktiknya, nilai ini diambil dari input form pengguna
  const data = {
    Internet_Access: "Yes",
    Attendance_Rate: 85.5,
    Assignment_Delay_Days: 2,
    Travel_Time_Minutes: 30,
    Part_Time_Job: "No",
    Stress_Index: 6,
    GPA: 3.2,
  };

  try {
    const res = await fetch("https://izmuhammadra8-dropout-classifier.hf.space/predict", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });
    if (!res.ok) throw new Error(`Request gagal (${res.status})`);
    const out = await res.json();
    result.textContent = `${out.prediction} (${(out.confidence * 100).toFixed(1)}%)`;
  } catch (err) {
    result.textContent = "Gagal: " + err.message;
  }
});
</script>
```
