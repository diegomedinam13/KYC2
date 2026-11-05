# HANDOFF — Prototipo digID KYC (corte 2026-07-14)

Estado al cierre de la sesion del 2026-07-14. Contexto vivo: boveda Obsidian
(`KYC-Obsidian/digid-kyc-sdk/08 Sesiones/` — `Sesion 2026-07-13 - Prototipo
funcional Rekognition.md` + addendums 1-6, `Sesion 2026-07-14 - HTTPS en LAN
para prueba movil.md`, `Sesion 2026-07-14 - Fix recorte superior INE (refinado
de bordes).md` + addendum 1) y ledger `.superpowers/sdd/progress.md`.

## Reporte de cierre + API de administracion (2026-09-24, rama `admin-api`) — LEER PRIMERO

- Sub-proyecto 2 de 3 del panel. Spec `docs/superpowers/specs/2026-09-24-reporte-y-api-admin-diseno.md`, plan
  `docs/superpowers/plans/2026-09-24-reporte-y-api-admin.md`, contrato para el panel `docs/admin-api.md`.
- `/api/admin/*` existe SOLO con `ADMIN_TOKEN` en `backend/.env` (Bearer, un token, sin roles). Sesiones
  (indice con `stage`, detalle, reporte calculado al pedirlo, evidencia por allowlist), workflows (GET/PUT/DELETE/
  reload, validados antes de escribir), `catalog` y `schema` para el editor, `health` con booleanos de proveedores.
- **VPS (HECHO 2026-09-26)**: `ADMIN_TOKEN` (64 hex) ya esta en `~/kyc/backend/.env` de la VM y el backend se
  reinicio (`[admin] API de administracion montada … cors: ninguno`); el Caddyfile ya lleva la exencion por matcher
  (respaldo `/etc/caddy/Caddyfile.bak-2026-09-26`) y el panel se sirve en `/panel/`. Sin `ADMIN_CORS_ORIGIN`
  (same-origin). Nota historica de como era antes: Caddy pone
  basic auth a todo `/api/*` y eso NO se combina con el Bearer: una peticion lleva UN solo header `Authorization`
  y el preflight CORS no lleva ninguno. TODO VPS (sub-proyecto 3): eximir `/api/admin/*` del `basic_auth` en
  Caddy con un MATCHER (`@noadmin not path /api/admin/*` + `basic_auth @noadmin { ... }`) — NO por la posicion
  textual de las directivas: Caddy ordena `basic_auth` antes que `handle`/`handle_path` segun su orden canonico
  fijo, sin importar en que linea del archivo este escrito cada uno, asi que un `handle /api/admin/*` puesto
  antes del `basic_auth` en el texto NO lo exime. En `/api/admin/*` el Bearer queda como unica autenticacion.
  Bloque completo en `KYC-PANEL/docs/caddy.md`.
- Pendiente: sub-proyecto 3 (panel, repo aparte). Tests: backend 628, frontend 523.
- Sub-proyecto 3 (panel) completado (2026-09-25): repo aparte `KYC-PANEL` (Vite+React bajo `/panel/`, 55 tests, deploy propio via GitHub Actions a la misma VM de Azure). Bloque de Caddy para eximir `/api/admin/*` del `basic_auth` documentado en `docs/caddy.md` de ese repo; estado y orden de despliegue en `KYC-PANEL/docs/HANDOFF.md`. El panel sirve del mismo host bajo `/panel/`, asi que en produccion NO hace falta `ADMIN_CORS_ORIGIN` (la nota de la linea 17 arriba sigue siendo correcta para pruebas cross-origin, pero el deploy real es same-origin); el log esperado tras el restart es `cors: ninguno`.

## Workflows: politica por sesion, sin reiniciar (2026-09-24, rama `workflows`) — LEER PRIMERO

- Sub-proyecto 1 de 3 del panel de control (2: reporte de cierre + API admin con `ADMIN_TOKEN`; 3: panel en
  repo aparte). Spec `docs/superpowers/specs/2026-09-24-workflows-politica-por-sesion-diseno.md`, plan
  `docs/superpowers/plans/2026-09-24-workflows-politica-por-sesion.md`.
- Un `.jsonc` por workflow en `WORKFLOWS_DIR` (default `backend/data/workflows/`, gitignored). `default.jsonc`
  se siembra desde `config/kyc.config.jsonc` al primer arranque. Cambios sin reiniciar (recarga por mtime);
  archivo roto en caliente (o `stat` que falla con EIO/EACCES) = ultima version buena + error en consola.
- Cada sesion guarda `policy.json` (politica resuelta + digest) y decide SOLO con eso. Link publico
  `?w=<id>`; `GET /api/config?workflow=`, `POST /api/sessions {workflow}`; expediente `policy.workflow`.
  `policy.json` corrupto = 500 `session_policy_invalid` (nunca cae a `default`).
- **Deploy (2026-09-24, post-merge)**: `deploy.yml` excluye `backend/data` del `rsync --delete` (sin eso cada deploy borraba los workflows de la VPS y solo quedaba `default` re-sembrado). Verificado en la VM tras el CD `36035578027`: `~/kyc/backend/data/workflows/default.jsonc` sembrado, `[policy] workflow default (…7b262590628158b3)`, `/api/config?workflow=default` OK, `.env` sin `KYC_CONFIG_PATH`.
- **Al desplegar en la VPS**: tras `git pull` + restart se crea `~/kyc/backend/data/workflows/default.jsonc`
  desde el `config/kyc.config.jsonc` del repo (MX + US). Si la VPS tenia `KYC_CONFIG_PATH`, ese archivo
  solo siembra; editar despues `data/workflows/default.jsonc`. Verificar `curl …/api/config` y que
  `backend/data/` no aparece en `git status`.
- Pendiente: probar en vivo por LAN con `?w=` desde el telefono; sesiones viejas en disco (sin `policy.json`)
  siguen decidiendo con `default` vigente y anotan `sessionDigest`.
- Tests: backend 567, frontend 523.

## KYC configurable (2026-09-23, rama `custom-kyc`) — LEER PRIMERO

- Politica del KYC en `WORKFLOWS_DIR` (`backend/data/workflows/<id>.jsonc`; `default.jsonc` se siembra
  desde `backend/config/kyc.config.jsonc`) — ver el bloque Workflows de arriba (guia: `docs/kyc-config.md`;
  diseno: `docs/superpowers/specs/2026-09-23-kyc-configurable-diseno.md`). Paises/documentos, checks,
  `impact`/`onUnavailable` (reject/review/report), umbrales e intentos. Solo `default.jsonc` invalido AL
  ARRANCAR detiene el proceso; cualquier otro archivo invalido conserva su ultima version buena.
  `npm run config:check` los revisa sin servidor.
- Ausente = no existe (no corre, no cobra, no sale). Dependencias se encienden como soporte (solo reportan).
- Paises: el archivo entregado deja MX (ine/passport/license) y US (passport/license) activos; los otros
  197 COMENTADOS. `main` ofrece los 198 — si esta rama se despliega tal cual, la VPS pierde esos paises
  hasta descomentarlos.
- Licencia de EE. UU. nueva: dos lados, sin OCR, solo liveness + CompareFaces. **Pendiente: prueba en vivo
  con una licencia real** (el perfil `credencial` solo se calibro con licencias mexicanas).
- `GOV_CHECKS_ENABLED`, `IP_ANALYSIS_ENABLED`, `OCR_MAX_ATTEMPTS`, `LIVENESS_MIN_CONFIDENCE`,
  `FACE_MATCH_MIN_SIMILARITY` ya no se leen (aviso al arrancar). La VPS tiene `GOV_CHECKS_ENABLED` y
  `OCR_MAX_ATTEMPTS` en su `.env`: quitarlos al desplegar. Copia local del env de la VPS:
  `backend/.env.vps` (gitignored).
- Deuda de diseno: en las consultas puras (NSS, certificados, cedulas, acta) un `no_record` cuenta como
  `failed` y, con el default `report`, llena `verdict.warnings` en sesiones aprobadas (p. ej. "sin cedulas"
  no es un fallo de la persona). La UI ya no pinta `warnings` (quedan en el JSON). Evaluar si esas
  consultas deben tratar `no_record` como `passed` con dato vacio.
- Sin `PROXYCHECK_API_KEY`, `ipRisk` sigue consultando la lista de Tor (gratis); antes solo pasaba con
  `IP_ANALYSIS_ENABLED=true`.
- Tests: backend 567, frontend 523 (rama `workflows`, 2026-09-24).

## Que esta hecho y verificado

**Backend completo** (18 tests verdes; smoke real contra AWS ok, cuenta 339713030892, us-east-1):
- Sesiones KYC en disco (`backend/storage/<uuid>/`), guardado front/back.
- `CreateFaceLivenessSession`, credenciales temporales via `GetSessionToken` (fallback dev — `GetFederationToken` denegado al user jorgeP).
- `POST /verify`: `GetFaceLivenessSessionResults` + `CompareFaces` (SimilarityThreshold 0, umbral nuestro) + `DetectFaces`; veredicto server-side (>= 85 / >= 80); guarda `selfie-reference.jpg` + `analysis.json` (sin campos `Bytes`).
- Endurecimiento: bind 127.0.0.1, CORS restringido a `localhost:5173` (dos hallazgos de security review, mitigados para dev).

