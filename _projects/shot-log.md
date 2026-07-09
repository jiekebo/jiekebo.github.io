---
title: Shot Log & Shot Detection
eyebrow: Machine Learning · Full-Stack
subtitle: >-
  An end-to-end target-scoring system — a computer-vision model that detects
  bullet impacts on paper targets, wired into a full-stack shooting-sports app
  that automatically scores shots.
summary: >-
  Computer-vision instance segmentation (homography + impact detection) for
  automatic target scoring, chained into a FastAPI + React Native shooting-log
  app deployed on DigitalOcean.
order: 1
cover: /assets/projects/shot-log/3_detection.jpg
tags: [PyTorch, ONNX, FastAPI, React Native, Terraform, DigitalOcean, Computer Vision]
stack: [PyTorch, Mask2Former, ONNX, FastAPI, React Native, Terraform, Caddy]
---

> Two repos, one product. **Shot Detection** is the ML brain that finds bullet
> holes and the target geometry in a photo. **Shot Log** is the product around
> it — clubs, events, series and scores. They are designed to be **chained
> together**: Shot Log sends a target-paper image to the Shot Detection model,
> gets back the detected impacts and target reference points, and turns that
> into an automatic score — no manual entry required.

## The idea

Live target scoring with nothing more than a cheap camera and a phone.

The goal is to clip a low-cost camera to a target rig, capture the paper, and
have computer vision do the rest: rectify the skewed photo into a flat,
metric view, detect every bullet impact, map each one to the correct scoring
ring, and log the full series into a cloud service that tracks progress over
time. The same pipeline can run on-device for instant feedback (with a shot
replay and audio cue) or in the cloud for batch scoring and analytics.

The work splits into three phases:

1. **Rectify** the target paper — obtain a homography and a flat, undistorted
   view of the paper.
2. **Detect impacts** — segment bullet holes with masks tight enough that
   scoring by edges is meaningful.
3. **Score** — detect the scoring rings and assign each shot a value, with
   edge cases (a shot landing exactly on a ring boundary) handled precisely.

## System architecture

The whole product spans an edge app, a cloud backend, an ML inference
service and the supporting cloud infrastructure. The diagram shows the full
request path — from a shooter capturing a target on their phone, through the
React Native app and Caddy edge, into the FastAPI backend (where scoring,
club management and witnessing live), out to the Shot Detection ONNX model,
and back as an automatic score persisted in PostgreSQL — alongside the
delivery pipeline and the external services (object storage, email, the
postal/police licensing flow).

<div class="diagram">
<div class="mermaid">
flowchart TD
  subgraph client["Client"]
    PHONE["Shooter phone<br/>Expo / React Native<br/>expo-camera + OpenCV warp"]
    CLUB["Club admin<br/>web dashboard"]
  end

  subgraph edge["Edge"]
    CADDY["Caddy<br/>auto-TLS · HTTP/3<br/>reverse proxy"]
  end

  subgraph be["FastAPI backend · Droplet"]
    AUTH["Auth · OAuth2 JWT<br/>roles: admin > moderator > member"]
    DOMAIN["Domain layer<br/>clubs · events · series · shots<br/>weapons · witnesses · awards · stats"]
    SCORE["Scoring flow<br/>homography → ring projection"]
    RATE["SlowAPI rate limiting"]
  end

  subgraph ml["Shot Detection · inference"]
    ONNX["ONNX instance-seg model<br/>SAHI sliced inference<br/>shot · center_ring · target_paper"]
    WEB["onnxruntime-web (WASM)<br/>runs in the phone browser"]
  end

  PG[("PostgreSQL 16<br/>managed · private VPC")]
  S3[("Spaces / S3<br/>target images")]
  MAIL["Resend / AWS SES<br/>transactional email"]

  PHONE -->|"capture & rectify"| CADDY
  CLUB --> CADDY
  CADDY -->|"/api/* · /uploads/*"| be
  CADDY -->|"static SPA"| PHONE
  CADDY -->|"static SPA"| CLUB

  RATE --> be
  be --> AUTH
  AUTH --> DOMAIN
  DOMAIN -->|"POST /score-image<br/>image"| ONNX
  ONNX -->|"JSON: masks + boxes<br/>+ reference points"| DOMAIN
  DOMAIN --> SCORE
  SCORE --> PG
  DOMAIN --> PG
  DOMAIN -->|"presigned URLs"| S3
  DOMAIN -->|"invite / witness / bounce"| MAIL

  PHONE -.->|"optional: on-device<br/>inference (no round-trip)"| WEB

  PHONE -->|"witness gold-level series<br/>(activity proof)"| CLUB
  CLUB -.->|"member activity record<br/>→ police license recommendation"| POLICE["Police licensing authority"]

  subgraph cicd["CI/CD"]
    GH["GitHub · push to master"]
    GHA["GitHub Actions"]
    GHCR["ghcr.io<br/>backend + frontend images"]
  end
  GH --> GHA
  GHA --> GHCR
  GHCR -->|"SSH · docker compose pull"| be
