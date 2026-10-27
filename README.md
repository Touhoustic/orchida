# Orchida

A self-hosted cloud storage and secure messaging web app built with Flask. Users upload and organize files in folders, share them through groups, and exchange end-to-end encrypted messages. Admins manage users, groups and storage quotas.

## Features

**Files and folders**
- Upload multiple files (max 100 MB per request), per-user storage quota (default 5 GB)
- Nested folders with optional size limit (admin only)
- Folder sharing through groups; owner and admin always have access
- In-browser preview for text/code, PDF and images; download for other types
- File integrity check (HMAC-SHA256 computed at upload, verified on demand via `/api/files/<id>/verify`)
- Uploaded files are checked by SHA-256 hash against the VirusTotal API and rejected if flagged

**Messaging**
- One-to-one messages encrypted per message:
  - AES-256-CBC for content (random key + IV per message)
  - AES and HMAC keys wrapped with the receiver's RSA-2048 public key (OAEP, SHA-256)
  - RSA-PSS (SHA-256) signature by the sender, HMAC-SHA256 for integrity
- Unread counters, delete message (removes it for both users)

**Administration**
- Create / delete users, reset passwords, set storage limits
- Create groups, assign users, search users and groups
- Global stats (users, files, folders, storage used)
- Announcements targeted at everyone or at specific groups

**Authentication**
- Flask-Login sessions, optional "remember me" (30 days)
- Password policy: 8+ chars, upper, lower, digit, special character
- Login flow: Google reCAPTCHA, then a simple arithmetic captcha, then credential check
- Brute-force throttling: after 3 failed attempts, exponential wait (5 s, 10 s, 20 s ... capped at 300 s)

## Stack

Python, Flask 2.0.3, Flask-SQLAlchemy (SQLite), Flask-Login, `cryptography`, `requests`, Jinja2 templates.

## Project layout

```
app.py            # Flask app: models, routes, API endpoints
utils.py          # Crypto helpers (RSA keys, AES, HMAC, signatures)
create_db.py      # (Re)creates the SQLite DB and the admin account
requirements.txt
templates/        # Jinja2 pages (login, captcha, browse, upload, messagerie, admin_panel, ...)
static/           # CSS, JS, images
uploads/          # Stored files (UUID-prefixed names)
keys/             # Per-user RSA keys: user_<id>_private.pem / user_<id>_public.pem
cloud.db          # SQLite database
```

## Getting started

```bash
git clone https://github.com/Touhoustic/orchida.git
cd orchida
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

export SECRET_KEY="change-me"   # Windows: set SECRET_KEY=change-me

python create_db.py             # creates cloud.db and the admin account
python app.py                   # http://127.0.0.1:5000
```

`create_db.py` **deletes any existing `cloud.db`** before recreating it.

> Note: `create_db.py` seeds `admin@example.com` with password `admin123`, which does not satisfy the password policy enforced by `User.set_password` (no uppercase, no special character), so the script raises a `ValueError`. Change the password in the script to a compliant one (e.g. `Admin123!`) before running it.

## Configuration

| Setting | Where | Default |
|---|---|---|
| `SECRET_KEY` | env var | `your-secret-key-here` (change it) |
| Database | `SQLALCHEMY_DATABASE_URI` in `app.py` | `sqlite:///cloud.db` |
| Upload folder | `UPLOAD_FOLDER` in `app.py` | `uploads` |
| Max upload size | `MAX_CONTENT_LENGTH` in `app.py` | 100 MB |
| Default quota | `User.storage_limit` | 5 GB |

## Main routes

| Route | Purpose |
|---|---|
| `/login`, `/captcha`, `/logout`, `/forgot-password` | Authentication |
| `/user/dashboard`, `/admin/dashboard` | Dashboards |
| `/browse`, `/browse/<folder_id>` | Folder browser |
| `/upload`, `POST /api/files/upload` | Upload page and API |
| `/download/<id>`, `/view_file/<id>` | Download / preview |
| `/api/files/<id>` (DELETE), `/api/files/<id>/verify` | Delete / integrity check |
| `POST /api/folders`, `DELETE /api/folders/<id>`, `/api/folders/<id>/groups` | Folder management and sharing |
| `/messagerie`, `/api/messages`, `/api/messages/<user_id>`, `/api/messages/unread` | Encrypted messaging |
| `/admin/panel`, `/admin/users`, `/admin/groups`, `/admin/users/<id>/reset-password` | Administration |
| `/report`, `/contact_us` | Announcements, contact form |

## Security notes and known limitations

This project is a work in progress and should not be exposed to the internet as is.

- **Hardcoded secrets**: the VirusTotal API key and the reCAPTCHA secret are in `app.py`. Move them to environment variables and rotate them.
- **Committed sensitive data**: `keys/*.pem` (private keys, unencrypted), `cloud.db`, `instance/*.db`, `uploads/` and `app.log` are tracked in git. Add them to `.gitignore` and purge them from history.
- **RSA keys regenerated at startup**: `python app.py` regenerates every user's RSA key pair, which makes previously received messages undecryptable (and signature checks fail). Remove that block from the `__main__` section for persistent use.
- **File HMAC key is ephemeral**: `SHARED_HMAC_KEY` is random per process start, so integrity checks of files uploaded before a restart will fail.
- **Files are stored unencrypted** on disk; only messages are end-to-end encrypted. Sender copies are kept in clear text in `original_content`.
- **No CSRF protection** on form/JSON endpoints, and `/uploads/<id>` serves files without authentication.
- **Credentials kept in the session** between the login and captcha steps (`pending_password`).
- Runs with `debug=True`; use gunicorn (already in `requirements.txt`) behind HTTPS for any real deployment.
- No self-registration route (`register.html` exists but is not wired); accounts are created by an admin.
- The `3/` directory appears to be an older snapshot of the project.

## License

No license specified.