**Frontend completo** (30 tests verdes, build limpio):
- Port 1:1 verificado byte-a-byte del prototipo `REACT DIEGUITO/kyc-react.html`.
- Wiring a backend (sesion, uploads, busy/error). `FaceLivenessDetectorCore` en espanol con `credentialProvider`. `ResultsView` con veredicto real + JSON de Amazon visible y descargable.
- **Escaneo v2 (reescrito post-plan por feedback de Mario)**: OpenCV 4.7.0 en Web Worker; deteccion libre en todo el encuadre con overlay de bordes en vivo; multi-candidato (nunca "el contorno mas grande") + validacion de geometria ID-1; doble variante (Canny estandar + equalizeHist/Canny 25-100 para frente lavado); fallback minAreaRect (dedos); preferencia por contenedor + rechazo de puro-patron (evita recorte solo-QR del reverso); compuerta de calidad sin OCR (Laplaciano + sharpRatio anti-QR-borroso, exposicion, glare, contraste) con hints especificos.
- **Escaneo v2.1 (2026-07-14, validado con 6 fotos reales de INE en mano)**: (a) refinado a bordes exteriores en el worker (`refineToOuterEdges`, cache-bust `?v=7`) — corrige que la captura recortara el encabezado "INSTITUTO NACIONAL ELECTORAL" y la greca (el detector se quedaba con el rectangulo interior bajo la franja gris, aspecto 1.84-1.97, y `validateQuad` lo aceptaba); (b) captura manual y archivos de galeria detectan con escalera de escalas (`detectDocumentStill`: 720→480→960 px, params por escalon) y recortan desde resolucion nativa — antes galeria detectaba a resolucion nativa (3/6 y con recorte interior), ahora 5/6 con aspecto 1.58-1.61.
- **Rediseno UI de escaneo (2026-07-14, solo Camera.jsx + styles.css, pipeline intacto)**: guia de encuadre persistente hasta que hay deteccion (antes desaparecia al primer segundo; regresa con retardo anti-parpadeo si se pierde el doc), copy nuevo (eyebrow/titulo/pista por lado), boton `?` con bottom-sheet de 4 consejos de captura, restyling navy+aqua (badge vidrio, shutter nucleo aqua, esqueleto sutil). Verificado en Chrome + 30 tests + build. Detalle: `Sesion 2026-07-14 - Rediseno UI de escaneo (guia, copy y micro-guia).md`.
- **Hosting movil en LAN (2026-07-14)**: `frontend/certs/` (mkcert, gitignored) activa HTTPS + exposicion a la red en Vite (requisito de getUserMedia en navegadores moviles); sin certs todo queda como antes. CA raiz para el telefono en `~/Desktop/mkcert-rootCA.pem`. Backend sigue bind 127.0.0.1 (el proxy de Vite lo cubre).
- **Escaneo v2.2 (2026-07-14 tarde, calibrado con 17 fotos nuevas de Mario en `~/Downloads/fotos pruebas/`: 12 webcam Photo Booth 1440x960 dia/noche-con-foco + 5 HEIC iPhone sobre escritorio)**: (a) deteccion EN VIVO con escalera 480→720→960 (rota al no detectar, se queda en la que pega; 4/12 tomas de webcam solo detectan a 720/960); (b) tarjeta VERTICAL en galeria/manual: `validateQuad` acepta aspecto ~1/1.586 con `allowPortrait`, `detectDocumentStill` reetiqueta esquinas para que el warp la acueste y `orientDocumentForStep` decide el 180 comparando bandas top/bottom (frente: franja gris arriba; reverso: MRZ abajo) — los 5 HEIC salen derechos; (c) metricas nuevas de calidad: `washedRatio` (bloques 4% del ancho con media>225 y std<14 = reflejo difuso del foco que lava datos; umbral 5%) y `lowLightRatio` (>28% de luma<100 = el quad se llevo escritorio en vez de tarjeta); regla de borrosidad combinada (sharp<25 || ratio<1.9 || (sharp<45 && ratio<2.6)) para no rechazar webcam suave-pero-legible; (d) `detect` regresa stats del FRAME (brightness/contrast/glareRatio) siempre — sin deteccion y glare>=0.2% la UI dice "una luz deslumbra la camara" (foco dentro del encuadre: caso real night-3, indetectable a toda escala); (e) guia direccional `frameGuidance` (acerca/aleja/centra/endereza) + badge con retencion y prioridad (`showStatus`) — los mensajes ya no parpadean a 200ms; (f) extract con ancho adaptativo 600-1200px (no inflar tarjetas chicas: diluia el Laplaciano); (g) escalera still reordenada [720,960,480] (a 480 el quad de una tarjeta chica recortaba el encabezado). Worker cache-bust `?v=9` (v=8 fue esta sesion; v=9 la pasada de limpieza posterior, sin cambio de comportamiento — verificado con corrida identica de las 17 fotos). Ademas la vista previa quedo EN ESPEJO (pedido de Mario): la constante `MIRROR_PREVIEW` (Camera.jsx) es la UNICA fuente de verdad — pone la clase `.is-mirrored` en video y overlay e invierte izquierda/derecha en `frameGuidance`; solo visual, la captura usa el frame crudo. Dictamen de las 17: 12 aceptadas; night-1/night-5/0285/0289 rechazadas por reflejo-lava (correcto: el foco tapa DOMICILIO/retrato), 0288 rechazada por quad-falso-fondo (antes ese recorte de escritorio pasaba como OK), night-3 sin deteccion pero con mensaje de foco. Detalle en la nota de sesion.
- **Liveness sin luces + reskin (2026-07-14 noche)**: backend manda `Settings: { AuditImagesLimit: 4, ChallengePreferences: [{ Type: 'FaceMovementChallenge' }] }` en `CreateFaceLivenessSession` (ya no hay secuencia de luces de colores; el contrato del request tiene test en `aws-endpoints.test.js`). Frontend: `disableStartScreen` en `FaceLivenessDetectorCore` — el check arranca solo al llegar al paso, sin boton "Iniciar verificacion"; la vista selfie es full-bleed navy como el escaner (`fullCamera` en App.jsx incluye `'selfie'`); fases propias animadas (preparando/analizando/error) con ovalo biometrico SVG; reskin del detector via tokens `--amplify-*` scoped a `.liveness-view` + clases `.amplify-liveness-*` (anillo del ovalo aqua via `--amplify-colors-border-secondary` que lee el canvas, hints como pildoras de vidrio navy, cancel/REC de vidrio, loaders aqua); `displayText` completo en espanol. LIMITE de Amplify: el exterior del ovalo durante el check es BLANCO hardcodeado (`#fff` en canvas, no themeable) — por eso el modulo va enmarcado como tarjeta redondeada sobre el navy. Fases custom verificadas por screenshot headless; en movil el check corre (capturas reales del 2026-07-14).
- **Fix congelamiento liveness + preflight de camara (2026-07-14 noche, 2a pasada)**: con `disableStartScreen` el detector llama `getUserMedia` al montar y su maquina NO tiene timeout en `cameraCheck` — si el permiso queda pendiente (escritorio: nunca entrega camara, sintoma reportado por Mario) la UI congela en loader mudo. Fix: `src/lib/cameraPreflight.js` (con tests) pide la camara ANTES de montar el detector — hint a los 6s ("busca el aviso del navegador"), error con motivo real (NotAllowedError/NotReadableError/timeout...) y Reintentar a los 30s; el detector monta con permiso caliente. ADEMAS: en movil Amplify va a full-screen con su propia UI (toast, REC, su X, match bar) y nuestro chrome encimado chocaba (doble X, titulo pisado, texto ilegible sobre el scrim blanco — capturas IMG_0299/0300): ahora el check activo renderiza SOLO el detector (`.liveness-view.is-check`); el chrome de marca vive unicamente en las fases propias (preparando/analizando/error). **Arnes dev**: `frontend/liveness-harness.html` + `src/liveness-harness.jsx` montan LivenessStep aislado (crea sesion sola; `?fakecam=1` = camara canvas falsa; instrumenta getUserMedia y loguea en pantalla + metricas de layout) — util para iterar la UI de liveness sin escanear INE cada vez; solo dev, no entra al build. **3a pasada (misma noche)**: el full-screen movil de Amplify (position:fixed via clases `--mobile`, activado por user-agent) se desalineaba en ventanas no estandar (captura de Mario en Stage Manager) — neutralizado por CSS scoped: el modulo queda como tarjeta enmarcada SIEMPRE (decision de Mario, manteniendo auto-inicio); errores runtime del detector mapeados a espanol (`livenessErrorMessage`, el onError entrega ingles crudo).
- Nota: commit `e2ef711` ("acabado de documento escaneado profesional") se hizo en otra sesion — revisar su contenido si algo del acabado visual sorprende.
- **Escaneo v2.3 (2026-07-16, spec `docs/superpowers/specs/2026-07-16-escaneo-robusto-diseno.md`, plan `docs/superpowers/plans/2026-07-16-escaneo-robusto.md`)**: (a) filtro de soporte de bordes (`quadEdgeSupport`, `SUPPORT_SAMPLES=48`/`SUPPORT_MIN_COVERAGE=0.45` en scan-worker.js): el quad ganador por contornos debe descansar sobre bordes fisicos en 3 de 4 lados — mata el falso positivo de blobs convexos (el marco ya no se pinta sobre una cabeza); (b) confirmacion temporal del overlay (`createDetectionConfirmer`, docscan.js, `CONFIRM_FRAMES=2`/`CONFIRM_MOVEMENT=0.05`): 2 frames coincidentes antes de pintar, y el overlay se apaga con `setQuad(null)` al perder la confirmacion (fix post-review: antes quedaba un marco congelado); (c) variante V3 por lineas Hough (`collectHoughCandidates`, `HOUGH_THRESHOLD=40`/`HOUGH_MIN_LINE_FRAC=0.22`/`HOUGH_MAX_GAP_FRAC=0.08`/`HOUGH_MAX_PER_FAMILY=6`/`HOUGH_ANGLE_TOL=20`/`HOUGH_DEDUP_DIST_FRAC=0.03`/`HOUGH_PENALTY=0.9`): arma el quad cuando los dedos rompen el contorno (tarjeta en mano al aire) — solo corre si los contornos no dejan ganador, mismo embudo de filtros; endurecido post-review con `houghQuadSupport`/`HOUGH_SUPPORT_MIN_WORST=0.70` (exige soporte de borde en los 4 lados, no solo 3 de 4 como los candidatos de contorno) porque en IMG_0285 un quad Hough rotado 45 grados cortaba la credencial y pasaba el embudo viejo, robandole a la escalera el peldano 960 donde los contornos si encontraban la tarjeta completa; con el fix esa foto vuelve a detectarse por contornos, vertical, ACEPTA (identica al baseline); (d) recorte manual con cropperjs v1 (`ManualCrop.jsx` + `manualCrop.js`, `manualCropDecision`, `MANUAL_CROP_AFTER_REJECTS=3`): aparece si la deteccion falla en captura manual/galeria o tras 3 rechazos seguidos de calidad (boton "Ajustar manualmente"); aspecto fijo ID-1, rotar 90, SIN correccion de perspectiva (decision de Mario, plan B documentado en el spec); la compuerta de calidad advierte una vez y "Usar de todos modos" en la segunda; endurecido post-review con `manualCropSeqRef` (cancela de verdad una confirmacion en vuelo) y un guard de timeout en `processFile`. Worker cache-bust `?v=13`. Arnes 21/21 (casos nuevos: 18 cabeza-blob, 19 tarjeta en mano al aire via Hough, 20 estanteria); vitest frontend 88 (antes 78); backend 23. devDependency `jpeg-js` nueva + runner `scripts/corpus-run.cjs` (compara worker baseline vs HEAD sobre `~/Downloads/fotos pruebas/`, los 5 HEIC se convierten con sips a una carpeta temporal): sobre 17 fotos, 0 regresiones de deteccion vs baseline, 13 ACEPTA en ambos, +1 veredicto mejorado (9.04 #3, wash 12.7%→9.5%), IMG_0285 identica al baseline tras el fix Hough. PENDIENTE: validacion humana en vivo de Mario (cabeza ya no se pinta; tarjeta en mano en movil y escritorio; flujo del recorte manual) y 3-5 fotos reales de tarjeta en mano al aire para validar `HOUGH_SUPPORT_MIN_WORST` en condiciones reales (hoy solo probado con el fixture sintetico del arnes, peor lado 0.792 vs umbral 0.70; margen medido: quad malo real 0.625 / quad bueno sintetico 0.792).
- **Escaneo v3 — marco fijo guiado (2026-07-17, spec `docs/superpowers/specs/2026-07-17-escaneo-marco-guiado-diseno.md`)**: v2.3 seguia fallando en condiciones reales pese a pasar el corpus de 17 fotos (Mario lo probo en local/movil/Azure). Diagnostico offline con material nuevo (8 fotos HEIC IMG_0350-0357 + 23 videos): IMG_0352 daba recorte MUTILADO (solo zona de QRs) y la calidad lo ACEPTABA; IMG_0354/0355 sin deteccion (tarjeta en mano, fondo de oficina venciendo a la deteccion libre); IMG_0357 recorte correcto pero RECHAZADO por reflejo (wash 11.9% vs umbral 5%) siendo legible. Hallazgo estructural: el camino EN VIVO nunca habia tenido prueba offline (el arnes solo cubria fotos fijas), asi que cada calibracion pasaba el corpus y fallaba en vivo.
  - **Que cambio**: (a) marco fijo guiado ID-1 (~80% del ancho, scrim) dibujado en el overlay SVG en vivo — el hilo de UI recorta el frame al ROI del marco + 15% de margen (`ROI_MARGIN_FRAC`) y solo eso va al worker, a ancho fijo 720 (la escalera 480/720/960 en vivo ya no aplica: con marco la tarjeta alineada tiene tamano conocido); (b) validacion nueva `validateQuadInMarco` (`frontend/src/lib/marco.js`, pura, sin DOM): el quad ganador debe cubrir >=70% del area del marco (`MARCO_MIN_COVERAGE`) y ningun lado puede quedar "muy adentro" de la banda de margen — mata la CLASE del recorte mutilado (no solo el caso IMG_0352), y es el UNICO camino que aplica esta regla (still/galeria no la usan, por diseno); (c) compuerta de calidad DEGRADADA a consultiva (decision explicita de Mario tras revisar el spec): `qualityVerdict` ya NO bloquea ninguna captura (washedRatio, lowLightRatio, brillo, contraste, glare pasan a avisos en la vista previa) — el UNICO freno que queda es `extremeBlur` nuevo en docscan.js (`EXTREME_BLUR_SHARPNESS=15`/`EXTREME_BLUR_RATIO=1.35`), solo para que la auto-captura no dispare con la tarjeta en pleno movimiento; el obturador manual nunca pasa por el; (d) obturador manual sin quad: captura el rectangulo del marco tal cual (el usuario alineo la credencial al marco y presiono el boton) — red de seguridad garantizada, util cuando el CV no engancha (p.ej. webcam de escritorio). **RECORTE MANUAL (cropperjs) ELIMINADO 2026-07-17 por decision de Mario** (commit `3fb1348`): se borraron `ManualCrop.jsx`/`manualCrop.js`/su test y la dependencia cropperjs; el obturador sin quad ahora recorta el rectangulo del marco (sin cropper) y la galeria sin deteccion usa la foto completa. vitest 109 (antes 114, -5 tests de manualCrop).
  - **Que murio** (dead code removido de Camera.jsx): la escalera de resolucion en vivo (`LIVE_DETECT_WIDTHS` 480/720/960), la valvula de wash, el boton `offerManual`, y `frameGuidance` aplicado al camino en vivo (se conserva la funcion pero relativa al marco).
  - **Verificado**: vitest frontend 114 (incluye `marco.test.js` nuevo 20 casos), arnes worker 21/21 (sin cambios, worker intacto), backend 23, build limpio. Runner nuevo `frontend/scripts/video-replay.cjs` (extrae frames con ffmpeg ~5fps y replica el flujo en vivo completo — ROI de marco, confirmacion de 2 frames, gate de movimiento/inclinacion, extremeBlur — contra el worker REAL, sin mocks) sobre 23 videos: **4/23 capturan** (IMG_0331/0336/0340/0342, score 93-96, recortes inspeccionados a ojo: credenciales completas sin mutilar). WATCH: el desglose por camara es desparejo — de los 14 videos de iPhone (`IMG_03xx`) capturan 4 (~29%), pero de los **9 videos de webcam** (`Movie on 16-07-26...`) capturan **0** (dan cero quads validos en todo el clip); vigilar el camino de webcam de escritorio en la validacion en vivo. Salvedad honesta: NINGUNO de los 23 videos se grabo con el marco visible (no existia aun) — el harness aproxima con un marco virtual centrado, asi que 4/23 NO es la tasa real de exito del modo marco, es artefacto de metraje. Corpus de fotos migrado a `Resources/Examples/Fotos/` (25 fotos: 17 viejas + 8 nuevas), `corpus-run.cjs` default nuevo con conversion HEIC automatica via sips: 0 regresiones en las 17 viejas.
  - **Criterios de exito del spec** (evaluados en Task 7, sin cambio de umbrales — la evidencia no lo pedia): C1 (>=90% captura en video) **NO MEDIBLE LIMPIAMENTE** por la salvedad de metraje sin marco (cota 4/23); C2 (ningun mutilado ACEPTA en camino de marco) **CUMPLIDO**; C3 (ninguna foto legible bloqueada por calidad/extremeBlur) **CUMPLIDO** con margen amplio (peor foto real del corpus: sharp=33/ratio=3.55 vs umbral 15/1.35, >=2x de margen); C4 (suites verdes) **CUMPLIDO**.
  - **Limite conocido**: galeria/captura-still CONSERVAN la deteccion libre de v2.3 (sin ROI de marco, sin `validateQuadInMarco`) — el recorte mutilado tipo IMG_0352 sigue siendo posible por esa via; mitigado por la vista previa obligatoria (ya sin cropper como red desde la eliminacion del recorte manual). El OBTURADOR en modo marco tampoco aplica `validateQuadInMarco` en su happy-path (detecta con `detectDocumentStill` dentro del ROI y manda ese recorte a la vista previa; sin quad recorta el rectangulo del marco). La regla >=70% solo blinda la AUTO-captura.
  - **Reverso en webcam de escritorio — DIAGNOSTICADO Y ARREGLADO (2026-07-17, commit `781b43f`)**: Mario reporto que el reverso no engancha en escritorio. Diagnostico con 6 videos nuevos de webcam (`~/Downloads/nuevas pruebas`, scripts scratch replay-diag/sweep): con el marco a `MARCO_WIDTH_FRAC=0.8` la credencial de escritorio queda ~40% del ancho del frame (webcam gran angular + distancia de brazo) → por debajo de `minAreaRatio` → el worker devuelve **0 esquinas SIEMPRE** (ni frente ni reverso a esa distancia; el reverso, casi todo QR/bajo contraste, es donde mas se nota). NO era problema de cobertura: bajar `minAreaRatio` no ayuda porque la regla `>=70%` sigue rechazando una tarjeta chica en un marco grande — hay que ACHICAR el marco. Barrido: a `widthFrac~0.42` la tarjeta llena el ROI y detecta (cobertura 73-107%). Fix: **marco segun dispositivo** — `MARCO_WIDTH_FRAC_DESKTOP=0.42` (marco.js) que Camera.jsx elige via `isMobileDevice()`; movil sigue en 0.8 (el frente en movil, que ya era perfecto, no se toca). Replay completo sobre los 6 videos: 0/6 → 1/6 con captura + varios videos ya con quads validos (el resto no encadena 3 frames estables por ser metraje pasivo sin marco visible; en vivo con el marco para apuntar sube). Recorte del reverso capturado: completo, sin mutilar. Nuevo override `MARCO_WF` en `video-replay.cjs` para replicar el marco de escritorio sobre metraje de webcam. **PENDIENTE**: validacion en vivo de Mario del reverso en escritorio con el marco a 0.42 (¿se siente bien el marco mas chico? ¿captura a la primera?). Palancas si hace falta afinar: `MARCO_WIDTH_FRAC_DESKTOP` (marco.js), y en ultimo caso `minAreaRatio` del detect en vivo (Camera.jsx).
  - Worker `scan-worker.js` SIN cambios esta ronda (cache-bust sigue en `?v=13`, no subio).

## Verificacion — como validar cambios al escaneo

Arnes headless que corre el `scan-worker.js` REAL en Node (21/21 al corte 2026-07-16), ya dentro
del repo: `cd frontend && node scripts/worker-harness-test.cjs` (no usa AWS; casos:
tarjeta rotada, escena-cuarto, cuadrado, distractores, frente bajo-contraste+dedo,
solo-QR, reverso-con-QR, metricas quality, frente-INE-encabezado+mano, tarjeta
vertical con/sin allowPortrait, washedRatio, frame stats, cabeza-blob, tarjeta
en mano al aire via Hough, estanteria). Ademas: `npm test` en
ambas carpetas (frontend 114, backend 23 al corte 2026-07-17) y `npm run build` en frontend. El
corpus oficial de fotos vive en `Resources/Examples/Fotos/` (25 fotos: 17 viejas + 8
nuevas IMG_0350-0357, HEIC convertido con sips automatico) — `cd frontend && node
scripts/corpus-run.cjs` corre ahi por default (compara worker baseline contra HEAD,
no usa AWS). Para el camino EN VIVO (marco): `cd frontend && node
scripts/video-replay.cjs` replica el flujo completo (ROI de marco, confirmacion,
extremeBlur) contra el worker REAL sobre los videos de `Resources/Examples/Videos/`
(necesita ffmpeg); recortes capturados quedan en `frontend/replay-out/` para
inspeccion visual, gitignorado. Ambas carpetas de `Resources/Examples/` contienen
PII real de Mario y viven fuera del repo (fuera de git).

## Calibracion con corpus real (2026-07-23, Task 8 del plan grabadora — primera vuelta del loop)

- **Corpus**: 70 sesiones reales de Mario grabadas contra la VPS (`?record=1`), sincronizadas a `SCANNER TESTING/sesiones/` y etiquetadas (18 buena / 2 recorte-malo / 50 no-capturo; etiquetas escritas por Claude inspeccionando capture.jpg — auditables en el visor).
- **Bug de grabadora cazado por el gate de fidelidad** (`bd624e5`): finish() anulaba el id antes de que el toBlob asincrono subiera el frame decisivo (POST a /scan-recordings/null/) — TODA sesion auto quedo con frames = capturedSeq-1. Fidelidad 50/70 con el 100%% de divergencias explicadas (frame perdido o ganadas del modo contenido pre-cambio). Arreglado; las sesiones futuras traen el frame decisivo.
- **Umbrales adoptados** (`c1d50fa`, sweep 4 combos x 70 sesiones, 10 recortes del ganador inspeccionados sin mutilacion): `minAreaRatio` 0.25 -> **0.15** (Camera.jsx; una credencial a ~55%% del marco moria ahi antes de los gates) y `MARCO_CONTAINED_COVERAGE` 0.5 -> **0.42**. Ganancias: 28d9 (timeout 20s en vivo) -> captura perfecta; 45db (manual) -> completa y legible; 4994 sale mejor que su captura viva. `minAreaRatio` 0.10 no aporta.
- **Limites del corpus (NO son de umbrales)**: la mayoria de los 38 manuales fallan por (a) credencial blanca sobre fondo CLARO — el detector no ve los bordes (el tip "fondo liso y oscuro" existe pero el uso real es la mesa blanca; candidato: guia mas fuerte en UI o trabajo de worker con evidencia); (b) reverso de escritorio en mano: motion blur + dedos rompiendo el contorno.
- **Pendiente para cerrar el loop (paso 7)**: ronda corta de Mario con la calibracion nueva + grabadora arreglada (reverso movil y reverso escritorio primero, idealmente sobre fondo oscuro para separar la familia de contraste). Si captura bien -> C4 cumplido y a decidir merge.

## Calibracion 2a vuelta — reverso (2026-07-24, rama `mejora-reverso`, corpus 56 reverso + 22 frente)

- **Corpus**: 46 sesiones nuevas de Mario desde la VPS (post-merge a main) + las 32 de reverso conservadas de la 1a vuelta (las de frente viejas se borraron a peticion de Mario: foco en reverso). En vivo el reverso capturaba solo 8/56 (14%).
- **Causa raiz encontrada (sonda con worker instrumentado)**: el candidato tipico del reverso es el rectangulo INTERIOR del contenido impreso (barcode->MRZ) — mismo aspecto ~1.586 que la tarjeta, lados sobre plastico liso — y moria en `quadEdgeSupport` con cobertura 0.10-0.27 (umbral 0.45). `refineToOuterEdges` no podia rescatarlo porque elige por aspecto, que aqui EMPATA con el interior. Por eso el reverso "parpadeaba": dependia de que V1 diera un contorno perfecto en el tick exacto.
- **Fix del worker** (`924cbd8`, cache-bust `?v=14`): `refineToSupportedEdges` — cuando un candidato pasa zonas-lisas pero falla soporte, se expande a las lineas de borde fisicas eligiendo por COBERTURA (aspecto desempata) y se re-exige `quadEdgeSupport` sobre el refinado. Estrictamente aditivo; anti-mutilado intacto. Replay reverso: capturas 5 -> 19, 19/19 recortes completos.
- **Umbrales adoptados** (`0b6e5a3`, sweep + replays dirigidos): `STABLE_NEEDED` 3 -> **2** (Camera.jsx + espejo en session-replay-core; +7 reverso +2 frente, 0 perdidas) y `MARCO_CONTAINED_COVERAGE` 0.42 -> **0.55** (la banda 0.42-0.55 resulto ~50%% recortes malos: 9654 mutilado y 9085 lavado mueren; 18/19 buenas de reverso se conservan; frente 8 -> 9 neto). Resultado final corpus reverso: **8/56 vivo -> 23/56 replay, cero recortes mutilados o lavados**.
- **Blanco-sobre-blanco NO es arreglable a este costo** (probado hasta umbral adaptativo V4: 0 candidatos): se guia al usuario — `marcoGuidance` acepta `noQuadStreak` y tras `MARCO_NOQUAD_TIP_TICKS` (20 ticks ~6-7s) sin quad sugiere "Prueba sobre un fondo oscuro" (`1b2cbb9`).
- **Pendiente**: validacion EN VIVO de Mario con la rama `mejora-reverso` desplegada; si el reverso captura bien -> merge a main.

## Modo contenido + pulido del paso de camara (2026-07-22 tarde, pedidos de Mario en la VPS)

- **Captura por contencion (`MARCO_CONTAINED_COVERAGE = 0.5`, marco.js)**: una credencial bien detectada DENTRO del marco captura aunque no lo llene — antes solo capturaba el camino alineado (cobertura >= 0.70 + lados en banda). El piso 0.5 es el nuevo guardian anti-mutilado del vivo (bajo el se sigue pidiendo "Acercala"); `containedCoverage: 1` restaura el modo estricto y es palanca sweepeable (passthrough en session-replay-core). El veredicto trae `contained: true`; los bordes detectados se pintan (el quad ahora entra a la rama confirmada). Evidencia: video-replay paso de 4/23 a **7/23**, incluidos 2 de los 9 videos de webcam que daban 0 (cov 53-55%) — los 3 recortes nuevos inspeccionados a ojo: credenciales COMPLETAS, sin mutilar. WATCH: validar el piso 0.5 con el corpus grabado de la calibracion.
- **Marco desde el primer frame**: el overlay del marco ya no espera al worker (`scannerActive`) — se siembra `marcoView` al hacer play; durante la carga del wasm ya NO se ve la guia vieja de esquinitas (esa era la "vista con el diseno anterior" reportada). La guia del fallback queda montada oculta (frameRef conserva layout para el pipeline heuristico).
- **Tips fuera de la zona de captura**: `.cam-tips` salio de `.camera-copy` (JSX) y en el stage ancho (>=760px `.is-camera`) se ancla como columna en la franja lateral izquierda; en movil fluye igual que antes. Transicion suave del shell angosto->ancho (`transition` en `.phone`).

## Fix stage de camara en escritorio (2026-07-22)

- Bug reportado por Mario en la VPS: recuadro de captura diminuto en escritorio. Causa raiz (verificada con numeros): la tarjeta `.phone` angosta (376-640px) + video 16:9 con `object-fit: cover` en stage vertical = solo se veia una rebanada (~35%) del FOV de la webcam, y el marco 0.42 aplicado a esa rebanada daba un ROI efectivo de ~15% del frame — 2.8x menor que el 42% del frame con que se calibro offline (video-replay usa marco virtual DEL FRAME). Fix: `.phone.is-camera` (App.jsx `fullCamera`) ensancha el stage a `min(1080px, 100vw-48px)` en >=760px solo durante los pasos de camara — el cover casi no recorta y el ROI vuelve a ~40% del frame. Movil intacto (media query). Los umbrales NO se tocaron.

## Hosting de prueba (Azure VM, 2026-07-15)

- URL: `https://20.9.194.109.sslip.io` — basic auth (usuario `digid`, password compartida por WhatsApp/aparte, no vive en este repo).
- VM Ubuntu 24.04 existente de Mario; alias SSH local `digid-kyc-vm` (`~/.ssh/config`), llave dedicada `~/.ssh/digid_kyc_azure` (copiada con `ssh-copy-id`, login por password deshabilitado de hecho al no usarse mas).
- Arquitectura: Caddy (TLS automatico Let's Encrypt via sslip.io, `basic_auth` en todo el sitio) sirve el build del frontend desde `/var/www/digid-kyc` (dueno `caddy:caddy`, fuera de `/home` para no abrir permisos del homedir) y hace `reverse_proxy` de `/api/*` a Express en `127.0.0.1:3001` (systemd `digid-kyc-backend.service`, `EnvironmentFile=~/kyc/backend/.env`). `ufw` (22/80/443) como capa extra sobre el NSG de Azure.
- Codigo en la VM via `rsync` a `~/kyc` (no git clone: el repo es privado, se evito crear una deploy key) — excluye `node_modules`, `backend/storage`, `frontend/certs`, `.git`, `.env`.
- **Actualizar tras cambios**: `rsync -az --delete --exclude node_modules --exclude backend/storage --exclude frontend/certs --exclude .git --exclude backend/.env -e "ssh -i ~/.ssh/digid_kyc_azure" ./ digid_dev@20.9.194.109:~/kyc/`, luego en la VM: `cd ~/kyc/frontend && npm ci && npm run build && sudo rsync -a --delete dist/ /var/www/digid-kyc/ && sudo chown -R caddy:caddy /var/www/digid-kyc`; si cambio el backend, `cd ~/kyc/backend && npm ci --omit=dev && sudo systemctl restart digid-kyc-backend`.
- **CD (2026-07-16)**: `.github/workflows/deploy.yml` corre este mismo procedimiento automatico en cada push a `main` (o manual via `workflow_dispatch`) — ya no hace falta actualizar a mano salvo debug. SSH via llave dedicada solo-CI `digid_kyc_ci_deploy` (agregada a `authorized_keys` de la VM, separada de la llave personal de Mario), guardada como secret `DEPLOY_SSH_KEY` en el repo de GitHub (junto con `DEPLOY_HOST`, `DEPLOY_USER`). Primer run verificado exitoso. Deuda: esta llave CI hereda el mismo sudo passwordless que `digid_dev` (necesario para el `chown`/`systemctl restart`); si se filtrara el secret de GitHub, alcanza para modificar la VM.
- **Deuda nueva**: `backend/.env` (keys AWS de jorgeP) ahora vive tambien en la VM (`~/kyc/backend/.env`, permisos 600) — mismo riesgo ya registrado abajo, ahora en dos lugares. Rotar las keys sigue pendiente y aplica doble.
- Pendiente de validar: corrida real de camara/liveness a traves de este dominio HTTPS (el flujo de escaneo+liveness aun no se probo end-to-end en esta URL, solo se verificaron API y carga del frontend por curl).

## Grabacion remota en la VPS (2026-07-22, spec `docs/superpowers/specs/2026-07-22-grabacion-remota-vps-diseno.md`)

- **Que es**: la grabadora de sesiones (`?record=1`, plan 2026-07-20) habilitada en la VPS para grabar el corpus de calibracion (Task 8) desde cualquier parte. La VPS sirve la rama **`escaneo-robusto`** (deploy manual `gh workflow run deploy.yml --ref escaneo-robusto`, run verificado; `main` NO se ha mergeado — sigue esperando la validacion en vivo, que es esta misma calibracion).
- **Grabar**: `https://20.9.194.109.sslip.io/?record=1` (basic auth de siempre). Los frames+decisiones caen en la VM en `/home/digid_dev/kyc-recordings/sesiones/` (FUERA de `~/kyc`: el rsync `--delete` del deploy no lo toca).
- **Candado (reemplaza al "nunca en produccion" del spec 2026-07-20)**: los endpoints `/api/scan-recordings/*` existen SOLO mientras `RECORDINGS_DIR=/home/digid_dev/kyc-recordings` viva en `~/kyc/backend/.env` de la VM. Proteccion: el basic auth de Caddy cubre todo `/api/*`. **Al terminar la calibracion**: quitar esa linea del `.env`, `sudo systemctl restart digid-kyc-backend`, borrar `~/kyc-recordings` y correr un ultimo `sync-sesiones`.
- **Bajar sesiones al corpus local**: `scripts/sync-sesiones.sh` (rsync VM → `SCANNER TESTING/sesiones/`); con `--limpiar` borra de la VM solo sesiones TERMINADAS ya verificadas en local (nunca una `incompleta` que podria estar grabandose). Ciclo completo verificado 2026-07-22: sesion de prueba grabada via endpoints en la VPS → sync → visor-ready → `--limpiar` dejo la VM en cero.
- El visor, `session-replay.cjs` y `session-sweep.cjs` no cambiaron: leen `SCANNER TESTING/sesiones/` como siempre. Subida movil ~0.5-1 MB/s durante el escaneo; huecos por red lenta quedan como `dropped` (el replay los reporta como `gaps=N`).

## Analisis de IP anti-VPN/Tor (2026-07-22, spec `docs/superpowers/specs/2026-07-22-ip-analysis-antifraude-diseno.md`)

- **Que es**: deteccion server-side de red anonima en `POST /verify`. Bloquea el
  veredicto (`approved=false`, via `verdict.ipCheckPassed`) si la IP es **Tor, VPN
  o proxy publico**; **datacenter** solo se reporta como flag. Todo el detalle cae
  en `analysis.ipCheck` (incluye `raw` saneado, sin la API key).
- **Fuentes (hibrido)**: lista de exit nodes de Tor (`backend/src/torList.js`,
  cache en `storage/ip-cache/tor-exit-list.txt`, refresco 12h) + proxycheck.io
  (`backend/src/ipAnalysis.js`) para VPN/proxy comercial.
- **Fail-open**: IP privada/loopback (siempre en dev local), sin API key, o API
  caida => `ipCheck.status='no_evaluado'`, NO bloquea. El bloqueo real solo ocurre
  en la VPS con IP publica.
- **Config** (`backend/.env`): `PROXYCHECK_API_KEY` (free tier 1000/dia),
  `IP_ANALYSIS_ENABLED` (default: on si hay key), `TOR_LIST_URL`,
  `IP_ANALYSIS_TOR_REFRESH_MS`, `IP_ANALYSIS_CACHE_TTL_MS`. **La key va tambien en
  el `.env` de la VPS** (`~/kyc/backend/.env`) — el rsync del deploy no sincroniza `.env`.
- **Privacidad**: la IP del usuario final se envia a proxycheck.io (tercero).
- **Fuera de alcance**: capturar la IP en la creacion de sesion (evasion: VPN al
  escanear + apagada al verificar); geolocalizacion; persistir la cache de IP.
- **Tests**: `backend/test/ipAnalysis.test.js` (14), `torList.test.js` (7),
  casos nuevos en `verify.test.js`, `frontend/src/lib/ipCheck.test.js` (4) — todo mockeado.

### Geolocalizacion (2026-07-23, spec `docs/superpowers/specs/2026-07-23-geolocalizacion-ip-diseno.md`)

- **Que es**: `ipCheck.geo` (pais/ciudad/estado/ISP/organizacion/lat-long) via
  ipgeolocation.io (`backend/src/ipGeo.js`), llamado SIEMPRE en paralelo en cada
  `/verify`. **SOLO informativo**: NO afecta `approved`. Fail-open: `geo:null` ante
  cualquier fallo, nunca rompe el verify.
- **Independiente de la deteccion**: key y cuota propias (`IPGEOLOCATION_API_KEY`);
  funciona aunque proxycheck/tor esten desactivados y viceversa.
- **Limite de la key de Mario**: plan SIN modulo de seguridad — solo geolocalizacion
  (el `include=security` de ipgeolocation da 401). NO es backup de la deteccion
  VPN/Tor; solo garantiza pais/ISP en el reporte aunque proxycheck caiga.
- **Config**: `IPGEOLOCATION_API_KEY`, `IP_GEO_CACHE_TTL_MS` (default 24h). **La key
  va tambien en el `.env` de la VPS** (`~/kyc/backend/.env`) — el rsync no lo sincroniza.
- **Privacidad**: la IP del usuario final ahora tambien se envia a ipgeolocation.io.

#### Direccion aproximada (reverse-geocoding, 2026-07-23, spec `docs/superpowers/specs/2026-07-23-reverse-geocoding-diseno.md`)

- **Que es**: `ipCheck.geo.approxAddress` = `formatted_address` de Google Maps
  Geocoding sobre las coords de ipgeolocation (`backend/src/reverseGeo.js`,
  inyectado en `ipGeo`). Viaja con `approxAddressNote` que lo rotula como
  APROXIMADO nivel-ciudad (NO la direccion del usuario). Report-only, fail-open.
- **Advertencia clave**: las coords de IP son nivel-ciudad; la direccion es
  precision fabricada (edificio del centroide, no del usuario). NO usar para
  verificar la direccion real. El `approxAddressNote` existe para eso.
- **Config**: `GOOGLE_MAPS_API_KEY` (Geocoding API + cuenta de facturacion de
  Google; ~$5/1000 tras el credito, mitigado por cache por coord). Sin key =>
  `approxAddress:null`. **Va tambien en el `.env` de la VPS**.
- **Privacidad**: las coords de IP se envian a Google.

## Modulo de licencia (2026-08-12, plan `docs/superpowers/plans/2026-08-12-modulo-licencia.md`)

- **Que es**: el escaneo ahora corre por **perfil de documento** en vez de
  constantes fijas. `frontend/src/lib/docProfiles.js` (nuevo, objeto puro sin
  DOM) define `INE_PROFILE` (copia literal de la calibracion historica, con
  test candado que lo fija byte a byte) y `CREDENCIAL_PROFILE` (calibrado esta
  ronda para licencia de conducir / credenciales tamano tarjeta genericas).
  `getProfile(docType)` devuelve `INE_PROFILE` si `docType` es `undefined` o
  desconocido — default seguro. El worker (`scan-worker.js`) acepta `geom` en
  el mensaje `detect`: sin `geom` se comporta byte a byte como antes (contrato
  verificado por el caso 21 del arnes + el caso 22, candado de contaminacion,
  ver abajo). Cache-bust del worker en `?v=15`. `docscan.js` reenvia el
  perfil; `Camera.jsx`/`App.jsx` lo consumen (deteccion, marco, extraccion,
  gates de movimiento, copy de chip/pistas); la grabadora de sesiones registra
  `docType`/`profileId`. En `App.jsx`, la opcion "Licencia de conducir" ya NO
  muestra el badge "Proximamente" (Pasaporte si lo sigue mostrando).

### Bateria de cierre (Task 10, corrida y verificada el 2026-08-25/26)

Los cinco resultados de esta corrida, en la rama `escaneo-robusto`:

| Medicion | Resultado |
|---|---|
| `cd frontend && npx vitest run` | **163/163** |
| `npm run build` | limpio, `built in 21.11s` (unico aviso: chunk >500kB, preexistente) |
| `node scripts/worker-harness-test.cjs` | **23/23** (nota: el Step 1 del brief de la Tarea 10 decia "22/22" — numero desactualizado de antes de la Tarea 4, que agrego el caso 22 de candado de contaminacion; 23/23 es el numero correcto y vigente) |
| `node scripts/corpus-run.cjs` | **0 regresiones** — las 25 lineas de veredicto salen **identicas linea por linea** a las registradas en `task-9-report.md` (comparacion directa, no solo el resumen) |
| `cd ../backend && npm test` | **76/76** (el `stderr: Error: boom` que imprime es un test de error mockeado, no un fallo) |

INE intacto confirmado por esta misma corrida: vitest, arnes y corpus-run
arriba. El replay de INE (**8/23 identico linea por linea al baseline**) NO se
re-corrio en esta tarea — no esta en el Step 1 del brief y ya quedo medido y
documentado en `task-9-report.md`; se cita, no se repite.

### Numeros de la calibracion de credencial (de tareas anteriores del mismo plan, citados aqui — no re-medidos en la Tarea 10)

- Baseline sobre los 35 videos de `Resources/Examples/licencias/`: **16/35**
  (Tarea 1, tras corregir la deriva del arnes — ver boveda Obsidian). Tras la
  calibracion (Tarea 9): **24/35**, con los **24 recortes inspeccionados uno
  por uno sin mutilados**.
- Valores adoptados, SOLO en `CREDENCIAL_PROFILE` (INE_PROFILE intacto):
  - `marco.containedCoverage`: 0.55 -> 0.45.
  - `motion.confirmMovement` / `motion.movementMax`: 0.05/0.045 -> **0.08/0.08**
    los dos (aflojar solo uno no recupera nada).
  - `geom.houghSupportMinWorst`: 0.70 -> **0.66**.
  - El clamp de `extract` (`aspectClampMin`/`aspectClampMax`) **NO se toco**:
    ninguna medicion del barrido lo respaldaba (mediana del aspecto ganador
    1.62, comodo dentro de 1.35-1.9).
- **Decision explicita de Mario sobre `houghSupportMinWorst`**: se queda en
  **0.66 (24/35)**. El escalon de retirada, **0.68 (23/35)**, se midio,
  inspecciono y documento por si mas adelante se prefiere prudencia — cuesta
  exactamente 1 captura (IMG_0886, el recorte mas al ras del conjunto: pierde
  media linea de microimpresion legal pero conserva banda magnetica, panel de
  firma y numero). Esto es una decision humana, no un hallazgo tecnico: los
  datos por si solos no distinguen si 0.70 discrimina bien o si estas tarjetas
  caen apenas debajo (los ganadores por Hough en este corpus tienen peor-lado
  >= 0.71 y los rechazados <= 0.69). Detalle completo, rejilla barrida valor
  por valor y los 24 recortes: `docs/superpowers/diagnosticos/2026-08-12-calibracion-credencial.md`.
- **Perdida aceptada**: `orient: false` en `CREDENCIAL_PROFILE` — el perfil
  credencial NO endereza el 180 (a diferencia del INE, que endereza segun la
  franja gris arriba / MRZ abajo). La licencia no tiene esas senales
  exclusivas para decidir el 180 automaticamente; decision de diseno, no un
  umbral pendiente de barrer.
- Documentos de evidencia ya commiteados (consultar ahi el detalle, no
  repetirlo): `docs/superpowers/diagnosticos/2026-08-12-baseline-credencial.md`,
  `docs/superpowers/diagnosticos/2026-08-12-credencial-donde-mueren-los-candidatos.md`,
  `docs/superpowers/diagnosticos/2026-08-12-calibracion-credencial.md`, spec
  `docs/superpowers/specs/2026-08-12-modulo-licencia-diseno.md`.

### Verificacion visual pendiente (checklist de la Tarea 6, nunca ejecutada — sin navegador en el entorno de los subagentes)

Vive en `.superpowers/sdd/2026-08-12-modulo-licencia/task-6-report.md`
(gitignored); se copia aqui completa para que no se pierda. Con
`cd backend && npm run dev` y `cd frontend && npm run dev`:

1. México -> **Licencia de conducir**: la opcion ya NO debe mostrar el badge
   "Próximamente" y debe ser seleccionable (Pasaporte si debe seguir
   mostrandolo).
2. Camara del frente: el chip bajo el titulo debe decir **"Licencia de
   conducir"** (no "Credencial para votar (INE)").
3. Reverso: la pista de la guia (visible solo mientras opencv/worker cargan,
   camino `!scannerActive`) debe decir **"Reverso: el lado de atrás"**, SIN
   mencionar el codigo QR.
4. Repetir el flujo con **Credencial de elector** y confirmar que el chip
   dice "Credencial para votar (INE)" y la pista del reverso "Reverso: el
   lado con el código QR" — copy identico al de antes del cambio.
5. Con `?debug=1`, mover la credencial y confirmar que el panel muestra
   `movimiento (0.0xx >= 0.045)` en el perfil INE y `movimiento (0.0xx >=
   0.08)` en el perfil credencial (antes de esta tarea los dos compartian
   0.045; ahora deben diferir).
6. (De la Tarea 10) Con la licencia real de Mario: frente y reverso en
   movil, frente y reverso en webcam de escritorio, y una credencial
   DISTINTA (bancaria o de trabajo) para confirmar que el modulo detecta
   cualquier tarjeta ID-1, no solo licencias.

### Pendientes y deudas (los 7 que cierran esta tarea)

1. **C4 — validacion en vivo de Mario, PENDIENTE.** Licencia en movil (frente
   y reverso) y en webcam de escritorio (frente y reverso), mas una
   credencial distinta (bancaria o de trabajo) para confirmar que el modulo
   es generico. **Hasta que esto pase, el modulo NO se da por bueno**: los 35
   videos de la calibracion se grabaron SIN el marco visible (no existia
   metraje con marco) y el arnes de replay solo lo aproxima con un marco
   virtual centrado — 24/35 es el techo de ESE metraje, no la tasa real de
   captura en vivo. Se fusiona con la checklist visual de la Tarea 6 de
   arriba: correr ambas en la misma sesion con Mario.
2. **Checklist visual de la Tarea 6** (los 6 puntos de arriba): nunca se pudo
   ejercitar por falta de navegador/camara en el entorno de los subagentes.
   Queda copiada arriba completa; se corre junto con el punto 1.
3. **Los umbrales de movimiento 0.08/0.08 necesitan confirmacion en vivo.**
   El replay tickea a ~5 fps; la app en vivo tickea mas lento, y a menos
   ticks por segundo el movimiento medido entre frames crece — el gate puede
   comportarse mas estricto de lo medido en el replay. Watch item explicito
   dejado en el comentario de `docProfiles.js`.
4. **8 de los 11 fallos restantes del corpus de 35 NO son recuperables por
   umbral**: la tarjeta esta genuinamente chica dentro del ROI del marco,
   artefacto de que ese metraje se grabo sin el marco visible guiando al
   usuario a acercarla. No es una palanca de calibracion pendiente.
5. **Deuda deferida a proposito**: `frontend/scripts/session-replay-core.cjs`
   (el arnes que replay-ea sesiones grabadas en la VPS, distinto de
   `video-replay.cjs`) **NO** recibio soporte de perfil en este plan — sigue
   con su propia copia de los umbrales de movimiento (`:121`), sin nocion de
   `docType`/perfil. Como la grabadora ya registra `docType`, la proxima
   calibracion que use sesiones de licencia grabadas en la VPS va a
   encontrar ese arnes ciego al perfil y va a tener que darle soporte de
   perfil importando `docProfiles.js`, igual que se hizo en
   `video-replay.cjs` (`await import()` funciona desde CJS).
6. **RESUELTO el 2026-08-27 (ronda de arreglos de la revision final)**:
   `video-replay.cjs` ya **no** espeja los perfiles a mano — los importa de
   `docProfiles.js` con `await import()`, como ya hacia con `docscan.js` y
   `marco.js` (la premisa "CJS no puede importar ESM" era falsa). No queda
   copia que desincronizar. Dos notas de uso: con `PROFILE=ine` el mensaje
   `detect` va sin `geom` a proposito (mide el camino de los defaults del
   worker), asi que `SUPPORT_MIN`/`HOUGH_MIN_WORST` se rechazan con ese
   perfil; y cualquier override por env que no parsee a numero finito mata
   el arnes en vez de degradar a NaN. Verificado sin cambio de medicion:
   24/35 en credencial y 8/23 en INE, linea por linea.
7. **El clamp de extraccion esta hardcodeado en el arnes** (`video-replay.cjs`):
   no lee `docProfiles.js` para ese valor, asi que esa palanca no es
   barrible hoy sin tocar el arnes primero.

## Modulo de pasaporte (2026-08-28, rama `escaneo-pasaporte`, spec `docs/superpowers/specs/2026-08-28-escaneo-pasaporte-diseno.md`)

**Worktree de trabajo**: `/Volumes/LaCie/DIGID/KYC ITNERNAL/PROJECT V1 PASAPORTE`
(no `BRANCH B` — ese nombre lo tomo otra sesion en paralelo).

- **Que es**: escaneo automatico de la pagina de datos del pasaporte, como
  **modulo aparte** del escaner de INE/licencia (opcion B del spec: nada de
  `docProfiles.js`). Archivos nuevos en `frontend/`: `src/lib/passportProfile.js`
  (umbrales puros; tiene `marco.aspect` = 1.42 porque la pagina de datos es
  **TD-3 de ICAO 9303** — 125x88mm — y no ID-1 como las tarjetas; NO tiene
  campo `orient`, el pasaporte no endereza 180 por decision explicita),
  `src/lib/passportGuidance.js` (fork del copy de `marcoGuidance`),
  `src/lib/docFlow.js` (modulo puro: cuantos lados tiene cada documento; el
  pasaporte es UNO solo y conserva la clave `'front'` para subir `front.jpg`
  sin tocar el backend), `src/components/PassportCamera.jsx` (fork del tick de
  `Camera.jsx` al commit `33e4715`), `scripts/passport-replay.cjs` (fork de
  `video-replay.cjs`, arnes de calibracion). Se tocaron ademas `App.jsx`
  (router) y tres componentes de copy (`Progress`, `Preview`, `Instruction`),
  cada uno con un prop OPCIONAL cuyo default es el comportamiento de hoy.
- **Garantia central de la rama**: `Camera.jsx`, `docscan.js`, `marco.js`,
  `docProfiles.js`, `scan-worker.js`, `video-replay.cjs` y todo `backend/`
  quedan **byte-identicos** — verificado con `git diff --stat 33e4715..HEAD`
  sobre esos paths: **sin salida**.
- **Bateria de regresion, toda verde** (conteos tras el merge de `main` del
  2026-09-07): backend 206 tests PASS; frontend 253 tests PASS;
  `cd frontend && node scripts/worker-harness-test.cjs` 23/23 PASS; `npm run
  build` del frontend OK.
- **Calibracion (medida, no intuida)**: corpus de **26 videos reales** (PII de
  Mario, `Resources/Examples/pasaportes/`, fuera del repo). Resultado:
  **10/26 -> 12/26** capturas automaticas moviendo UN SOLO umbral (`motion`
  de 0.05/0.045 a 0.08/0.08). Documento completo:
  `docs/superpowers/diagnosticos/2026-08-28-pasaporte-calibracion.md`;
  medicion previa: `docs/superpowers/diagnosticos/2026-08-28-pasaporte-medicion-inicial.md`;
  corpus (clip por clip, con una correccion de dato el 2026-08-28: `IMG_1670`
  paso de `normal` a `espejado`, 7 de 26 clips espejados en vez de 6):
  `docs/superpowers/diagnosticos/2026-08-28-pasaporte-corpus.md`.
- **Verificacion en vivo por LAN — PENDIENTE.** La hace Mario, con telefono y
  pasaporte real (`cd frontend && npm run dev` con `frontend/certs/`
  presente). Todavia NO ocurrio. No se afirma que se hizo.
- **Fusionado en `main` el 2026-09-07** junto con el OCR de API Market: el
  pasaporte corre por `/ocr/front` con `docType: 'passport'` (backend ya lo
  esperaba con un solo lado); la verificacion en vivo por LAN sigue
  PENDIENTE.

### Deuda viva del modulo de pasaporte

1. **La deriva del fork.** Hay TRES pares fork/original: `PassportCamera.jsx`
   / `Camera.jsx`, `passportGuidance.js` / `marcoGuidance` (`marco.js`),
   `passport-replay.cjs` / `video-replay.cjs`. Un arreglo de LOGICA en
   cualquier original debe espejarse en su fork, y al reves. Solo el copy y
   los umbrales divergen a proposito — es el modo de falla conocido de la
   opcion B, sin mas defensa que las notas en cada archivo mas la disciplina.
2. **El umbral de movimiento calibrado necesita confirmacion real.** El
   replay tickea a ~5 fps; la app en vivo tickea mas lento, y a menos ticks
   por segundo el movimiento medido entre frames crece — el `0.08` calibrado
   en replay puede comportarse mas estricto en vivo de lo medido. Esto es
   exactamente lo que la verificacion pendiente del punto anterior debe
   confirmar.
3. **7 videos del corpus no producen NI UN cuadrilatero** (IMG_1652, 1659,
   1675, 1676, 1677, 1681, 1686) y ninguna palanca de `passportProfile.js` los
   toca. La hipotesis evaluada — que la pagina de datos en una libreta
   abierta solo tiene 3 bordes libres de 4 (el cuarto es el lomo) — quedo
   **debilitada, no confirmada**. Es trabajo pendiente, no de calibracion.
4. **Riesgo aceptado: sin correccion de 180 grados**, un pasaporte sostenido
   al reves produce un recorte invertido y falla el face match (decision
   explicita: la geometria no puede resolverlo porque un rectangulo es
   simetrico bajo ese giro, y la senal de contenido se descarto). **El
   alcance real es mas ancho que "sostenido al reves"**: `detectDocumentStill`
   (en `docscan.js`) acepta el documento VERTICAL y reetiqueta las esquinas
   para que el recorte salga acostado, girando **siempre en el mismo
   sentido**. En el INE eso lo repara `orientDocumentForStep`; el pasaporte lo
   omite por decision. Consecuencia: **una foto de GALERIA de un pasaporte en
   vertical — la forma natural de fotografiar una pagina de pasaporte — sale
   invertida aproximadamente la mitad de las veces, sin que el usuario haga
   nada raro.** El camino EN VIVO es inmune (ahi la deteccion corre sin
   `allowPortrait`), y por eso el corpus de video no podia medirlo. **Ningun
   clip del corpus de 26 esta rotado 180** — el riesgo NO esta medido, ni a
   favor ni en contra.
5. `video-replay.cjs` conserva su clamp de extraccion hardcodeado para los
   perfiles ine/credencial (ver punto 7 de la seccion anterior), y
   `session-replay-core.cjs` sigue sin nocion de perfil — ninguno de los dos
   se toco en este modulo.
6. **Falta probar a mano, en vivo, el escaneo de INE y el de licencia**
   despues de este cambio (criterio §7.5.4 del spec: "los tests prueban que
   el codigo no se movio; esto prueba que la app tampoco"). No se hizo y no
   estaba registrado. Importa porque `App.jsx`, `Progress.jsx`, `Preview.jsx`
   e `Instruction.jsx` SI cambiaron y SI los usa el INE, y este repo no tiene
   tests de componentes React. Atenuante: para `docType === 'ine'` la ruta es
   demostrablemente identica por inspeccion (los defaults de los tres props
   nuevos son el comportamiento anterior), asi que el riesgo es bajo — pero
   la comprobacion empirica falta.
7. **El corpus de calibracion de `passportProfile.js` es de un solo
   pasaporte.** Los 26 videos son **un solo pasaporte mexicano biometrico,
   filmado por una sola persona con un solo telefono**. El 12/26 no dice
   nada sobre otros pasaportes, otros paises ni otras camaras.
8. **12/26 es 46%.** Implicacion de producto: mas de la mitad de los
   escaneos de pasaporte van a terminar en el obturador manual (que recorta
   el rectangulo del marco), no en captura automatica.
9. **Falta un candado de paridad para los otros dos pares de fork.** El
   candado nuevo (`passportGuidance.test.js`, bloque "paridad con
   marcoGuidance") solo cubre la guia. Quedan sin candado mecanico
   `PassportCamera.jsx`/`Camera.jsx` y `passport-replay.cjs`/`video-replay.cjs`.
   Idea barata registrada por la revision: un test que guarde el diff
   esperado entre los dos componentes/scripts y falle cuando cambie.
10. **Dos limites de arquitectura documentados, no arreglados** (arreglarlos
    exigiria editar `docscan.js`, que esta rama promete no tocar):
    - `clampExtractAspect` en `docscan.js` tiene un fallback a 1.586 (ID-1)
      **no parametrizable**, mientras que `passport-replay.cjs` cae en 1.42.
      Si `detection.aspect` faltara, la app y el arnes medirian cosas
      distintas. En la practica es inalcanzable (el worker siempre devuelve
      `aspect` junto con las esquinas), pero el spec lo listaba entre lo que
      "ya recibe por parametro lo que el pasaporte necesita", y no del todo.
    - Mientras el worker de OpenCV inicializa, el tick cae a un fallback
      heuristico legado (`scan.js`) afinado al INE, que puede auto-capturar
      el frame completo sin marco ni nocion de TD-3. Es reuso deliberado
      segun el spec, pero es una ventana real donde el pasaporte se captura
      con criterios de tarjeta, y el arnes de replay no la simula.
## Mapas de ubicacion en el resultado (2026-08-28, mergeado desde `main`, plan `docs/superpowers/plans/2026-08-28-mapas-leaflet-tres-fuentes.md`)

- **Que es**: mapas desnudos de Leaflet con tiles de OpenStreetMap (sin key, sin
  iframe de Google) en la vista de resultado, uno por FUENTE con coordenadas.
  `backend/src/locations.js` (puro) arma `analysis.locations` en cada `/verify`:
  `{ sources: { gps, ip, document }, distances: [] }`, y cada fuente es un punto
  (`latitude`/`longitude`) mas `radiusM` (null = punto, numero = circulo), `precision`,
  `trust` y un `mapsLink` publico SIN key. Frontend: `frontend/src/lib/locations.js`
  (puro) + `LocationMaps.jsx` (que tarjetas hay) + `LocationMap.jsx` (el mapa).
  Report-only, fail-open, no afecta `approved`.
  - `gps`: ubicacion del firmante, capturada por el navegador
    (`frontend/src/lib/geolocate.js` -> `api.sendGeo` -> `POST /sessions/:id/geo` ->
    `meta.geo`). Es el UNICO dato de la feature que viene del cliente y viaja
    rotulado como tal (`trust: 'cliente'`): es falsificable con las devtools.
  - `ip`: coords de `ipCheck.geo`, dibujadas como circulo de 2 km (`IP_RADIUS_M`)
    porque la geo por IP tiene error de kilometros.
  - `document`: sale de `document.geo`, NO de `address.raw` — Leaflet pinta
    coordenadas, no texto. Convertir el domicilio a coordenadas es trabajo del
    adaptador `forwardGeo.locate(address) -> {latitude, longitude}|null`, que
    `server.js` inyecta **como `null` hoy**: mientras no haya geocodificador, la
    tarjeta del documento simplemente no aparece. `/verify` lo llama sobre
    `analysis.document.address.raw` (fail-open, nunca imprime el domicilio).
- **La key de Google Maps Embed murio con esta entrega**: no hay
  `GET /api/maps-config`, ni `fetchMapsConfig`, ni `GOOGLE_MAPS_EMBED_KEY`, ni
  `embedQuery`. `GOOGLE_MAPS_API_KEY` sigue viva pero SOLO server-side, para el
  reverse-geocoding de `ipGeo`. El candado de tests que impide que una key del
  entorno entre en `analysis.json` se quedo.
- **El domicilio del documento ya no es un hueco**: el adaptador `documentOcr`
  existe (`backend/src/documentOcr.js`, API Market OCR ID) y `server.js` lo inyecta a
  partir de `APIMARKET_TOKEN`; el OCR corre AL CAPTURAR cada lado y `/verify` lee la
  cache, asi que `analysis.document.address.raw` ya viaja en el analysis. Lo que
  falta para el tercer mapa es `forwardGeo`, no el OCR. **Salvedad grande: todavia
  no se ha hecho ni una llamada real al proveedor** y `ocr.Address` para la INE-G NO
  esta confirmado en el contrato — ver la seccion **OCR de documento con API Market**
  mas abajo.
- **`distances` siempre `[]` en esta entrega**: con GPS e IP ya hay dos fuentes
  con coords reales, asi que la distancia SI es calculable — pero su presentacion
  exige un piso de 25 km ("consistentes (< 25 km)", no un numero exacto) porque la
  geo por IP tiene error a nivel kilometros. Se decide con diseño propio, no aqui;
  esta vacio a proposito, no es un bug.

## OCR de documento con API Market (2026-09-04, rama `ocr-apimarket`, con `origin/main` ya mergeado dentro desde el 2026-09-07, spec `docs/superpowers/specs/2026-09-04-ocr-apimarket-diseno.md`, plan `docs/superpowers/plans/2026-09-04-ocr-apimarket.md`)

### Estado de despliegue (2026-09-07)

- **`ocr-apimarket` esta fusionada en `main`** (merge 1dc1d84: `origin/main` con los mapas
  Leaflet + geo del firmante entro en la rama, y la rama se empujo a `main` como
  fast-forward). **La VPS vuelve a servirse por el CD** (`deploy.yml` en cada push a
  `main`): Leaflet + OCR juntos. Entre el 2026-09-07 por la manana y ese push, la VPS
  corrio un rsync manual de `ocr-apimarket` SIN los mapas Leaflet (regresion temporal,
  ya corregida por el CD).
- El `.env` de la VPS tiene las cinco claves del OCR con `APIMARKET_TOKEN` real y
  `APIMARKET_SANDBOX=false` (el CD excluye `backend/.env`, asi que sobrevive); el
  arranque NO muestra `[documentOcr] sin APIMARKET_TOKEN`: el gate de documento esta
  ACTIVO alli y cada sesion completa gasta 2 unidades (hasta 6 si hay rechazos con
  `OCR_MAX_ATTEMPTS=3`) **segun la cifra que se tenia por buena en ese momento —
  correccion 2026-09-22: `ocr-id` cobra 2 unidades por llamada, no 1 (medido
  contra `GET /api/balance`), y la VPS sigue en el diseno de DOS llamadas por
  sesion (§ Llamada unica + pre-filtro, mas abajo, confirma que no esta
  desplegada ahi todavia), asi que el costo real desplegado hoy es 4 unidades
  por sesion feliz, hasta 12 con rechazos**. Saldo al desplegar: 20 unidades.
  Cero llamadas reales hechas; nada de la checklist de abajo se ha ejecutado
  (orden de Mario: no probar hasta nuevo aviso). La VPS corre Node 20 (el SDK de
  AWS avisa que exigira >=22 en 2027).
- Token tambien en `backend/.env` de `PROJECT V1` y de `PROJECT V1 OCR`.
- `forwardGeo` sigue inyectado como `null`: el mapa del domicilio del documento no
  aparece hasta escribir `backend/src/forwardGeo.js` (el OCR ya entrega `address.raw`).
- Ramas locales `mapas-leaflet` (checkout `PROJECT V1`) y `mapas-leaflet-impl` quedaron
  detras de `main`; no borradas.

### Que se construyo

> **2026-09-07 — el CRUCE de datos frente/reverso esta ELIMINADO.** La primera
> sesion REAL (INE-G autentica de Mario) fue rechazada dos veces con
> `cross_check_failed` y agoto el reverso: la VIZ del frente mete el apellido
> materno en `GivenNames` (la MRZ de la combinada no lo trae) y el proveedor
> ademas emitio `ocr_id_validation_sides_curp_mismatch` y
> `ocr_id_validation_curp_name_mismatch` sobre ese documento autentico. Decision
> de Mario: quitar el cruce por completo. **El reverso se acepta por tipo + lado
> (aportacion de MRZ)**; ya no existen `crossCheck`, `cross-check.json` ni la
> razon `cross_check_failed`, y `verdict.documentPassed` es `true` con OCR activo
> (llegar a Rekognition ya implica ambos lados aceptados) o `null` sin OCR. Lo
> que sigue abajo describe el diseno original salvo donde se diga lo contrario.

> **2026-09-22 — SUPERADO por la llamada unica.** Todo lo que sigue de aqui
> hasta el final de la seccion "Que se construyo" (contrato del endpoint,
> "2 unidades por sesion feliz", `evaluateBack`, `ocr-front.json`/`ocr-back.json`
> como par, `mergeLicenseData`) describe el **diseno de DOS llamadas, ya
> superado** en la rama `ocr-llamada-unica` (no mergeada a `main` todavia). Se
> deja como historia para entender como se llego hasta aqui; el contrato
> vigente, con la llamada unica y el pre-filtro, esta en
> `### Llamada unica + pre-filtro (2026-09-22, rama `ocr-llamada-unica`)`, mas
> abajo en esta misma seccion de OCR.
>
> **Nota 2026-09-22 sobre las unidades de esta seccion historica**: "2 unidades
> por sesion feliz" (dos llamadas) asumia `x-price: 1` del OpenAPI del
> proveedor. Medido despues contra `GET /api/balance`, `ocr-id` siempre cobro
> **2 unidades por llamada**, no 1 — el diseno de dos llamadas costaba en
> realidad **4 unidades** por sesion feliz, no 2. No es un cambio de precio del
> proveedor, es una correccion de la documentacion. Cifras corregidas en § Llamada
> unica + pre-filtro, mas abajo.

- **Proveedor**: API Market OCR ID (`POST https://apimarket.mx/api/ai/v2/ocr-id`). Guia
  completa del contrato, costos y trampas: `docs/memory/api-apimarket-ocr-id.md`.
- **Adaptador** `backend/src/documentOcr.js` — `createDocumentOcr({ token, timeoutMs,
  sandbox })` devuelve `null` si no hay token. Expone `analyze({ files })` y
  `fetchCropped(data)`. Solo transporta: multipart `files[]` (con corchetes),
  `Authorization: Bearer`, `x-sandbox` si aplica; `AbortController` propio (nada fuera
  del adaptador corta la llamada); **sin reintentos**; clasifica los fallos en
  `transient` (429, 5xx, timeout, red) vs `config` (400/401/422 y codigos de
  configuracion). Nunca loguea el token ni campos del `ocr`: solo `status`,
  `error_code` y `codigoValidacion`.
- **Reglas puras** `backend/src/documentRules.js` — `sniffImageType`,
  `classifyDocument` (tipo por `document_name`/`document_country`/`document_type`;
  lado por retrato/VIZ/MRZ), `expectedDocument`, `evaluateSide` (FRENTE),
  `evaluateBack` (REVERSO, por contribucion de la MRZ a la respuesta combinada),
  `listValidationCodes`, `extractDocument`, `stripSignedUrls`.
- **Recortes** `backend/src/croppedImages.js` — `downloadCropped` soporta data URI y
  URL firmada (GET con timeout, `content-type: image/*`, 5 MB), fail-open por imagen.
- **Punto de llamada: el upload de cada lado, NO `/verify`.** El frente se manda solo;
  el reverso se manda **frente + reverso** en una sola llamada para que el proveedor
  correlacione lados (`SIDES`). **2 unidades por sesion feliz, +1 por cada rechazo.**
- **Frontend**: `api.ocrDocument()`, `continueFromPreview`/`runDocumentOcr` en
  `App.jsx`, vista `DocumentRejected.jsx`, textos en `src/lib/ocrMessages.js`, filas de
  resultado en `src/lib/document.js` + `ResultsView` null-safe para el `analysis`
  parcial, y compresion unica en `src/lib/imageBudget.js` (lado mayor 1600 px, tope
  1.8 MB, escalera de calidad 0.92 -> 0.7 y caida a 1200 px) aplicada a TODA imagen
  antes de subir, venga de camara, obturador manual o galeria.

**Contrato del endpoint `POST /api/sessions/:id/ocr/:kind`** (body `{ docType }`,
`kind` en `front|back`, `docType` en `ine|license|passport`):

| Codigo | Cuerpo | Nota |
|---|---|---|
| 404 | `{ error: 'ocr_disabled' }` | sin `APIMARKET_TOKEN`; el frontend avanza como antes |
| 400 | `kind debe ser front o back` / `docType debe ser ine, license o passport` / `{ error: 'docType_mismatch' }` / `{ error: 'side_not_expected' }` | el `docType` se fija en la primera llamada; `side_not_expected` (pasaporte + `/ocr/back`) corta **antes** de pagarle al proveedor |
| 409 | `{ error: 'image_missing' }` / `{ error: 'front_required' }` / `{ error: 'attempts_exhausted', attemptsLeft: 0, analysis }` | precondiciones |
| 200 aceptado | `{ accepted: true, attemptsLeft, document, debug, files }` | `document` = `extractDocument(...)`; `files` = nombres reales de `portrait.*`/`signature.*` |
| 200 rechazo | `{ accepted: false, reason, consumed, attemptsLeft, exhausted, classification, expected, debug, codigoValidacion, analysis? }` | `reason` en `no_document`, `wrong_document_type`, `wrong_side`, `curp_invalid` (gastan intento) y `too_large` / `unreadable` (`consumed: false`) |
| 502 | `{ error: 'ocr_transient' \| 'ocr_config', retryable, errorCode, codigoValidacion }` | nunca gasta intento; `ocr_transient` ofrece "Reintentar analisis" |

El upload (`POST /api/sessions/:id/images/:kind`) es la compuerta de formato: JPEG/PNG
**por firma de bytes** (400 si no lo es, aunque el dataURL diga otra cosa) y tope duro
`OCR_MAX_IMAGE_BYTES` (413 `image_too_large`). PDF fuera por diseno.

**Estado en la sesion** (`session.json`): `docType`, `ocrAttempts: { front, back }`
(inicializados en la primera llamada a `/ocr`) y `documentRejected`. Intentos **por
lado**, `OCR_MAX_ATTEMPTS` (default 3). Gastan intento solo los dictamenes sobre el
contenido; NO gastan `too_large`, los transitorios, los de configuracion ni el timeout
de captura de `Camera.jsx` (vence antes de subir nada). Archivos que quedan en
`backend/storage/<uuid>/`: `ocr-front.json`, `ocr-back.json` (respuesta cruda **sin**
`cropped_images`), `ocr-<kind>-rechazo-<n>.json` +
`<kind>-rechazo-<n>.jpg` (copia de los bytes que se analizaron: la recaptura
sobreescribe `<kind>.jpg`), `portrait.<ext>` y `signature.<ext>` (extension por firma de
bytes, el nombre real vive en `ocr-front.json.files`), y un `analysis.json` **parcial**
si se agotan los intentos de un lado (la sesion termina rechazada, sin liveness).

**`/verify`** ya no llama al OCR: lee la cache. Con OCR habilitado responde 409
`document_rejected` (sesion ya rechazada) o `document_incomplete` (falta un lado
aceptado) **antes** de gastar Rekognition; `analysis.document` sale de
`extractDocument` del ultimo lado aceptado mas `{ docType, files, sides }`;
`verdict.documentPassed` es `true` con OCR activo (llegar hasta aqui ya implica que
todos los lados esperados estan aceptados) y `null` sin OCR; entra en `approved`;
`verdict.thresholds` gana `ocrMaxAttempts`.

**CURP como dato de validacion del INE (2026-09-07 noche, decision de Mario).** El frente
del INE debe traer la CURP: `evaluateSide` exige `ocr.Curp` con forma de CURP
(`curpLooksValid`, 18 caracteres) y sin `ocr_id_validation_curp_checksum_fail`; si no,
`curp_invalid` (gasta intento del frente, `debug.curpPresent` dice si faltaba). El codigo
`curp_name_mismatch` del proveedor ya NO rechaza: lo emitio sobre la INE-G autentica de la
primera sesion real.

**Licencia: acepta todo, fusiona y exige un dato (2026-09-07 tarde, decision de Mario).**
Para `docType: 'license'` no hay gate por foto: `evaluateSide`/`evaluateBack` aceptan
siempre (`debug.rule = 'license.acceptAll'`; la clasificacion y la traza se calculan
igual, solo para diagnostico). Al cerrar el reverso, `mergeLicenseData` fusiona frente +
combinada y `licenseRequiredData` busca nombre, fecha de nacimiento y domicilio. Con AL
MENOS UNO pasa: `ocr-back.json` guarda la fusion (`data`) y `required: {found, missing}`,
que `/verify` copia a `analysis.document.required` y el resultado pinta como "Datos
encontrados" / "Faltantes". Sin ninguno: 409 `license_no_data` con `restart: true`,
NO gasta intento, cierra la sesion (`meta.documentRejected` + `analysis.json` parcial con
`document.rejected.required`) y la pantalla solo ofrece "Iniciar de nuevo" (recarga =
sesion nueva). Evidencia: `ocr-back-rechazo-<timestamp>.json` + `back-rechazo-<timestamp>.jpg`.
INE y pasaporte no cambian. Sin evidencia real de licencia todavia: los nombres de campo
que busca `licenseRequiredData` (`FullName`, `Nombres`/`GivenNames`, `ApellidoPaterno`/
`Surname`, `DateOfBirth`/`FechaNacimiento`, `Address`/`Domicilio`) son los del contrato
de INE; la primera licencia real dira si el proveedor usa otros.

**Modo debug (2026-09-07)**. Antes, el JSON de una verificacion rechazada solo
decia `reason` y `classification`: no decia QUE senal fallo. Ahora cada evaluacion
del OCR (`evaluateSide`/`evaluateBack`) devuelve una traza `debug` con
`{ rule, expected, detectScore, detectScoreMin, signals: { front, combined },
mrzContributed, providerCodes: { VERIF, MRZ, OCR, INE, SIDES }, providerErrors,
providerSuggestions, imageQuality, globalScore, valid }` — **solo codigos,
banderas y numeros, NUNCA valores de campos** (esos ya estan en
`ocr-<kind>-rechazo-<n>.json`). Ademas `classification.signals` trae
`{ portrait, viz, mrz, ocrFields, mrzFields }`. La traza viaja en las respuestas
200/409 de `/ocr/:kind`, en `ocr-front.json`/`ocr-back.json`, en los JSON de
rechazo, en `analysis.document.rejected.debug` y en
`analysis.document.sides.<kind>.debug`; el frontend la resume en la fila
"Diagnostico OCR" de `ResultsView` (`describeOcrDebug` en `src/lib/document.js`).
La bandera `OCR_DEBUG=true` (env) **solo** enciende ademas una linea por consola
por evaluacion (`[ocr] session=... kind=... attempt=... accepted=... reason=...
type=... side=... score=... signals=... codes=... codigoValidacion=...`); sin la
bandera la traza JSON sigue viajando igual.

**Razon `unreadable` (2026-09-07)**. Un frente que el proveedor no pudo leer
vuelve con `ocr = {}` y `MRZ = {}` (respuesta 200): antes se rotulaba
`wrong_side` y gastaba un intento con motivo enganoso. Ahora hay razon propia
`unreadable` (se evalua despues de `no_document`/`wrong_document_type` y antes de
`wrong_side`) que **NO gasta intento** (`consumed: false`, como `too_large`). La
evidencia se guarda igual, con sufijo de timestamp para no chocar con la
numeracion de los rechazos que si consumen:
`ocr-<kind>-unreadable-<ISO con : y . como ->.json` + `<kind>-unreadable-<ts>.jpg`.

**Variables de entorno** (`backend/.env.example` ya las documenta):
`APIMARKET_TOKEN` (sin el, el OCR no existe), `APIMARKET_OCR_TIMEOUT_MS=20000`,
`APIMARKET_SANDBOX=false`, `OCR_MAX_ATTEMPTS=3`, `OCR_MAX_IMAGE_BYTES=2097152`,
`OCR_DEBUG=false`.
**El `backend/.env` real de PROJECT V1 ya tiene las cinco claves, con
`APIMARKET_TOKEN` vacio** — asi el prototipo sigue corriendo como antes (404
`ocr_disabled`, `/verify` sin gate de documento) hasta que llegue el token.

### Como se construyo

Ejecucion subagent-driven sobre el plan de 12 tareas: 11 tareas de codigo (cada una con
su implementador y su revision independiente), revision final de rama completa
(`cda1f87..f58d16a`) y una **ola de fixes** con los 12 items triados (informe en el
ledger). Todos los commits viven en la rama `ocr-apimarket` del worktree
`/Volumes/LaCie/DIGID/KYC ITNERNAL/PROJECT V1 OCR`. Ledger completo (gitignored):
`.superpowers/sdd/2026-09-04-ocr-apimarket/`. Suites al cierre: **backend 202/202,
frontend 199/199, `npm run build` limpio** (desde 98 y 172 de base).

### Decisiones tomadas durante la ejecucion (rulings del ledger)

- El `docType` de la sesion se **fija en la primera llamada a `/ocr`** y no cambia (400 `docType_mismatch`), en vez de ser cambiable hasta aceptar el frente.
- `ocrAttempts` se **inicializa en la primera llamada a `/ocr`**, no al crear la sesion; la ruta trata su ausencia como ceros.
- Retrato y firma se guardan con la **extension que dicta la firma de bytes** (`portrait.jpg` o `.png`), no siempre `.png`; el nombre real queda en `ocr-front.json.files`.
- `fetchCropped` **se expone en el adaptador** (mismo `fetch` inyectado) en vez de inyectar `croppedImages` en `createApp`: los tests doblan un solo objeto.
- El reverso se juzga **por contribucion** (`evaluateBack`): la combinada es una fusion, asi que el reverso es reverso si la MRZ aparece ahi y no estaba en el frente solo.
- **400 y 422 son `config`**, no rechazo del usuario: el formato, la cantidad y el tipo de archivo los controla nuestro backend, no la captura.
- Un **abort o corte al leer el cuerpo** de la respuesta se clasifica como `transient` (timeout/red), no como `config`.
- Un **502 sin `error: 'ocr_config'`** (el manejador generico de Express tambien responde 502) se trata como transitorio y reintentable: lo contrario dejaba la sesion sin salida.
- `curp_checksum_fail` / `curp_name_mismatch` gatean el **FRENTE** (`curp_invalid`, gasta intento del frente: la CURP y los nombres estan impresos ahi). OJO (2026-09-07): el proveedor emitio `curp_name_mismatch` sobre una INE-G autentica; el gate se conserva porque solo mira el frente, pero es candidato a revisar con mas evidencia real.
- **Cruce de datos frente/reverso ELIMINADO (2026-09-07)**: ver la nota del inicio de esta seccion.
- **`acceptedImage` por lado en `App.jsx`**: volver atras y darle a Continuar con la MISMA foto ya aceptada avanza sin subir ni volver a pagarle al proveedor (una recaptura siempre re-analiza; aceptar un frente nuevo limpia el reverso recordado).
- **No se guarda `document` en el estado de `App`** (la spec lo mencionaba): nadie lo consume, el resultado lo lee de `/verify`. Codigo muerto evitado.
- `extractDocument` **no normaliza fechas**: el contrato dice que el proveedor aplica `date_to_iso` antes de puntuar. A confirmar con la llamada real (item 9 de la checklist).

### Deuda conocida (minors deferidos, no bloquean)

- **Tests que faltan**: `MAX_CROPPED_BYTES` y cuerpo vacio en `croppedImages`; frontera exacta de `OCR_MAX_IMAGE_BYTES`; candado URL-free en los tests del reverso y en `ocr-<kind>-rechazo-<n>.json`; huecos en `ocrMessages`/`document.js` (default, dedupe, `attemptsLeft` null, portrait-only, sin `version`/`country`). **`App.jsx` no tiene suite** en este repo, asi que el atajo de `acceptedImage` esta cubierto solo por lectura.
- **Carreras que solo se toleran por ser prototipo**: el snapshot de `ocrAttempts` se pisa entre lados si dos llamadas corren a la vez (el flujo es secuencial); el `busy` se libera antes del `fetchAnalysis` del 409 (doble clic).
- **`app.js` crecio ~170 lineas** con el handler inline de `/ocr/:kind`: extraerlo a `src/ocrRoute.js` cuando entre el flujo de pasaporte.
- **PNG con alfa se aplana a negro** al reencodear a JPEG: no hay ningun `fillStyle` blanco antes del `drawImage` en `frontend/src` (el canvas nace transparente y el JPEG pinta esos pixeles de negro); el aviso de formato no compatible dura 2400 ms (tier `qualityHint`) — quiza corto para leerlo.
- **El reverso de licencia no tiene gate real**: sin MRZ no hay senal de contribucion, asi que un frente de licencia subido dos veces pasaria como reverso. Es el punto debil declarado del diseno.
- Otros: `classification.side` vale `'unknown'` en un reverso aceptado (nadie debe leerlo esperando `'back'`); el tipo puede caer a `unknown` si `document_country` viene null o el nombre viene en espanol; `ocr-front.json` puede dejar de describir a `front.jpg` si la imagen se re-sube sin re-analizar (el backend no guarda hash; `acceptedImage` solo lo mitiga desde el frontend); `side_not_expected` no tiene copy propio en el frontend (hoy es inalcanzable, el pasaporte esta marcado "Proximamente"); en modo `ocr_disabled` volver atras si re-sube la imagen (no hay costo que ahorrar, deliberado).
- **Retencion de PII del lado del proveedor sin confirmar por escrito** (item 15 de la checklist).

### Estado real y pendiente inmediato

**Estado real: NO se ha hecho ninguna llamada al proveedor.** Todo el codigo esta
escrito, revisado y con las suites verdes, pero el token nunca existio en esta sesion:
ni el sandbox ni una sesion real se han corrido, y por lo tanto **nada de lo que el
proveedor devuelve esta confirmado empiricamente**.

Pendiente inmediato, en orden:

1. `APIMARKET_SANDBOX=true` + token en `backend/.env` -> recorrer INE frente / reverso /
   liveness de punta a punta (gratis) para validar el cableado.
2. **Una** sesion real (2 unidades) con la INE de Mario por LAN, y guardar la respuesta
   cruda y el `analysis.json` en la bitacora de Obsidian.
3. La checklist de abajo, que es lo unico que no se puede resolver leyendo.

### Checklist de verificacion con el proveedor real

1. ~~`Sex` en la combinada con `source: 'MRZ'`: ¿traduce M/F a H/M?~~ RESUELTO 2026-09-07: `ocr.Sex` y `MRZ.fields.Sex` valen `"H"` (no M/F). Ya no importa: el cruce local se elimino. **Matiz 2026-09-22**: esto es cierto solo para la INE — el pasaporte de la misma persona devolvio `Sex: "M"` (ICAO crudo, sin traducir). Ver `docs/memory/api-apimarket-ocr-id.md` § Tercera llamada REAL.
2. `ocr.Address` en la respuesta del frente de una INE-G (habilita el mapa del INE).
3. ¿El bloque `validations.INE` viene en la respuesta del frente solo? Si solo viene en la combinada, mover el gate `curp_invalid` al reverso.
4. Forma de los bloques `validations.*` (arreglo de strings, `{code}` u objeto anidado).
5. `document_name`/`document_country` exactos para INE-G y licencias estatales (¿`Driving`/`Driver`/en espanol?); `document_type === 'P'` en pasaporte.
6. Heuristica de lado: respuesta de reverso solo (¿retrato null? ¿VIZ vacio? ¿MRZ?) y frente solo (¿MRZ vacia?).
7. Llamada combinada: MRZ presente; codigos `SIDES` para INE-G; si `detect_score` refleja la peticion o la peor imagen. (El cruce local que dependia de `scores[campo].source === 'MRZ'` ya no existe.)
8. `cropped_images`: URL firmada o data URI; `content-type` del GET; que `portrait.*` abra.
9. Formato de fechas de `ocr.DateOfBirth`/`DateOfExpiry` (se asume ISO).
10. `ocr_id_result_empty` (hoy 502 → transitorio, nunca gasta intento).
11. `FormData`+`Blob` de Node con `files[]` aceptado por su PHP; 401 vs scope `use_ai_ocr_id`.
12. La respuesta del sandbox pasa los gates de INE como dice la spec §5.
13. Latencia p50/p95 de la combinada y de las descargas de recortes → `APIMARKET_OCR_TIMEOUT_MS`.
14. Nuestros JPEG de 1600 px: `detect_score`/`class_similarity` y legibilidad de CURP/domicilio.
15. Politica de retencion de PII por escrito.

### Llamada unica + pre-filtro (2026-09-22, rama `ocr-llamada-unica`, spec `docs/superpowers/specs/2026-09-22-ocr-llamada-unica-prefiltro-diseno.md`, plan `docs/superpowers/plans/2026-09-22-ocr-llamada-unica-prefiltro.md`)

**Estado**: implementado y con suites verdes en la rama `ocr-llamada-unica` del
worktree `/Volumes/LaCie/DIGID/KYC ITNERNAL/PROJECT V1 OCR-UNICA` (backend 284,
frontend 280, build limpio). **El pre-filtro del REVERSO se elimino el mismo
2026-09-22, despues de la implementacion original** (decision de Mario): API
Market ya caza un reverso malo cuando recibe frente+reverso juntos, y el viaje
extra a OpenAI justo antes de la llamada pagada le subia latencia al flujo de
KYC; consecuencia aceptada, un reverso malo ahora cuesta 2 unidades del proveedor
en vez de cero (correccion 2026-09-22: `ocr-id` siempre cobro 2 unidades por
llamada, no 1 — medido contra `GET /api/balance`, ver mas abajo). El pre-filtro
del FRENTE se queda igual. **Todavia NO esta mergeada a `main` ni desplegada en
la VPS** — el `.env` de la VPS descrito arriba (§ Estado de despliegue 2026-09-07)
sigue en el diseno de 2 llamadas y sin `OPENAI_API_KEY`. Ejecucion subagent-driven
sobre 7 tasks; ledger completo (gitignored):
`.superpowers/sdd/2026-09-22-ocr-llamada-unica-prefiltro/progress.md`.

**Que cambia**: una sola llamada pagada a API Market por sesion (en vez de dos),
disparada al cerrar el reverso con `files[] = frente + reverso`, aplicable a INE y
a licencia. El frente del INE pasa primero por un **pre-filtro de vision**
(OpenAI, `backend/src/documentPrefilter.js`) que solo puede rechazar y nunca hace
OCR. El pasaporte no cambia. `evaluateBack` y `mergeLicenseData` ya no existen;
los sustituye `evaluateCombined`. Detalle tecnico completo, con nombres de
funciones y archivos, en `CLAUDE.md` § Arquitectura y § Trampas conocidas (item
9). El pre-filtro **solo esta activo si existe `OPENAI_API_KEY`** en `backend/.env`;
sin ella, el flujo corre exactamente igual que sin pre-filtro (una sola llamada
pagada, sin aviso temprano) — no es un fail-closed como la ausencia de
`APIMARKET_TOKEN`.

**Riesgos vivos, sin suavizar** (de la spec, seccion Riesgos declarados, mas los
que aparecieron durante la implementacion):

1. **`hasFrontViz` esta calibrado con un solo especimen** (la INE-G de Mario, tres
   llamadas reales sobre el mismo documento). Otras versiones de INE (D, E, F)
   podrian etiquetar sus campos de frente distinto y romper la deteccion de "el
   frente aporto".
2. **El pre-filtro gasta intentos.** Un falso rechazo del modelo (dice que la
   imagen esta mal cuando no lo esta) puede cerrarle la sesion a un usuario
   legitimo antes de que el proveedor real vea una sola imagen. Mitigado por el
   enum conservador (`unsure` cuenta como `pass`) y por el fail-open ante
   error/timeout, pero **no eliminado**: sigue siendo posible que un usuario
   legitimo agote `OCR_MAX_ATTEMPTS` por rechazos del pre-filtro solamente. Los
   archivos `prefiltro-front-*.json` de cada sesion son el mecanismo para medir
   esta tasa en produccion (solo del FRENTE: el pre-filtro del reverso se
   elimino, ver punto 7).
3. **Segundo procesador de PII.** Las imagenes del documento (INE/licencia) ahora
   viajan tambien a OpenAI, no solo a API Market. **Esto requiere actualizar el
   aviso de privacidad y revisar la politica de retencion antes de produccion —
   no es una decision tecnica**, es una decision de Mario/legal que sigue
   pendiente.
4. **La licencia perdio el merge frente+reverso.** Antes `mergeLicenseData`
   fusionaba la respuesta del frente sola con la combinada; ahora solo existe la
   combinada. Si esa unica respuesta omite un campo que la respuesta del frente
   sola si traia, `license_no_data` cerrara sesiones de licencia que antes
   pasaban. Sin evidencia real de licencia todavia (ver deuda arriba).
5. **Pendiente explicito: el eval de ~40 fotos** del corpus real
   (`Resources/Examples/`, PII real, fuera del repo) comparando `gpt-5.4-mini`
   (default de `OPENAI_MODEL`) contra Haiku 4.5, midiendo tasa de captura de
   basura y tasa de falso rechazo. Cuesta menos de 1 MXN y **no toca creditos de
   API Market** (el pre-filtro nunca llama al proveedor de OCR). Hasta que ese
   eval exista, la eleccion del modelo es una **hipotesis razonada, no un hecho
   medido** — por eso el modelo vive en la variable de entorno `OPENAI_MODEL` y
   no esta hardcodeado.
6. **La verificacion E2E real de esta rama sigue pendiente.** Nada de lo anterior
   se ha corrido contra el proveedor real (API Market ni OpenAI) en el flujo
   completo de esta rama — solo la llamada suelta del 2026-09-22 documentada en
   `docs/memory/api-apimarket-ocr-id.md` y las suites con todo mockeado. Antes de
   mergear/desplegar: repetir el pendiente inmediato de la seccion de arriba
   (sandbox de punta a punta, luego una sesion real) ya con el pre-filtro
   activado y midiendo el costo real contra la estimacion corregida de ~1.04
   MXN por INE feliz (piso con el plan de referencia de API Market, ~0.50
   MXN/unidad x 2 unidades por llamada, mas el check de pre-filtro).
   **Correccion 2026-09-22**: la estimacion original decia ~0.54 MXN asumiendo
   1 unidad por llamada; medido contra `GET /api/balance` (gratuito, antes y
   despues de cada llamada), `ocr-id` siempre cobro 2 unidades por llamada, no
   1 — ver CLAUDE.md § Costo AWS y `docs/memory/api-apimarket-ocr-id.md` §
   Costo y limites reales. El costo real de la unidad va de $0.40 a $1.00 MXN
   segun el plan contratado, y el piso del pre-filtro no cuenta los tokens de
   razonamiento de `gpt-5.4-mini`.
7. **El pre-filtro del REVERSO se elimino (2026-09-22, decision de Mario,
   post-implementacion).** Corria en ambos lados del INE en la implementacion
   original de esta task; el del reverso se quito porque API Market ya detecta
   un reverso malo (p.ej. "INE adelante, licencia atras") cuando recibe
   frente+reverso juntos, y el viaje extra a OpenAI justo antes de la llamada
   pagada le subia latencia al flujo de KYC. **Consecuencia aceptada**: un
   reverso malo ahora cuesta 2 unidades del proveedor en vez de cero (correccion
   2026-09-22: `ocr-id` siempre cobro 2 unidades por llamada, no 1, ver item 6
   arriba) — ya no hay aviso temprano ni gratis para ese caso. El pre-filtro del
   FRENTE (item 2 de arriba) no cambio.
8. **Extraccion por procedencia en `extractDocument` (2026-09-22, commits
   `57aef30`/`201cba3`/`c0caed0`, misma rama): riesgo de truncamiento de la MRZ
   sin medir, pendiente de verificar.** `extractDocument` ahora prefiere la MRZ
   sobre la VIZ para nombre/apellidos en INE y pasaporte (arregla un bug medido:
   la VIZ metio el apellido materno dentro de los nombres por un error de OCR,
   ver `docs/memory/api-apimarket-ocr-id.md` § Tercera llamada REAL). Pero la
   MRZ de la INE (formato TD-1 de ICAO 9303) tiene el campo de nombre de
   longitud FIJA: un nombre mexicano largo (dos nombres, dos apellidos largos)
   **se trunca ahi**. Este cambio sustituye un modo de falla YA MEDIDO (la VIZ
   con error de OCR) por uno **sin medir** (la MRZ truncada), y ademas la
   variante de la VIZ ya no queda guardada en `analysis.json` para poder
   comparar si el caso aparece. **Pendiente**: conseguir o esperar una INE real
   con nombre largo y revisar la MRZ a simple vista — el corte se ve sin gastar
   creditos del proveedor. Detalle completo (las tres trampas del cambio, con
   evidencia real y `codigoValidacion`) en `docs/memory/api-apimarket-ocr-id.md`
   § Implementación de la selección por procedencia, y en `CLAUDE.md` § Trampas
   conocidas (item 14).

## Vinculo por contenido entre el registro del OCR y la imagen (2026-09-22, arreglo de seguridad)

**El defecto (existia tambien en `main`, no lo introdujo esta rama).** El
registro aceptado en disco (`ocr-combinada.json`, o `ocr-front.json` para
pasaporte) decia "el proveedor valido este documento", pero nada lo ataba a los
bytes concretos de `front.jpg` — que es justo la imagen contra la que
`CompareFaces` compara la selfie del liveness. Secuencia reproducida con PoC:
el atacante sube la INE de la victima, llama a `POST /ocr/back`, y **mientras el
servidor espera al proveedor** (`analyze` + `fetchCropped`, varios segundos de
red) hace un `POST /images/front` con su propia cara. El handler termina y
escribe el registro aceptado con los datos de la victima; el liveness corre con
la cara del atacante; `CompareFaces` compara contra el `front.jpg` del atacante
y coincide; `/verify` responde `200 approved: true`. **Cara del atacante,
identidad de la victima.**

**Por que el candado por EVENTO no alcanzaba.** Ya existia la mitigacion de que
`POST /images/:kind` borre el registro aceptado (agregada en la revision
anterior). En la carrera el borrado corre *antes* de que el registro se escriba:
borra algo que todavia no existe. Ningun candado que dependa del ORDEN de los
eventos puede cerrar esto, porque el atacante elige el orden.

**El arreglo: atar el registro al CONTENIDO.** Cuando se hace la llamada pagada
se calcula el `sha256` de **los buffers exactos que se le mandaron al
proveedor** (el arreglo `files`, nunca una relectura de disco — releer
reintroduce la misma carrera) y se guarda en `record.imageHashes`. La combinada
de `/ocr/back` guarda `{front, back}`; el frente-solo del pasaporte, `{front}`.
`/verify` recomputa el hash de lo que hay en disco para **los lados que el
documento tiene** (`expectedDocument(docType).sides`, no una lista a mano) y
corta con **409 `document_image_mismatch`** (con `sides: [...]`) si alguno no
coincide. El corte va junto a `document_rejected` / `document_incomplete`:
**antes de gastar Rekognition**. El registro NO se borra en el corte — es
evidencia de lo que el proveedor vio frente a lo que hay ahora en disco.

**La misma carrera existia una segunda vez, DENTRO de `/verify`.** El gate leia
`front.jpg` de disco y, despues del `await` de `GetFaceLivenessSessionResults`
(segundos de red), se volvia a leer el archivo para `CompareFaces`: dos lecturas
distintas del mismo nombre con una ventana en medio, y el atacante subiendo su
cara justo ahi. Ahora `/verify` lee `front.jpg` **una sola vez**, al principio, y
ese mismo buffer es el que pasa el gate y el que se le entrega a
`CompareFaces`/`DetectFaces`. **REGLA: nunca releer `front.jpg` de disco despues
de un `await` en `/verify`** — que lo hasheado sea lo comparado tiene que valer
por construccion, no por otro chequeo mas.

**La mitigacion por evento se queda.** Es defensa en profundidad y da un error
mas comprensible (`document_incomplete`) en el caso normal de recaptura, que es
el 99% de los casos. No sustituirla por el hash ni al reves.

**Fail-closed a proposito.** Un registro aceptado SIN `imageHashes` (sesiones de
`main`, o de una version anterior a este vinculo) no tiene con que comprobarse:
`/verify` responde `document_image_mismatch` en vez de aprobar a ciegas.

**Consecuencia de despliegue.** Las sesiones que esten en vuelo en la VPS cuando
esto se despliegue tienen registro sin `imageHashes` y recibiran el 409 nuevo al
llegar a `/verify`: el usuario tiene que volver a escanear el documento (y se
pagan otras 2 unidades del proveedor, correccion 2026-09-22: `ocr-id` cobra 2
por llamada, no 1). Es una ventana corta y no hay forma de
migrarlas: el hash tiene que nacer de los bytes que el proveedor vio, y esos
bytes ya no se pueden reconstruir con confianza.

**Frontend**: `LivenessStep.jsx` mapea `document_image_mismatch` a un mensaje
propio ("La imagen de tu documento cambió después de validarse…"); sin el, el
usuario veia el mensaje generico de "la prueba de vida no se completó", que
apunta al lado equivocado.

**Tests que lo cubren** (backend 270): en `verify.test.js`, el bloque
`el registro esta atado a los BYTES analizados` reproduce **las dos** carreras de
forma determinista, sin depender de timing — el doble de `documentOcr.analyze`
ejecuta el `POST /images/front` del atacante *dentro* de la llamada pagada (INE y
pasaporte), y el doble de `GetFaceLivenessSessionResults` lo ejecuta *dentro* de
`/verify` (esa se mide por los bytes que recibio `CompareFaces`, no por el
status, porque Rekognition esta mockeado). Mas: fail-closed sin huella, reverso
cambiado, conservacion de la evidencia y el flujo normal aprobando. En
`ocrRoute.test.js`, que el registro lleva la huella de los buffers que recibio
`analyze`.

**Salida en la UI, con matiz.** La pantalla de error de `LivenessStep` ofrece
"Volver", que devuelve al ultimo paso del documento en vista previa. Pero el
frontend solo re-sube la imagen cuando cambio: si el usuario confirma la misma
vista previa, no hay upload, no hay `/ocr/back` y `/verify` vuelve a dar el mismo
409. La salida existe, pero pasa por **recapturar de verdad** — que es lo que
dice el mensaje. Si el 409 se vuelve comun (no deberia: en operacion normal lo
cubre la mitigacion por evento con `document_incomplete`), la UI deberia forzar
la recaptura en vez de dejar el "continuar" disponible.

## Cascada de validaciones gubernamentales (2026-09-23, rama `ocr-llamada-unica`, spec `docs/superpowers/specs/2026-09-22-validaciones-gubernamentales-diseno.md`, plan `docs/superpowers/plans/2026-09-23-validaciones-gubernamentales.md`)

**Estado**: implementada y revisada tarea por tarea (subagent-driven, 12 tareas 0-11; tareas
0-10 completas con revision independiente cada una, ledger completo gitignored en
`.superpowers/sdd/2026-09-23-validaciones-gubernamentales/progress.md`). Suites verdes al
cierre de la tarea 10 y re-confirmadas en la tarea 11 (documentacion): **backend 470/470,
frontend 305/305, `npm run build` limpio** (470 = 449 del plan + 6 del fix de lista nominal
+ 8 del fix post-plan de `readVigencia` + 7 de la revision final de rama, todos del 2026-09-23). Commits `3e7f48d..ad73db5` sobre la rama
`ocr-llamada-unica` (worktree `PROJECT V1 OCR-UNICA`), mas el commit de esta tarea 11. **NO
fusionada a `main`, NO desplegada en ningun lado, y NINGUNA sesion real corrida todavia** —
ni de este plan ni de la tarea 11: el permiso de Mario para gastar unidades reales de API
Market/datosnonstop sigue sin darse. Esta tarea 11 tampoco hizo llamadas de red de ningun tipo.

**Que corre**: dentro de `/verify`, DESPUES de Rekognition, siempre que haya OCR aceptado.
Con `govChecks` (`APIMARKET_TOKEN` presente y `GOV_CHECKS_ENABLED` != `'false'`) corre de
verdad; con `GOV_CHECKS_ENABLED=false` corre SIN cliente y toda INE / pasaporte MEX cae a
`pending_review` con las compuertas no preguntadas como razon (revision final de rama,
2026-09-23: la bandera ahorra unidades, no salta compuertas; licencia y pasaporte no MEX
siguen `approved`). Orden: compuerta `valida-curp` -> compuerta lista nominal (solo INE,
adaptador `datosnonstop`) -> ramas en paralelo (RFC/SAT, NSS/IMSS, acta/RENAPO,
cedula+constancia/SEP) bajo un deadline que las dos compuertas NO miran (solo su timeout de
adaptador; `DATOSNONSTOP_TIMEOUT_MS` subio a 45 s por los 28 s medidos en vivo). Un 404 HTTP
de datosnonstop es `failed` => `pending_review`, nunca rechazo.
Veredicto de tres estados: `approved` | `rejected` | `pending_review`. Detalle tecnico
completo (mecanica, archivos, funciones) en `CLAUDE.md` § Arquitectura (parrafo "Cascada
gubernamental") y § Trampas conocidas (items 15 a 22, que cubren: el mapeo MRZ de la lista
nominal, costo/latencia real, las dos unicas compuertas que rechazan, por que la lista
nominal no se apaga por configuracion, por que `pending_review` es pegajoso, el vinculo por
CONTENIDO del replay idempotente y el limite conocido de `GET /analysis`, por que los codigos
del proveedor viven en disco aunque no salgan al cliente, la falla intermitente de la suite,
y la trampa del fixture de MRZ redactado).

**Verificado SOLO con mocks — nada de esto se ha probado contra el proveedor real dentro de
este plan**:
- `valida-curp`, `obtener-curp`: contrato leido del OpenAPI descargado el 2026-09-22
  (`docs/memory/apimarket-gov-endpoints.md`), nunca llamados en vivo por este plan.
- `consultar-vigencia`, `obtener-rfc`/`validar-rfc`/`lista69b`/`obtener-certificados-rfc`/
  `calcular-rfc`, `localizar-nss`, `buscar-nacimiento`, `obtener-cedula`/`validar-cedula`/
  `descargar-constancia-profesional`: mismo caso, cero llamadas reales de este plan.
- **Lista nominal real**: el adaptador `datosnonstop` (tarea 3) esta escrito y probado contra
  fixtures; el mapeo `cic`/`idCiudadano` de la MRZ se confirmo UNA sola vez en vivo, ANTES de
  este plan, el 2026-09-22 (`task-3-evidencia-lista-nominal.md`, gitignored) — no contra una
  credencial "E" vigente real, y no dentro de este plan tampoco.
- `GET /api/balance`: la FORMA real de la respuesta sigue sin confirmar (el OpenAPI no trae
  schema ni ejemplo); `readBalance` lee de forma defensiva contra `BALANCE_KEYS`.

**Pendientes, en el orden en que conviene resolverlos**:

1. **Primera sesion REAL con la INE-G de Mario y `GOV_CHECKS_ENABLED=true`** (gasta ~11
   unidades de API Market + 1 token de datosnonstop, mas Rekognition ~$0.015 USD). Objetivo:
   confirmar `renapoSource`/`nssBranch`/`sepBranch` (`backend/src/govChecks/cascade.js`) contra
   las llaves REALES de cada respuesta (hoy se leen con `pick()` sobre listas de nombres
   candidatos, ninguno confirmado), revisar `gov-checks.json` completo, y medir la latencia
   real de `/verify` contra el techo de ~75 s documentado en `CLAUDE.md` item 16 (re-derivado
   en la revision final de rama: 10 saldo + 10 valida-curp + 45 lista nominal + 0 ramas + 10 saldo).
2. **Confirmar el mapeo de la MRZ** primero con el sandbox del proveedor de OCR y luego con
   una credencial "E" vigente real — el unico dato que existe hoy es una prueba puntual del
   2026-09-22, anterior a este plan (ver arriba).
3. **La forma real de `GET /api/balance`**: sin una llamada real, `BALANCE_KEYS`
   (`backend/src/govChecks/endpoints.js`) sigue siendo una lista candidata sin confirmar.
4. **El campo del lugar de nacimiento del pasaporte** (`PLACE_KEYS` en
   `backend/src/govChecks/curpHypotheses.js`: `PlaceOfBirth`/`BirthPlace`/`Lugar_Nacimiento`/
   `LugarNacimiento`/`Estado_Nacimiento`) — CERO sesiones de pasaporte en disco al cierre de la
   tarea 5 de este plan (10 ine, 5 license, segun quedo registrado ahi), asi que ninguno de los
   cinco nombres esta confirmado. Lo confirma la primera sesion real de pasaporte (punto 6).
5. **El scope IMSS `use_localizar_nss_curp`** falta en el token actual (`localizar-nss`
   responde 403), por eso la cascada feliz de INE mide 11 unidades y no las 13 declaradas
   (`consultar-vigencia` queda `skipped` sin NSS con que preguntar). Pedirlo al proveedor si se
   quiere medir la rama NSS completa alguna vez.
6. **Primera sesion real con PASAPORTE**: nunca ha ocurrido (ni en este plan ni en ninguno
   anterior). Confirmaria a la vez el punto 4 (lugar de nacimiento) y el comportamiento real de
   las hasta 4 hipotesis de CURP (`MAX_CURP_HYPOTHESES`, hasta 8 unidades de API Market).
7. **Verificacion visual EN VIVO de los tres estados del veredicto**, nunca ejercitada (sin
   sesion real ni permiso para gastar): el tono ambar del encabezado, el glifo de reloj en el
   circulo de 58px, y el espaciado alrededor (`frontend/src/components/ResultsView.jsx` y
   companeros de la tarea 10). Correr junto con el punto 1.
8. **Asimetria de UX conocida y aceptada, no un bug**: un pasaporte mexicano con la cascada
   APAGADA no pinta ninguna fila de CURP, mientras una INE en la misma condicion pinta "no
   verificada". Es real (hallazgo de la revision de la tarea 10) porque no existe una senal
   confiable para distinguir "pasaporte elegible sin cascada" de "pasaporte no elegible"; se
   acepta tal cual hasta que aparezca esa senal.
9. **Limpiar `backend/storage/` antes de exponerlo a nadie o de desplegar** — LOCAL y en el
   deploy de Azure VM (`~/kyc/backend/storage`, ver § Hosting de prueba arriba). Los
   `analysis.json`/`gov-checks.json`/`ocr-*.json` de sesiones de antes de esta rama todavia
   llevan los codigos crudos del proveedor (`validations`, `debug.providerCodes`) que C7 dejo
   de mandar al cliente pero que nunca se redactan al leer (ver CLAUDE.md item 20).
10. **Falla intermitente de la suite, sin investigar** (CLAUDE.md item 21): aparecio en
    `ocrRoute.test.js` (tareas 0 y 3) y `verify.test.js` (dos veces en la tarea 8, una en la
    tarea 10) con firmas que incluyen `expected 404 to be 409`; buscada a proposito en la tarea
    9, NO aparecio ahi. Siempre verde al reintentar. Necesita su propia investigacion
    (hipotesis de trabajo: carrera de FILESYSTEM entre tests que comparten la carpeta real de
    `storage`, no un mock). **Dato nuevo (tarea 11, 2026-09-23)**: en una corrida de
    `cd backend && npm test` durante esta misma tarea aparecio OTRA firma, `Parse Error:
    Expected HTTP/, RTSP/ or ICE/` (1 failed | 448 passed), verde de nuevo en el reintento
    inmediato (449/449). Es un error del parser HTTP de Node, no un `404`/`409` — huele mas a
    reuso de socket/puerto entre archivos de test en paralelo que a la carrera de filesystem
    de la hipotesis de arriba. Puede ser la MISMA falla con otra cara, o una segunda falla
    distinta; no se investigo mas por estar fuera del alcance de esta tarea (documentacion).
11. **Modelos I/J de la INE, fuera de alcance por decision explicita, no confirmados.** El
    modelo I (`GENERO NB`, junio 2026) ya se menciona en el spec (§ Aplazados por decision
    explicita): `obtener-curp` solo acepta el binario H/M, asi que una INE modelo I cae a
    `sex_not_binary` (advertencia, CLAUDE.md item 15d) y ese eslabon queda `skipped`. Para la
    LISTA NOMINAL el codigo hoy solo mapea modelos `D`-`H` (`listaNominalParams`,
    `backend/src/govChecks/datosnonstop.js:26-35`; el modelo `C` se excluye a proposito, fuera
    de vigencia desde 2019); cualquier otro valor de `version` — incluidos I y J — devuelve
    `null` y cae a `error: 'unsupported_model'`, que la cascada traduce a
    `lista_nominal_unavailable` -> `pending_review` (nunca `approved`, ver CLAUDE.md item 15b).
    **No confirmado**: si el modelo J existe ya como credencial real, y si I/J deberian tener
    su propio mapeo de MRZ para la lista nominal en vez de caer en `unsupported_model`. Sin
    una credencial modelo I o J real, no hay como saberlo.

**Riesgo operativo** (documentado tambien en `backend/.env.example` y CLAUDE.md item 17):
datosnonstop factura por TOKEN consumido, no por sesion — la nota de memoria
`ocr-apimarket-estado.md` de este usuario registra 14 tokens disponibles. Al agotarse, la
compuerta de lista nominal cae a `lista_nominal_unavailable` y **toda sesion de INE va a
`pending_review`, nunca a `approved`**, hasta recargar el saldo. No es un bypass ni una falla
de diseno: es el costo de que la lista nominal sea una compuerta real y no decorativa.

## Pendiente inmediato (bloquea dar por bueno el prototipo)

1. **Corrida E2E humana con camara y rostro real** (~$0.02 AWS): flujo completo hasta ResultsView. AVANCE 2026-07-14 noche: el pipeline de streaming YA se confirmo empiricamente en escritorio (Chrome perfil limpio via arnes: preflight -> camara -> websocket -> check corriendo hasta TIMEOUT por falta de rostro posando) — **`StartFaceLivenessSession` funciona con las credenciales de GetSessionToken**. Falta solo la corrida con rostro real hasta ResultsView (movil ya corre el check: capturas IMG_0299/0300). OJO escritorio: el navegador de Mario era SAFARI — macOS Safari solo muestra el aviso de camara si getUserMedia nace de un GESTO del usuario; `disableStartScreen` habia quitado el unico clic (el boton de AWS) y la peticion pendia muda para siempre. Fix: con permiso 'granted' arranca solo (movil/Chrome/flujo completo); con 'prompt' aparece el boton "Activar camara" (el clic es el gesto). Ademas: un solo getUserMedia siempre (WebKit entrega track mudo con abrir/soltar/reabrir — eso rompio movil temporalmente) + watchdog de DOM que convierte cualquier atasco en error guiado con Reintentar. `mkcert -install` ya corrido por Mario (higiene, no era la causa). ULTIMO fix de la cadena: Safari colapsaba el modulo a 2x4 px (camara corriendo invisible — el "puntito" en el centro) porque `width: min(100%, 380px)` resuelve el % contra un flex sin ancho definido; ahora `min(380px, calc(100vw - 76px))` (ancho definido, REGLA: nada de porcentajes de ancho en los amplify-* dentro de .liveness-view). Verificado en el Safari real de Mario: modulo 380x286, video vivo ~35s, check corriendo, auto-arranque con permiso ya concedido. Despues, a pedido de Mario, el check paso a PANTALLA COMPLETA del stage: cadena `position:absolute; inset:0` anclada a `.camera-view` (`.liveness-detector-check` con `!important` por su position:relative inline + `.amplify-liveness-camera-module`); video centrado a su aspecto SIN cover (el ovalo va en coordenadas del frame); loader de conexion y chip del watchdog con z-index encima. Verificado: modulo 638x858 full-stage con video vivo en Safari. Ajuste final: stage del check BLANCO plano (las franjas navy vs el scrim blanco parecian bug), cortina blanca con spinner hasta que el video da frames (estado `cameraLive` del watchdog), sin boton X ni indicador REC (display:none — salir del check queda solo por timeout, ok en prototipo).
2. **Calibracion con INE real**: umbrales de deteccion (`validateQuad`, smooth-ratio 0.12, `SNAP_*`) y calidad (`qualityVerdict`: borrosidad combinada sharp/ratio, washedRatio 5%, lowLightRatio 28%, brillo 60-215, glare 8%) recalibrados el 2026-07-14 con el corpus de 17 fotos (`~/Downloads/fotos pruebas/` — OJO: Mario reemplazo ahi el corpus anterior de 6; el caso "pulgar cruzando el borde inferior" quedo sin re-validar). La calibracion 2026-07-16 (escaneo v2.3) agrego a la lista de umbrales a vigilar: `SUPPORT_MIN_COVERAGE` (0.45, filtro de soporte de bordes de contornos) y los `HOUGH_*` (variante V3), en particular `HOUGH_SUPPORT_MIN_WORST=0.70` — WATCH ITEM: ese umbral solo esta validado contra el fixture sintetico del arnes (peor lado 0.792) y un quad Hough malo real (IMG_0285, peor lado 0.625); faltan 3-5 fotos REALES de tarjeta en mano al aire de Mario para confirmar que el rescate Hough sigue funcionando en condiciones reales y que fondos ruidosos distintos a madera-nocturna no inflan el soporte de un quad malformado. Nota 2026-07-17: la escalera 480/720/960 EN VIVO murio con el escaneo v3 (marco fijo, ROI a ancho fijo 720) — ya no aplica. La validacion EN VIVO pendiente ahora es la del MARCO (criterio 5 del spec `2026-07-17-escaneo-marco-guiado-diseno.md`): correr `npm run dev` (y movil por LAN con los certs) y validar con Mario sensacion del marco, captura a la primera en escritorio y movil, obturador→cropper prefijado, y 3-5 tomas de tarjeta en mano — registrar veredicto aqui cuando se corra (ver seccion "Validacion en vivo" del brief de Task 8 en `.superpowers/sdd/task-8-brief.md`, gitignored). Borderline conocido: heic-0289 rechaza por reflejo con 5.x% vs umbral 5% — decidir con Mario (nota: la calidad ya no bloquea en el camino EN VIVO desde v3, asi que este borderline solo importa en still/galeria). Watch item del E2E: si el anclaje del refinado parpadea entre frames, el gate de estabilidad solo alarga la captura (no produce recortes malos).
3. Guardar el `analysis.json` de la primera corrida real en la bitacora (evidencia de stripBytes y del verify feliz).

## Deuda registrada (no bloquea dev; si bloquea staging)

- **Rotar las access keys de jorgeP** (viajaron en texto plano por chat) y pasar de `GetSessionToken` a `GetFederationToken` o user+rol dedicado; agregar auth al backend (hoy /api/aws-credentials es abierto en localhost — y via el proxy de Vite queda alcanzable desde toda la LAN cuando el server corre con certs; solo redes de confianza).
- Region acoplada: `LivenessStep.jsx` fija `us-east-1`; si cambia `AWS_REGION` hay que tocarlo a mano (README lo advierte).
- `CompareFaces`: usar max-across-matches (el INE tiene foto fantasma; hoy se toma `FaceMatches[0]`).
- Mapear errores de body-parser a 4xx (hoy 502) y revisar limite 15mb vs fotos de galeria grandes.
- Minors triados en la revision final: lista completa en `.superpowers/sdd/progress.md`.
- Escalones futuros del escaneo (si la tasa de acierto con INEs reales queda corta): detector ML DocCornerNet/Scanic; fallback de recorte manual con cropperjs (ambos de la investigacion de Mario, ver addendum 2/3).
- **Fuga de dinero (a): la compuerta del documento no corre antes de la prueba de vida (hallazgo 2026-09-22).** El check de liveness de Amazon Rekognition (~$0.015 USD, la parte cara de la verificacion completa) lo factura AWS en cuanto el usuario mueve la cara frente a la camara en el navegador — **antes** de que el servidor llegue a `/verify`, que es donde vive la compuerta real (`document_rejected`/`document_incomplete`/`document_image_mismatch`). Consecuencia: una sesion cuyo documento ya esta condenado a fallar (INE ilegible, tipo equivocado, lado equivocado) paga la prueba de vida de todos modos. Arreglo propuesto, no implementado: espejar esas mismas compuertas en `POST /liveness`, antes de crear la sesion de Amazon. No es un cambio de seguridad — `/verify` conserva la compuerta real — solo evita pagar por una sesion que de todos modos se iba a rechazar.
- **Fuga de dinero (b): el reintento tras `document_image_mismatch` cuesta dinero en cada vuelta (hallazgo 2026-09-22).** `LivenessStep` crea una sesion de liveness nueva cada vez que se monta, y el frontend solo vuelve a subir la imagen del documento si cambio: si el usuario confirma la misma vista previa tras el 409, no hay upload ni llamada al OCR, pero el componente se remonta igual y AWS cobra otro check de liveness. Arreglo propuesto, no implementado: ademas del punto anterior, que la UI obligue a recapturar de verdad en vez de dejar disponible el boton "continuar" sobre la misma vista previa ya rechazada.
- **El analisis de costos `docs/mockups/kyc-costos-1000.md` / `kyc-costos.html` (2026-09-15) asumio 1 unidad por llamada a `ocr-id`; quedo corto (2026-09-22).** Ese analisis es un documento fechado, no se reescribe. Pero la correccion medida el 2026-09-22 contra `GET /api/balance` (gratuito, antes y despues de cada llamada) confirmo que `ocr-id` siempre cobro **2 unidades por llamada**, no 1 -- el OpenAPI del proveedor declara `x-price: 1` mal solo para ese endpoint (el unico `x-type: special`; ver `docs/memory/api-apimarket-ocr-id.md` § Costo y limites reales). Cualquier total de esos mockups que incluya el OCR de documento hay que releerlo con el OCR al doble de lo que ahi se asumio.
- **Decision de producto sobre de donde sale cada dato segun el tipo de documento: YA IMPLEMENTADA (2026-09-22).** Mario decidio que la INE se lee de la **MRZ**, la licencia de la **OCR/VIZ** (casi ninguna licencia mexicana tiene MRZ) y el pasaporte de la **MRZ**; `extractDocument` (`backend/src/documentRules.js`) se reescribio el mismo dia en tres commits (`57aef30`/`201cba3`/`c0caed0`) para elegir cada campo por procedencia (`data.scores[llave].source`) en vez de "primer valor no vacio". Limite real que se respeto al implementarla: la MRZ de la INE **no** trae CURP, domicilio ni clave de elector (solo existen en la VIZ del frente), asi que en la INE la regla solo aplica a nombre/apellidos/fecha de nacimiento/sexo/vigencia/numero de documento — el resto sigue saliendo a fuerza del frente. Detalle completo, con evidencia de llamadas reales, en `docs/memory/api-apimarket-ocr-id.md` § Implementación de la selección por procedencia (y § Tercera llamada REAL para el origen de la decision). **Riesgo pendiente de verificar, sin medir**: la MRZ TD-1 de la INE trunca el nombre por longitud fija en documentos con nombres largos — ver item 8 de "Riesgos vivos" en § Llamada unica + pre-filtro, arriba en este mismo archivo.

## Pasaportes extranjeros (2026-09-23, rama `pasaportes-extranjeros`, sin commit ni merge)

- **Que hace**: el selector de pais ofrece Mexico + 198 paises (39 en la primera vuelta; la lista completa de Mario entro el mismo dia; Kosovo = `XK`) (`frontend/src/lib/countries.js`, espejo `backend/src/countries.js`). Fuera de Mexico la pantalla de identificacion solo ofrece pasaporte, no hay OCR (el frontend no llama `/ocr` y el backend lo rechaza con 400 `ocr_not_applicable`) ni cascada, y `/verify` decide con liveness + CompareFaces + IP. El expediente lleva `document: { docType: 'passport', country, ocrSkipped: true }` y ResultsView muestra "Pasaporte de <pais>: documento no validado".
- **Contrato**: `POST /api/sessions` acepta `{ country }` (ISO alfa-2; 400 `country_invalid` si no esta en la lista). Sin cuerpo = sesion de Mexico, igual que antes.
- **Verificado**: backend 485/485 (15 nuevos en `test/foreignPassport.test.js`), frontend 351/351 (incluye un test que renderiza las 40 banderas con `renderToStaticMarkup`), `vite build` OK. Tras la lista completa: backend 487/487, frontend 510/510 (el test de banderas renderiza las 199), bundle +48 KB gzip por importar la libreria de banderas entera. **Sin verificacion visual**: la extension de Chrome no estaba conectada, asi que el render de las 40 banderas y el flujo en telefono estan pendientes.
- **Riesgos aceptados**: (1) el pais lo declara el cliente — un pasaporte mexicano declarado como extranjero se salta OCR y RENAPO; (2) no se comprueba que el pasaporte sea del pais elegido; (3) `PassportCamera` esta calibrada con un solo pasaporte mexicano; otros paises comparten la geometria TD-3 pero no se han medido.
- **Decision abierta**: el analisis de IP (bloqueo por VPN/Tor/proxy) sigue aplicando a los pasaportes extranjeros; si el negocio quiere "solo liveness + face match" literal, quitarlo es una linea en `/verify`.

## Contexto de decisiones

- Plan ejecutado: `docs/superpowers/plans/2026-07-13-prototipo-kyc-rekognition.md` (11 tasks, subagent-driven, revision final "Ready to merge: Yes").
- Sin SDK/embed y sin S3 — decisiones explicitas de Mario para este prototipo. El OCR ya NO esta fuera de alcance: se integro el 2026-09-04 con API Market (rama `ocr-apimarket`), ver la seccion de arriba.
- No aplicar look "escaneado" B/N: retrato a color requerido por CompareFaces.
- Smoke test de permisos AWS y plan B de credenciales: nota `Sesion 2026-07-13 - Smoke test AWS aprobado.md`.

## JSON del cliente (2026-09-23, rama `json-cliente`, sin commit todavia)

Pedido de Mario: que el JSON del expediente le sirva al cliente, sin `debug`, sin `provider` y sin las confianzas del OCR.

- `backend/src/publicAnalysis.js` (nuevo, puro e idempotente) + `publicDocument`/`publicClassification` (`documentRules.js`). Se aplican al guardar `analysis.json`, a la respuesta de `/verify` (fresca y replay), a `GET /analysis` (limpia tambien los `analysis.json` viejos al servirlos, aunque el archivo en disco sigue sucio) y al `analysis` de `/ocr` por intentos agotados. Las respuestas de `/ocr/:kind` ya no llevan `debug` ni `classification.detectScore`.
- **Se va**: `debug`, `provider` (documento, IP, geo, cascada, `locations.sources.*`), `scores`/`confidence`/`fieldSources`/`imageQuality`/`detectScore` del OCR, `codigoValidacion` (documento, checks y fuentes), `faceMatch.raw`, `documentFace`, `liveness.sessionId`, `ipCheck.raw`, `govChecks.balance`/`summary.unitsSpent`/`units`/`latencyMs`.
- **Se queda**: veredicto + `thresholds`, `liveness.confidence`, `faceMatch.similarity`, datos del documento, cascada (`checks`/`sources`/`consistency`/`gates`/`rfcAudit`), `locations` y `files.frontDigest` (candado del replay de `/verify`: quitarlo hace que cada `/verify` vuelva a pagar).
- **Evidencia nueva en disco**: `rekognition.json` (`liveness` con `sessionId`, `compareFaces` y `detectFaces` crudos, sin `Bytes`). Antes esos dumps solo vivian en `analysis.json`; el resto de lo que se quito ya estaba en `ocr-*.json`/`gov-checks.json`.
- **Decisiones que Mario tiene que ratificar**: (1) `codigoValidacion` sale del expediente, pero se queda en las respuestas de ERROR de `/ocr` porque la UI lo muestra para que el usuario lo reporte a soporte en fallos de configuracion del proveedor; (2) `liveness.sessionId` se quito sin que se pidiera (sigue en `session.json` y en `rekognition.json`).
- Tests: backend 472, frontend 305, verdes.