</div>
</div>

## How the two components fit together

<div class="diagram">
<div class="mermaid">
flowchart LR
  CAM(["Phone camera<br/>target-paper photo"])

  subgraph product["Shot Log — product"]
    direction TB
    RN["React Native app<br/>Expo · Tamagui<br/>expo-camera + OpenCV warp"]
    API["FastAPI backend<br/>clubs · events · series · shots · stats"]
    RN -->|"capture & rectify"| API
  end

  subgraph model["Shot Detection — ML model"]
    ONNX["ONNX instance-seg model<br/>Mask2Former / RF-DETR<br/>SAHI sliced inference"]
  end

  DB[("PostgreSQL<br/>series & statistics")]

  CAM --> RN
  API -->|"POST /score-image<br/>image"| ONNX
  ONNX -->|"JSON: masks + boxes + scores<br/>shot · center_ring · target_paper"| API
  API -->|"homography → ring projection<br/>automatic score"| DB
</div>
</div>

Shot Log's backend calls the Shot Detection inference service with the
captured image. The model returns three classes of instance masks:
**target paper**, **center ring** and **shot** (bullet impacts). The backend
uses the target-paper and center-ring detections to compute the homography
that maps the photo onto a canonical target layout, then projects each shot
impact onto the scoring rings to produce a value per shot. That value flows
straight into the series and statistics — fully automatic scoring.

## The detection pipeline, visually

The images below show the progression through the pipeline — the same target
going from a raw, perspective-skewed photo to a flat homography-corrected
view, and finally to the model's segmentation overlay with detected shot
impacts, the target paper and the center circle.

<div class="pipeline">
  <figure>
    <img src="{{ '/assets/projects/shot-log/1_raw_image.jpg' | relative_url }}" alt="Raw captured photo of a shot-up target paper" loading="lazy" />
    <figcaption>1 · Raw image from the camera</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/projects/shot-log/2_homography.jpg' | relative_url }}" alt="Target paper after homography rectification" loading="lazy" />
    <figcaption>2 · Homography — flat, rectified paper</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/projects/shot-log/3_detection.jpg' | relative_url }}" alt="Final instance segmentation overlay with detected shots, target paper and center ring" loading="lazy" />
    <figcaption>3 · Detection — impacts, paper &amp; center ring</figcaption>
  </figure>
</div>

## The dataset

This is a **finely curated dataset I built myself**. Every target paper was
labelled manually using [X-AnyLabeling](https://github.com/CVHub520/X-AnyLabeling),
a tool that blends auto-suggested labels with hand correction — exactly what
you want when the mask edges determine the score.

- **Sources:** my own shot targets plus targets collected from other shooters,
  covering both .22 LR and air-pistol disciplines.
- **Annotations:** COCO instance-segmentation polygons for three classes —
  `shot` (bullet impact), `center_ring` and `target_paper`.
- **Why it matters:** scoring is all about what the mask edges touch, so the
  labels are drawn tight to the actual hole boundaries rather than loose
  blobs. A loose mask would silently corrupt every edge-of-ring decision.
- **Reference targets:** official ISSF 10 m air-pistol and 25 m target SVGs
  are kept alongside the data so the canonical scoring-ring layout is
  available for the scoring phase.

## Architecture by component

### 1. Inference pipeline (Shot Detection)

The ML side is built for **accuracy first, deployment second**: train a
strong teacher model, then distil and quantize it down to something that
runs fast enough on-device or in the browser.

<div class="diagram">
<div class="mermaid">
flowchart LR
  A["raw image"] --> B["SAHI slicing<br/>384×384 @ 0.4 overlap"]
  B --> C["per-tile inference<br/>Mask2Former / RF-DETR"]
  C --> D["reassemble confidence maps<br/>into full image"]
  D --> E["CombineStrategy<br/>stitch overlapping tiles"]
  E --> F["post-process<br/>instance segmentation"]
  F --> G["masks + boxes + scores<br/>shot · center_ring · target_paper"]
</div>
</div>

**Training.** Models are trained as `LightningModule`s on tiled COCO
instance-segmentation data with Albumentations augmentation, AdamW + cosine
warmup LR, mixed precision and gradient accumulation. Three model families
were explored and compared in MLflow under a single
`shot-detection-model-comparison` experiment:

- **Mask2Former** (`facebook/mask2former-swin-tiny-coco-instance`) — the
  primary instance-segmentation model.
- **RF-DETR** (`Roboflow/rf-detr-seg-medium`) — currently the checkpoint in
  use.
- **YOLO26** and **MobileViT** — earlier experiments, being phased out.

**Distillation & quantization.** A student model is distilled from the
teacher using a dense pixel-level KL-divergence loss between the teacher's
and student's per-pixel class distributions (computed via
`einsum("bqc,bqhw->bchw", class_probs, mask_probs)`), combined with the
standard Hungarian-matched task loss. The student is then exported to ONNX
and dynamically quantized to INT8 — roughly halving the model size while
keeping deployment friendly.

