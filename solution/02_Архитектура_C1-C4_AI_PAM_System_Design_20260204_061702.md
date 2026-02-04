# System Design — AI аналитика PAM-сессий — Архитектура (C1–C4)

> Этот файл: архитектура, диаграммы C1–C4, потоки данных, хранилища, наблюдаемость.

---

## 1. Архитектурные принципы
1) **Sidecar (out-of-band)**: AI не участвует в контуре “разрешить доступ”.
2) **Потоковая обработка**: SSH — near-real-time (tail по seq), RDP — micro-batch (скриншоты).
3) **Идемпотентность**: каждый алерт/отчёт имеет dedup_key.
4) **Деградация по нагрузке**: при перегрузе отключаем тяжёлые модели (Gemma), оставляем YOLO/правила.

---

## 2. C1 — System Context
```mermaid
flowchart LR
  U[Привилегированный пользователь] -->|RDP/SSH/VNC| PAM[PAM]
  PAM --> TR[Целевые ресурсы]

  subgraph PAM_STORE[Хранение PAM]
    SQL["(SQL БД SSH JSON segments)"]
    FS["(Локальный диск RDP скриншоты+видео)"]
  end

  PAM --> SQL
  PAM --> FS

  subgraph AI[AI Analytics Sidecar]
    COL["Сборщики артефактов (SSH DB Tailer + RDP FS Watcher)"]
    BUS["(Очередь/шина событий)"]
    PRE[Preprocess/Feature Extract]
    INF["Model Serving (Qwen/T-Pro + YOLO/Gemma)"]
    DEC[Risk+UEBA]
    RES["(Results DB + Vector Index)"]
    OBJ["(Object Store (thumbnails/ROI))"]
  end

  SQL --> COL
  FS --> COL
  COL --> BUS --> PRE --> INF --> DEC --> RES
  PRE --> OBJ

  DEC --> SIEM[SIEM/SOC]
  DEC -. человек .-> AUD["Аудитор/Админ PAM (ручное действие: прервать сессию)"]
```

---

## 3. C2 — Containers
```mermaid
flowchart TB
  subgraph PAM["PAM (существует)"]
    PAM_UI[Web UI]
    PAM_Core[PAM Core]
    PAM_DB[(SQL DB)]
    PAM_FS[(Local FS artifacts)]
  end

  subgraph AI_SYS["AI Analytics (добавляем)"]
    SSH_Tailer[SSH DB Tailer]
    RDP_Watcher[RDP FS Watcher]
    Orchestrator[Session Orchestrator]
    Queue["(Kafka/Rabbit/Redis Streams)"]
    Preproc["Preprocessor (ANSI clean, keyframes, ROI)"]
    ModelRouter[Model Router]
    ModelServe[Model Serving vLLM/Triton/ONNX]
    UEBA[UEBA Service]
    Results["(Postgres Results DB)"]
    Vector["(Vector Index, optional)"]
    Obj["(S3-compatible Object Store)"]
    Alert["Alerting Gateway (Syslog/CEF/Email/Telegram)"]
    AuditUI[AI Audit UI / API]
  end

  PAM_DB --> SSH_Tailer
  PAM_FS --> RDP_Watcher

  SSH_Tailer --> Queue
  RDP_Watcher --> Queue
  Queue --> Orchestrator --> Preproc --> ModelRouter --> ModelServe --> UEBA --> Results
  UEBA --> Alert
  Results --> AuditUI
  Preproc --> Obj
  ModelServe --> Vector
```

---

