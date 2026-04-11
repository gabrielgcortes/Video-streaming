# Flixy — Video Streaming Platform

> A full-stack video streaming platform inspired by YouTube, built with Django 5.2. Supports HLS adaptive streaming, async video processing, cloud storage, JWT authentication, and a role-based admin panel.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Django 5.2 + Gunicorn |
| Database | PostgreSQL |
| Task Queue | Celery 5.5 + Redis |
| Cloud Storage | AWS S3 (boto3) |
| Video Processing | FFmpeg (HLS segmentation) |
| Authentication | JWT (PyJWT) via HTTP-only cookies |
| Reverse Proxy | Nginx |
| Containerization | Docker (Redis service) |

---

## Key Features

- **HLS Video Streaming** — Uploaded `.mp4` files are asynchronously converted to HLS (`.m3u8` + `.ts` segments) via FFmpeg and stored on AWS S3. The frontend fetches signed streaming URLs, enabling adaptive playback.
- **JWT Authentication** — Stateless auth using HTTP-only cookies. Custom decorators (`verificar_token`, `intentar_verificar_token`) protect routes and distinguish authenticated vs guest users.
- **AWS S3 Integration** — A dedicated `S3Manager` service class handles all cloud operations: presigned upload/download URLs, folder deletion, M3U8 rewriting with signed segment URLs, and profile photo uploads.
- **Async Task Pipeline** — Celery workers consume a Redis queue to process video conversions in the background without blocking HTTP responses.
- **Role-Based Access Control** — Two roles (`admin`, `user`). Admins can approve videos, change user roles, and delete accounts. Regular users can upload, manage their own content, follow creators, and interact with videos.
- **Video Moderation** — Videos require admin approval (`revisado`) before becoming publicly visible. Private videos are protected by a unique access token.
- **Social Features** — Like/dislike system, threaded comments with replies, channel following, and watch history per user.
- **Admin Dashboard** — Dedicated `paneladmin` app with paginated user management, video approval, and role assignment.
- **Password Recovery** — Token-based recovery flow via email (SMTP).
- **Image Optimization** — Thumbnails are processed server-side with Pillow (JPEG compression, resize) before upload.

---

## Architecture Overview

```
┌─────────────┐    HTTP     ┌────────────────────────────────────┐
│   Browser   │ ─────────► │  Nginx  (reverse proxy + static)   │
└─────────────┘             └──────────────┬─────────────────────┘
                                           │
                                           ▼
                             ┌─────────────────────────┐
                             │  Gunicorn (WSGI server)  │
                             └──────────────┬──────────┘
                                            │
                                            ▼
                          ┌─────────────────────────────────┐
                          │           Django 5.2            │
                          │  ┌────────┐  ┌──────────────┐  │
                          │  │usuarios│  │    videos    │  │
                          │  └────────┘  └──────────────┘  │
                          │  ┌──────────┐ ┌────────────┐   │
                          │  │comentarios│ │ paneladmin │   │
                          │  └──────────┘ └────────────┘   │
                          │          ┌──────────┐           │
                          │          │ services │           │
                          │          │(S3Manager│           │
                          │          └────┬─────┘           │
                          └───────────────┼─────────────────┘
                                          │
               ┌──────────────────────────┼───────────────────┐
               │                          │                   │
               ▼                          ▼                   ▼
        ┌──────────┐              ┌──────────────┐    ┌──────────────┐
        │PostgreSQL│              │   AWS S3     │    │ Redis+Celery │
        └──────────┘              │(videos, imgs)│    │(HLS convert) │
                                  └──────────────┘    └──────────────┘
```

---

## Project Structure

```
flixy/
├── mysite/              # Django project config (settings, URLs, Celery)
├── videos/              # Core app: upload, streaming, likes, history
│   ├── models.py        # Videos, Historial, LikesDislikes, DB views
│   ├── views.py         # REST-style class-based & function views
│   ├── tasks.py         # Celery task: MP4 → HLS conversion
│   ├── decoradores.py   # JWT auth decorators
│   └── utils.py         # FFmpeg wrapper, image optimizer
├── usuarios/            # Auth app: registration, login, channels, followers
│   ├── models.py        # Usuarios, Canales, Roles, Seguidores
│   ├── views.py         # Login, register, logout, follow/unfollow
│   └── utils.py         # Token generation, password hashing
├── comentarios/         # Threaded comments
├── paneladmin/          # Admin dashboard (video approval, user management)
├── services/
│   └── s3_storage.py    # S3Manager: all AWS S3 operations
├── docker/              # Redis docker-compose + activation scripts
├── setup_flixy.sh       # Interactive Nginx + Gunicorn deployment script
└── requirements.txt
```

---

