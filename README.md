# 🩺 Asclepius Web Server — Ivan Alif Hadrian

> **Submission Dicoding | Belajar Penerapan Machine Learning pada Google Cloud**  
> Backend REST API berbasis **Hapi.js** untuk deteksi kanker dari gambar menggunakan TensorFlow.js, dengan penyimpanan histori ke **Firestore** dan deployment otomatis ke **Google Cloud Run** via GitHub Actions.

---

## 📋 Deskripsi

**Asclepius** adalah web server backend yang menerima gambar kulit dari user, menjalankan inferensi model TensorFlow untuk mendeteksi apakah gambar menunjukkan indikasi kanker, lalu menyimpan hasilnya ke **Cloud Firestore**. Seluruh infrastruktur berjalan di **Google Cloud Platform (GCP)**.

### Komponen GCP yang Digunakan

| Layanan GCP | Fungsi |
|---|---|
| **Cloud Run** | Hosting backend API (serverless, auto-scale) |
| **Cloud Firestore** | Database NoSQL untuk menyimpan histori prediksi |
| **Cloud Storage** | Menyimpan file model TensorFlow (`model.json`) |
| **App Engine** | Hosting frontend aplikasi |

---

## 🔗 URL Deployment

| Layanan | URL |
|---|---|
| **Backend API** | `https://asclepius-backend-1006361217345.asia-southeast2.run.app/` |
| **Frontend** | `https://submissionmlgc-ivanalifhadrian.et.r.appspot.com/` |
| **GCP Project ID** | `submissionmlgc-ivanalifhadrian` |
| **GCS Bucket** | `asclepius-bucket-van` |
| **Region** | `asia-southeast2` (Jakarta) |

---

## 🗂️ Struktur Folder

```
asclepius-web-server/
│
├── .github/
│   └── workflows/
│       └── deployBackend.yml       # CI/CD: deploy otomatis ke Cloud Run
│
├── src/
│   ├── server/
│   │   ├── server.js               # Entry point: inisialisasi Hapi + middleware
│   │   ├── routes.js               # Definisi semua endpoint API
│   │   └── handler.js              # Logic handler setiap endpoint
│   │
│   ├── services/
│   │   ├── loadModel.js            # Load model TF.js dari Cloud Storage URL
│   │   ├── inferenceService.js     # Prediksi gambar dengan model TensorFlow
│   │   └── storeData.js            # Simpan hasil prediksi ke Firestore
│   │
│   └── exceptions/
│       ├── ClientError.js          # Base error class (HTTP 400)
│       └── InputError.js           # Error untuk input tidak valid (extends ClientError)
│
├── .env                            # Konfigurasi env lokal (MODEL_URL, PROJECT_ID)
├── .gitignore
├── Dockerfile                      # Docker image untuk Cloud Run
├── package.json                    # Dependencies Node.js
└── requirements.json               # Metadata URL deployment GCP
```

---

## 🛠️ Tech Stack

| Teknologi | Versi | Fungsi |
|---|---|---|
| **Node.js** | 18 | Runtime JavaScript server |
| **@hapi/hapi** | ^21.3.12 | Framework web server |
| **@tensorflow/tfjs-node** | ^4.22.0 | Inferensi model TensorFlow di Node.js |
| **@tensorflow/tfjs** | ^4.22.0 | TensorFlow.js core |
| **@google-cloud/firestore** | ^7.10.0 | Client SDK Cloud Firestore |
| **dotenv** | ^16.4.5 | Load environment variables dari `.env` |
| **nodemon** | ^3.1.7 | Auto-restart server saat development |

---

## 🌐 API Endpoints

### `GET /`
Health check — memastikan server berjalan.

**Response:**
```json
{
  "status": "success",
  "message": "Asclepius Backend is running successfully!"
}
```

---

### `POST /predict`
Menerima gambar dan menjalankan prediksi kanker.

**Request:**
- **Content-Type:** `multipart/form-data`
- **Body:** field `image` berisi file gambar (JPEG)
- **Max Size:** 1.000.000 bytes (1 MB)

**Response (201 Created):**
```json
{
  "status": "success",
  "message": "Model is predicted successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "result": "Cancer",
    "suggestion": "Segera periksa ke dokter!",
    "createdAt": "2024-01-15T10:30:00.000Z"
  }
}
```

**Response Error (400) — gambar tidak valid:**
```json
{
  "status": "fail",
  "message": "Terjadi kesalahan dalam melakukan prediksi"
}
```

**Response Error (413) — ukuran file melebihi 1 MB:**
```json
{
  "status": "fail",
  "message": "Payload content length greater than maximum allowed: 1000000"
}
```

**Logika Prediksi:**
- Score > 50% → `result: "Cancer"`, `suggestion: "Segera periksa ke dokter!"`
- Score ≤ 50% → `result: "Non-cancer"`, `suggestion: "Penyakit kanker tidak terdeteksi."`

---

### `GET /predict/histories`
Mengambil semua histori prediksi dari Firestore.

