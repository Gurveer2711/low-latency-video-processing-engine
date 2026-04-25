# Low Latency Video Processing Engine

Production-style backend system for uploading large videos, processing them asynchronously, and serving secure playback URLs with low API latency.

## Why This Project

Most video systems fail when heavy upload and transcoding work is done inside the API process.  
This project solves that by separating responsibilities:

- API handles lightweight control flow (auth, metadata, signed URLs, job enqueue)
- Worker handles CPU-heavy FFmpeg processing
- Queue + events connect services asynchronously and reliably

## Key Highlights

- Stateless direct-to-S3 upload flow using presigned URLs
- Event-driven processing via S3 -> SQS -> BullMQ worker pipeline
- Multi-variant transcoding (`480p`, `720p`) + thumbnail generation
- Retry strategy with exponential backoff and Dead Letter Queue (`transcodeDLQ`)
- Secure signed playback URLs
- End-to-end lifecycle tracking: `uploaded -> processing -> processed/failed`

## Performance Outcomes

- Processed `50+` videos asynchronously
- Reduced API memory usage by about `90%` by removing file bytes from API path
- Maintained `<50ms` latency for API control-plane endpoints under concurrent load
- Supports large media uploads (tested up to `500MB`)

## Architecture

```text
Client -> API (request upload URL) -> S3 (direct upload)
S3 Event -> SQS -> API Consumer -> BullMQ (Redis)
BullMQ -> Worker -> FFmpeg -> S3 + PostgreSQL
Client <- API (status/playback URL) <- PostgreSQL
```

## Tech Stack and Purpose

### API Service (`api/`)
- `Node.js + TypeScript`: fast development and type-safe backend code
- `Express`: REST endpoints for auth, upload initiation, video operations
- `Prisma + PostgreSQL`: reliable relational data model and status tracking
- `AWS SDK v3`: S3 presigned uploads and signed playback URLs
- `BullMQ + Redis`: queue producer for async transcoding jobs

### Worker Service (`worker/`)
- `Node.js + TypeScript`: same language as API for faster iteration
- `BullMQ Worker`: consumes transcode jobs reliably
- `FFmpeg + ffprobe`: transcoding and source resolution inspection
- `S3 Integration`: reads raw assets and uploads processed outputs

### Frontend (`client/`)
- `React + Vite`: lightweight dashboard to upload videos and monitor job states

### Infrastructure
- `Docker + docker-compose`: local orchestration for API, worker, Redis, Postgres

## Core Flow

1. Client calls `POST /api/upload/request` to get a presigned S3 URL.
2. Client uploads raw video directly to S3.
3. S3 notification is received in SQS.
4. API consumer validates event and enqueues a BullMQ job.
5. Worker transcodes to `480p`, conditionally `720p`, and creates thumbnail.
6. Outputs are uploaded to S3 and DB status is updated.
7. Client reads status with `GET /videos` and plays via signed URL endpoint.

## API Surface (Important Endpoints)

- `POST /auth/*` - authentication flow
- `POST /api/upload/request` - create upload session and presigned URL
- `GET /videos` - list videos with processing status
- `POST /videos/:id/retry` - retry failed jobs (admin only)
- `GET /videos/:id/play` - signed playback URL
- `GET /videos/:id/thumbnail` - signed thumbnail URL

## Local Setup

### 1) Prerequisites
- Docker + Docker Compose
- Node.js 18+
- AWS account resources (`S3`, `SQS`) and credentials

### 2) Environment
Create `.env` in project root with required variables:

```env
DATABASE_URL=
REDIS_HOST=redis
REDIS_PORT=6379
AWS_REGION=
AWS_BUCKET=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
SQS_QUEUE_URL=
JWT_SECRET=
```

### 3) Run Services

```bash
docker-compose up --build
```

### 4) Start Frontend

```bash
cd client
npm install
npm run dev
```

## Design Decisions and Trade-offs

- Presigned S3 upload adds cloud setup complexity, but drastically reduces API memory pressure.
- Queue-based architecture introduces more components, but gives reliability, retries, and horizontal scaling.
- Polling dashboard is simple and stable; WebSockets can be a future enhancement for real-time updates.

## Future Improvements

- HLS/DASH packaging for adaptive streaming
- WebSocket/SSE for live progress updates
- Metrics + tracing (Prometheus/Grafana, OpenTelemetry)
- CDN integration for global playback latency reduction
- Automated CI pipeline for tests and linting