# Kontrak API: Meme Template Classifier

Dokumen ini untuk rekan Web yang akan memanggil API. Berisi semua yang dibutuhkan untuk integrasi.

## Ringkasan

| | |
|---|---|
| **Base URL** | `https://izmuhammadra8-meme-classifier.hf.space` |
| **Endpoint** | `POST /predict` |
| **Health check** | `GET /` → `{"status":"ok","num_classes":115,"image_size":[224,224]}` |
| **Dokumentasi interaktif** | `GET /docs` (Swagger UI) |

## Request

- Metode: `POST`
- Path: `/predict`
- Body: `multipart/form-data`
- Field: **`file`** — berisi berkas gambar (JPG/PNG)

## Response sukses (200)

```json
{
  "predictions": [
    { "label": "futurama_fry", "confidence": 0.8438 },
    { "label": "socially_awesome_awkward_penguin", "confidence": 0.0319 },
    { "label": "pepperidge_farm_remembers", "confidence": 0.0207 }
  ]
}
```

- `predictions` selalu berisi **3 kandidat teratas**, urut dari confidence tertinggi.
- `confidence` bernilai 0–1 (kalikan 100 untuk persen).

## Response error (400)

Jika berkas yang dikirim bukan gambar:

```json
{ "detail": "File harus berupa gambar." }
```

## Catatan operasional

- **Cold start:** layanan di free tier "tidur" saat idle. Request pertama setelah lama menganggur bisa lambat (beberapa detik) karena model dimuat ulang. Tangani dengan loading state di UI.
- **CORS:** saat ini mengizinkan semua origin (`*`), jadi pemanggilan dari domain mana pun tidak diblok. (Akan dipersempit ke origin produksi menjelang rilis.)

---

## Contoh pemanggilan (JavaScript `fetch`)

```javascript
const API_URL = "https://izmuhammadra8-meme-classifier.hf.space/predict";

async function classifyMeme(file) {
  const formData = new FormData();
  formData.append("file", file);   // nama field WAJIB "file"

  const res = await fetch(API_URL, { method: "POST", body: formData });

  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(err.detail || `Request gagal (${res.status})`);
  }

  const data = await res.json();
  return data.predictions;   // array top-3: [{label, confidence}, ...]
}
```

### Contoh penggunaan dengan input file di HTML

```html
<input type="file" id="memeInput" accept="image/*" />
<div id="result"></div>

<script>
document.getElementById("memeInput").addEventListener("change", async (e) => {
  const file = e.target.files[0];
  if (!file) return;

  const result = document.getElementById("result");
  result.textContent = "Memproses...";   // loading state

  try {
    const preds = await classifyMeme(file);
    result.innerHTML = preds
      .map(p => `${p.label}: ${(p.confidence * 100).toFixed(1)}%`)
      .join("<br>");
  } catch (err) {
    result.textContent = "Gagal: " + err.message;
  }
});
</script>
```