**Response (200 OK):**
```json
{
  "status": "success",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "history": {
        "result": "Cancer",
        "createdAt": "2024-01-15T10:30:00.000Z",
        "suggestion": "Segera periksa ke dokter!",
        "id": "550e8400-e29b-41d4-a716-446655440000"
      }
    }
  ]
}
```

---

## ⚙️ Arsitektur & Alur Kerja

### Inisialisasi Server (`server.js`)

```
server.js start
    │
    ├─→ loadModel() → tf.loadGraphModel(MODEL_URL dari Cloud Storage)
    │       └─→ model disimpan di server.app.model (shared antar request)
    │
    ├─→ server.route(routes) → daftarkan semua endpoint
    │
    └─→ server.ext('onPreResponse') → middleware error handler:
            ├─ InputError     → 400 + {status: 'fail', message}
            ├─ Boom 413       → 413 + pesan ukuran file
            └─ Boom lainnya   → statusCode + pesan generik
```

### Alur Prediksi (`POST /predict`)

```
Request multipart/form-data (image)
        │
        ▼
handler.js: postPredictHandler()
        │
        ├─→ inferenceService.predictClassification(model, image)
        │       │
        │       └─→ tf.node.decodeJpeg(image)
        │               .resizeNearestNeighbor([224, 224])
        │               .expandDims()
        │               .toFloat()
        │           → model.predict(tensor)
        │           → score = Math.max(...prediction) * 100
        │           → result = score > 50 ? 'Cancer' : 'Non-cancer'
        │           → tensor.dispose() + prediction.dispose()
        │
        ├─→ crypto.randomUUID() → generate ID unik
        │
        ├─→ storeData(id, data) → Firestore.collection('predictions').doc(id).set(data)
        │
        └─→ response 201 + {status, message, data}
```

### Hierarki Error

```
Error (built-in)
    └── ClientError (statusCode default: 400)
            └── InputError (untuk kesalahan input/prediksi)
```

---

## 🤖 Model AI

| Properti | Detail |
|---|---|
| **Tipe** | TensorFlow GraphModel (SavedModel format) |
| **Input** | Gambar JPEG → decode → resize 224×224 → float tensor |
| **Output** | Score probabilitas (0–1), threshold 50% |
| **Lokasi** | `gs://asclepius-bucket-van/model.json` |
| **Loader** | `tf.loadGraphModel(MODEL_URL)` via tfjs-node |

---

## 🐳 Docker

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["npm", "run", "start"]
```

- Base image: `node:18`
- Port: **3000**
- Install hanya production dependencies (`--production`)

---

## 🚀 CI/CD: GitHub Actions → Cloud Run

### Trigger
Pipeline berjalan otomatis saat ada **push ke branch `deployment`**.

### Tahapan Pipeline

| Step | Aksi |
|---|---|
| **Checkout** | Clone repo ke runner |
| **Auth GCP** | Autentikasi ke Google Cloud via `GCP_CREDENTIALS` secret |
| **Deploy** | Deploy source code ke Cloud Run menggunakan `deploy-cloudrun@v1` |
| **Output URL** | Print URL layanan yang ter-deploy |

### Environment Variables di Cloud Run

| Variabel | Nilai |
|---|---|
| `MODEL_URL` | `https://storage.googleapis.com/asclepius-bucket-van/model.json` |
| `GOOGLE_CLOUD_PROJECT` | `${{ secrets.GCP_PROJECT_ID }}` |

### GitHub Secrets yang Diperlukan

| Secret | Keterangan |
|---|---|
| `GCP_CREDENTIALS` | JSON key dari Service Account GCP |
| `GCP_PROJECT_ID` | ID project GCP (`submissionmlgc-ivanalifhadrian`) |

---

## 🚀 Cara Menjalankan Lokal

### Prasyarat
- Node.js 18+
- Akses ke Google Cloud (untuk Firestore & model di GCS)
- File `.env` terisi

### Setup `.env`

```env
MODEL_URL=https://storage.googleapis.com/asclepius-bucket-van/model.json
GOOGLE_CLOUD_PROJECT=submissionmlgc-IvanAlifHadrian
```

### Instalasi & Menjalankan

```bash
# Install dependencies
npm install

# Development (auto-restart)
npm run start:dev

# Production
npm start

# Server berjalan di:
# http://localhost:3000
```

### Test Endpoint dengan cURL

```bash
# Health check
curl http://localhost:3000/

# Prediksi gambar
curl -X POST http://localhost:3000/predict \
  -F "image=@/path/to/gambar.jpg"

# Lihat histori
curl http://localhost:3000/predict/histories
```

---

## 👤 Author

| Info | Detail |
|---|---|
| **Nama** | Ivan Alif Hadrian |
| **Program** | Dicoding — Belajar Penerapan Machine Learning pada Google Cloud |
| **Platform** | Google Cloud Run + Firestore + Cloud Storage |
| **Region** | asia-southeast2 (Jakarta) |