## API Endpoints

### Videos
| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/videos/` | List public videos (supports `?titulo=` search) | Optional |
| `POST` | `/videos/` | Upload a new video | Required |
| `GET` | `/videos/<id>` | Get video details | Public |
| `PUT` | `/videos/<id>` | Update video metadata | Owner / Admin |
| `DELETE` | `/videos/<id>` | Delete a video | Owner / Admin |
| `GET` | `/videos/ver/<id>/` | Render video page + history tracking | Optional |
| `GET` | `/videos/stream/<id>/` | Serve signed M3U8 manifest | Public |
| `GET` | `/videos/estado/<id>/` | Poll HLS conversion status | Required |
| `GET` | `/urls-s3/<id>` | Get presigned S3 upload URLs | Required |
| `POST` | `/videos/subido/<id>/` | Confirm S3 upload complete | Required |
| `GET/POST` | `/videos/<id>/like/` | Get likes / toggle like | Optional / Required |
| `GET/POST` | `/videos/<id>/dislike/` | Get dislikes / toggle dislike | Optional / Required |

### Users & Auth
| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET/POST` | `/login/` | Login (sets JWT cookie) | — |
| `GET/POST` | `/registro/` | Register user + create channel | — |
| `GET` | `/logout/` | Invalidate session | — |
| `GET/POST` | `/recuperar/` | Request password reset email | — |
| `POST` | `/recuperar/<token>/` | Reset password with token | — |
| `GET/POST/DELETE` | `/usuarios/seguir-autor-por-video/<id>/` | Follow / check / unfollow channel | Required |

### Admin Panel
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/paneladmin/` | Admin dashboard view |
| `POST` | `/paneladmin/aprobar/<id>/` | Approve a video |
| `POST` | `/paneladmin/cambiar-rol/<id>/` | Change user role |
| `POST` | `/paneladmin/eliminar-usuario/<id>/` | Delete user + channel + videos |
| `GET` | `/paneladmin/usuarios/` | Paginated user list |

---

## Video Upload Pipeline

```
1. User submits form (MP4 + thumbnail + metadata)
         │
2. Django saves MP4 to MEDIA_ROOT/tmp/ and creates a Videos record
         │
3. Celery task is queued via Redis (convertir_video_a_hls)
         │
4. FFmpeg converts MP4 → HLS segments (index.m3u8 + *.ts) in MEDIA_ROOT/stream/<id>/
         │
5. Videos.conversion_completa is set to True
         │
6. Frontend polls /videos/estado/<id>/, detects completion
         │
7. Frontend fetches presigned S3 PUT URLs from /urls-s3/<id>/
         │
8. Frontend uploads each segment directly to S3
         │
9. Frontend calls /videos/subido/<id>/ → Django sets estado=True, cleans up local files
```

---

## Local Development Setup

### Prerequisites

- Python 3.11+
- PostgreSQL
- Redis (via Docker)
- FFmpeg installed and on `PATH`

### Installation

```bash
git clone https://github.com/<your-username>/flixy.git
cd flixy
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Environment Variables

Create a `.env` file at the project root:

```env
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
AWS_STORAGE_BUCKET_NAME=your_bucket
AWS_S3_REGION_NAME=us-east-2
```

> Database credentials and the Django `SECRET_KEY` are currently configured in `settings.py`. For production, move them to `.env` as well.

### Start Redis + Celery Worker

```bash
# Linux / macOS
./docker/Scripts/activate_env.sh

# Windows (PowerShell)
.\docker\Scripts\activate_env.ps1
```

This starts the Redis container via Docker Compose and launches a Celery worker.

### Run the Development Server

```bash
python manage.py migrate
python manage.py runserver
```

---

## Production Deployment

An interactive shell script automates Nginx + Gunicorn configuration:

```bash
chmod +x setup_flixy.sh
./setup_flixy.sh
```

The script will prompt for your domain, Gunicorn port, and project paths, then:
1. Run `collectstatic`
2. Generate and enable an Nginx site config
3. Create and enable a `gunicorn.service` systemd unit
4. Restart both services

---

## Data Models (Simplified)

```
Usuarios ──< Canales ──< Videos ──< VideosEtiquetas >── Etiquetas
    │                       │
    │                  LikesDislikesVideos
    │                  Historial
    │                  Comentarios (self-referential for replies)
    │
    └──< Seguidores (self-referential follow system)
```

Database views (`managed = False`) are mapped as Django read-only models:
- `vwdetalle_video` — Full video detail (joined across tables)
- `vw_videos_con_etiquetas` — Videos with their tag categories
- `vista_canal_de_video` — Channel + profile photo per video

---

## License

This project was built as a personal portfolio project. Feel free to explore the code.