**Export.** Models are exported to ONNX via `optimum` with a `torch.onnx`
fallback, including custom symbolics for the antialias upsample ops the
legacy tracer can't handle. The exported ONNX runs through `onnxruntime`
(CUDA with CPU fallback) and is also served **in the browser** with
`onnxruntime-web` (WASM) — the full sliced-inference + post-processing path
is re-implemented in JavaScript so a phone can run inference without a
server round-trip.

**Homography / paper rectification.** A separate OpenCV.js stage detects the
target-paper quadrilateral (Canny → contours → `approxPolyDP` →
`getPerspectiveTransform` → `warpPerspective`) and rectifies the photo to a
flat, metric view before the impact model runs — which is exactly what
step 2 of the visual pipeline above shows.

### 2. Backend (Shot Log)

A FastAPI service that owns the domain model: clubs, events, series,
individual shots, weapons, witnesses, awards and statistics.

- **Auth:** OAuth2 password flow with JWT (HS256), bcrypt password hashing
  and a role hierarchy (`admin` > `moderator` > `member`) enforced by a
  `require_role(*roles)` dependency.
- **Persistence:** SQLAlchemy 2.0 ORM with Alembic migrations targeting
  PostgreSQL 16 (managed in the cloud; SQLite for local dev only).
- **Storage:** a storage abstraction that serves local files in dev or
  S3-compatible object storage (presigned URLs) in production.
- **Email:** a pluggable transport layer (`email_providers/`) that switches
  between AWS SES, Resend and a dev logger via a single `EMAIL_PROVIDER`
  setting — including SNS-based bounce/complaint suppression.
- **Rate limiting:** SlowAPI middleware on auth and upload endpoints.
- **Shot-detection integration:** the backend exposes the capture-and-score
  flow — it accepts the target image, forwards it to the Shot Detection
  inference service, and stores the resulting impacts and computed scores
  against the series. The image upload endpoint is rate-limited and stores
  both the original and processed image paths.

### 3. Frontend (Shot Log)

An Expo / React Native app targeting iOS, Android and Web from one
TypeScript codebase with Expo Router file-based routing.

- **UI & state:** Tamagui component system, React Query for server cache,
  Zustand for local/auth state, React Hook Form + Zod for forms.
- **Camera / scanner:** `expo-camera` with a live quadrilateral overlay; an
  on-device OpenCV pipeline (`react-native-fast-opencv` on native,
  `opencv.js` WASM on web) does the paper detection and perspective warp in
  real time so the user sees the rectified target before capturing.
- **Internationalisation:** a custom i18n store with 12 locales.
- **Native build:** a custom Expo dev build is required (not Expo Go) because
  of the native OpenCV module.

The app serves three distinct user journeys: individual shooters scoring
targets and tracking progress, club admins managing member activity, and a
club-witnessing workflow that feeds the police firearm-licensing process.

<div class="pipeline pipeline--screens">
  <figure>
    <img src="{{ '/assets/projects/shot-log/series_details.png' | relative_url }}" alt="Shot Log series detail screen showing an uploaded target and the automatic score returned by the system" loading="lazy" />
    <figcaption>Series detail — upload a target photo and get the automatic score back, logged against the series.</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/projects/shot-log/statistics.png' | relative_url }}" alt="Shot Log statistics screen showing a shooter's progress over time" loading="lazy" />
    <figcaption>Statistics — a shooter tracks their progress over time across events and series.</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/projects/shot-log/club_activity.png' | relative_url }}" alt="Shot Log club activity screen showing member activity overview for club management" loading="lazy" />
    <figcaption>Club activity — admins overview member activity and witness gold-level series, feeding the police firearm-licence recommendation.</figcaption>
  </figure>
