# digID KYC — Prototipo funcional (Etapa 1)

Prototipo end-to-end: escaneo de INE (frontal/reverso) + prueba de vida (Amazon
Rekognition Face Liveness) + face match (CompareFaces), con veredicto server-side
y analisis JSON completo visible y descargable.

## Correr

Requiere Node >= 20 y `backend/.env` con credenciales AWS (ver `.env.example`).

```bash
cd backend && npm install && npm run dev     # http://localhost:3001
cd frontend && npm install && npm run dev    # http://localhost:5173
```

## Tests

```bash
cd backend && npm test    # API con AWS mockeado
cd frontend && npm test   # helpers de escaneo
```

## Estructura

- `backend/` — Express + AWS SDK v3. Guarda por sesion: `front.jpg`, `back.jpg`,
  `selfie-reference.jpg`, `analysis.json` en `backend/storage/<sessionId>/`.
- `frontend/` — Vite + React 18. Escaneo OpenCV intacto del prototipo original;
  selfie reemplazada por `FaceLivenessDetectorCore`.

Nota: `AWS_REGION` debe permanecer `us-east-1` — el detector de liveness del
frontend tiene la region fija y debe coincidir con el backend.

## Costos

Cada verificacion completa ≈ $0.016 USD (liveness $0.015 sin free tier).

## Fuera de alcance (por decision)

Sin S3, sin empaquetado SDK/embed, credenciales solo en `backend/.env` (dev).