## 4. C3 — Components (внутри AI Analytics)
```mermaid
flowchart LR
  subgraph Ingest[Ingest]
    A1["SSH Segment Reader (tail по session_id+seq)"]
    A2["Stream Reassembler (stdin/stdout)"]
    A3["Command Extractor (backspace, multiline)"]
    B1["RDP Watcher (новый скрин/видео)"]
    B2["Frame Dedup (delta/hash)"]
    B3[ROI Extractor]
  end

  subgraph Proc[Processing]
    C1["Sanitizer (ANSI strip, normalize)"]
    C2[Feature Builder]
    C3[Sliding Summary Cache]
  end

  subgraph Models[Models]
    M1["Qwen Coder (intent/обфускация)"]
    M2["T-Pro / OSS LLM (summary/reason)"]
    M3["YOLO 11 (detector)"]
    M4["Gemma (multimodal/OCR assist)"]
  end

  subgraph Decision[Decision & Output]
    D1[Risk Scoring]
    D2["Rule Layer (thresholds, cooldown)"]
    D3[UEBA Baselines]
    D4[Alert Formatter]
    D5[Report Generator]
  end

  A1-->A2-->A3-->C1-->C2-->M1-->D1
  A3-->C3-->M2-->D5
  B1-->B2-->B3-->C2-->M3-->D1
  M3--подозр. кадры-->M4-->D1
  D1-->D2-->D4
  D1-->D3-->D4
  D5-->D4
```

---

## 5. C4 — Runtime sequences

### 5.1 SSH near-real-time (tail по seq)
```mermaid
sequenceDiagram
  participant DB as SQL БД PAM
  participant Tailer as SSH DB Tailer
  participant Q as Очередь
  participant P as Preproc
  participant L as LLM (Qwen/T-Pro)
  participant R as Risk/UEBA
  participant A as Alerting/SIEM

  Tailer->>DB: SELECT segments WHERE session_id=? AND seq>last_seq ORDER BY seq
  DB-->>Tailer: segments batch
  Tailer->>Q: publish(batch)
  Q-->>P: consume(batch)
  P->>P: reassemble stdin/stdout + extract command events
  P->>L: infer(intent/risk) (gated)
  L-->>P: labels + confidences + facts
  P->>R: update state + score
  R-->>A: alert if threshold exceeded (evidence: seq range)
```

### 5.2 RDP micro-batch (скриншоты)
```mermaid
sequenceDiagram
  participant FS as FS PAM
  participant W as RDP Watcher
  participant Q as Очередь
  participant P as Preproc
  participant Y as YOLO 11
  participant G as Gemma
  participant R as Risk/UEBA
  participant A as Alerting/SIEM

  FS-->>W: new screenshot file
  W->>Q: publish(ref)
  Q-->>P: consume(ref)
  P->>P: dedup + resize + ROI
  P->>Y: detect
  Y-->>P: detections
  alt suspicious
    P->>G: multimodal/OCR assist
    G-->>P: text/semantics
  end
  P->>R: update score + evidence(frame_id)
  R-->>A: alert if needed
```

---

## 6. Хранилища и модель данных (Results)

### 6.1 Postgres (предложение)
- `ai_session_state(session_id, protocol, user_id, resource_id, start_ts, end_ts, status, risk_current, risk_max, summary_final, ...)`
- `ai_evidence(evidence_id, session_id, ts, type, severity, payload_json, pointer_obj, ...)`
- `ai_alerts(alert_id, session_id, ts, severity, dedup_key, delivery_status, ...)`
- `ueba_profile(user_id, baseline_json, updated_ts, ...)`

### 6.2 Object Store (MinIO/S3)
- `/evidence/{session_id}/{frame_id}.jpg` (thumbnail/ROI)
- `/evidence/{session_id}/{seq_from}_{seq_to}.json` (фрагменты команд, если нужно)

---

## 7. Наблюдаемость (Observability)
- Метрики:
  - lag по `seq` и lag по кадрам
  - P95/P99 latency end-to-end
  - GPU utilization, очередь, ошибки inference
  - FP rate (по фидбеку аудиторов)
- Логи:
  - structured JSON с `session_id`, `evidence_id`
- Tracing:
  - OpenTelemetry: ingest → preprocess → inference → decision → alert

---

## 8. Деградация и backpressure
- Если очередь растёт:
  - снижать частоту RDP обработки (sampling)
  - отключать Gemma (оставить YOLO)
  - увеличивать batch по SSH
- Если GPU недоступен:
  - fallback на правила/эвристику (SSH)
  - пропуск deep анализа (RDP), но сохранять доказательства для постфактум