</div>

### 4. Cloud deployment

Two deployment targets are maintained; the active one is **DigitalOcean**,
with an alternative AWS ECS Fargate stack.

**DigitalOcean (active):**

```
GitHub (push to master) ──► GitHub Actions
                            ├── build & push backend image  ──► ghcr.io
                            ├── build & push frontend image ──► ghcr.io
                            └── SSH into Droplet: pull + docker compose up -d

Internet → Droplet
            ├── Caddy (80/443, auto Let's Encrypt TLS, HTTP/3)
            │     ├── /api/*, /uploads/* → backend:8000
            │     └── /* → frontend:3000 (static SPA)
            ├── backend container (port 8000, image from ghcr.io)
            └── frontend container (port 3000, image from ghcr.io)

Backend → Managed PostgreSQL 16 (private VPC)
Backend → Spaces (S3-compatible file uploads)
Backend → Resend (transactional email)
```

- **Infrastructure as code:** Terraform provisions the Droplet (Ubuntu 24.04,
  cloud-init installs Docker), a managed PostgreSQL 16 cluster, a Spaces
  bucket, firewall and VPC. The Droplet never needs source access — it only
  pulls pre-built images.
- **Reverse proxy / TLS:** Caddy terminates TLS with automatic Let's Encrypt
  certificates and routes `/api/*` and `/uploads/*` to the backend and
  everything else to the static frontend.
- **CI/CD:** GitHub Actions builds both Docker images, pushes them to the
  GitHub Container Registry, then SCPs the compose file and SSHes in to
  `docker compose pull && up -d`. Every push to `master` ships to
  production.

**AWS (alternative):** ECS Fargate (nginx + backend sidecars), an ALB,
RDS PostgreSQL, S3, SES/SNS for email deliverability, and secrets injected
from SSM Parameter Store — all Terraform-managed.

## Technologies used

<div class="tech-grid">
  <div class="tech">
    <h4>Inference pipeline</h4>
    <ul>
      <li>PyTorch + PyTorch Lightning</li>
      <li>HuggingFace Transformers (Mask2Former, RF-DETR, MobileViT)</li>
      <li>Ultralytics (YOLO26)</li>
      <li>ONNX Runtime (CUDA / WASM)</li>
      <li>optimum, torchao (QAT/PTQ int4-int8)</li>
      <li>OpenCV &amp; OpenCV.js (homography)</li>
      <li>Albumentations, pycocotools</li>
      <li>MLflow, TensorBoard</li>
      <li>X-AnyLabeling (annotation)</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Backend</h4>
    <ul>
      <li>FastAPI + Uvicorn</li>
      <li>SQLAlchemy 2.0 + Alembic</li>
      <li>PostgreSQL 16</li>
      <li>Pydantic v2 / pydantic-settings</li>
      <li>python-jose (JWT), bcrypt</li>
      <li>boto3 (S3/SES/SNS), Resend SDK</li>
      <li>SlowAPI (rate limiting)</li>
      <li>uv (package management)</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Frontend</h4>
    <ul>
      <li>Expo SDK 55 / React Native 0.83</li>
      <li>React 19 + TypeScript 5.9</li>
      <li>Expo Router (file-based)</li>
      <li>Tamagui UI</li>
      <li>TanStack React Query, Zustand</li>
      <li>React Hook Form + Zod</li>
      <li>react-native-fast-opencv / opencv.js</li>
      <li>expo-camera, expo-secure-store</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Cloud deployment</h4>
    <ul>
      <li>Terraform (DigitalOcean &amp; AWS)</li>
      <li>Docker Compose</li>
      <li>Caddy (reverse proxy + auto-TLS)</li>
      <li>GitHub Actions + GHCR</li>
      <li>DigitalOcean Droplet, Managed PG, Spaces</li>
      <li>AWS ECS Fargate, RDS, S3, ALB, SES/SNS</li>
      <li>SSM Parameter Store (secrets)</li>
    </ul>
  </div>
</div>

<div class="callout">
  <p>
    <strong>Status:</strong> the Shot Detection model trains and infers
    end-to-end (ONNX export, quantization and a browser webapp are all
    working), and Shot Log is a deployed, multi-tenant product. The final
    API wiring that returns automatic scores from the model into Shot Log's
    series is the active integration work — the architecture above describes
    how the two are designed to chain together.
  </p>
</div>
