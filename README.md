<div align="center">

```
╔════════════════════════════════════════════════════════╗
║                                                        ║
║   ░█▀▄░█▀▀░█▀█░█░░░█▀█░█░█░░░█▀▄░█▀█░█▀▀░█░█░█▀▀░░░   ║
║   ░█░█░█▀▀░█▀▀░█░░░█░█░░█░░░░█▀▄░█░█░█░█░█▀█░█▀▀░░░   ║
║   ░▀▀░░▀▀▀░▀░░░▀▀▀░▀▀▀░░▀░░░░▀▀░░▀▀▀░▀▀▀░▀░▀░▀▀▀░░░   ║
║                                                        ║
║      Server & Deployment Playbook  ·  Fullstack        ║
╚════════════════════════════════════════════════════════╝
```

# 🚀 Server & Deployment Playbook

**Von `git push` bis „läuft live" – Django-Backend + Frontend, Schritt für Schritt**

![Python](https://img.shields.io/badge/Python-3.x-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-REST-092E20?style=for-the-badge&logo=django&logoColor=white)
![Gunicorn](https://img.shields.io/badge/Gunicorn-WSGI-499848?style=for-the-badge&logo=gunicorn&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-Reverse_Proxy-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Cloud_Server-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

[🌐 Projekte](https://www.marioramirez.de/projekte/) · [🚨 Notfall-Befehle](#-notfall-befehle)

</div>

---

## 🗺️ Architektur auf einen Blick

```mermaid
flowchart LR
    U[🌍 Browser] -->|HTTPS| FE[🎨 Frontend<br/>Hostinger]
    FE -->|fetch API| NG[Nginx + Certbot<br/>api‹nummer›.marioramirez.de]
    NG -->|Unix-Socket| GU[Gunicorn]
    GU --> DJ[Django]
    GH[(GitHub<br/>main)] -.->|git pull| DJ
    SD[systemd] -.->|startet & überwacht| GU
    DNS[netcup DNS<br/>A-Record] -.->|zeigt auf Server| NG
```

## 📌 Standard-Struktur & URLs

| | |
|:--|:--|
| 🎨 **Frontend** | `https://www.marioramirez.de/projekte/<frontend-ordner>/` (Hostinger / Webhosting) |
| ⚙️ **Backend-API** | `https://api<nummer>.marioramirez.de/api/` (Ubuntu Cloud Server `213.160.75.13`) |
| 🔑 **SSH-Zugang** | `ssh marito1010@213.160.75.13` |
| 📂 **Projektpfad** | `/var/www/projekte/<projektname>-backend/` |
| ⚙️ **Dienstname** | `<projektname>-backend` (systemd + Nginx + Socket) |

> [!TIP]
> Ersetze überall `<projektname>` durch den Namen deines Projekts, `<nummer>` durch die Subdomain-Nummer (z. B. `api2`, `api3`) und `<frontend-ordner>` durch den Ordner auf Hostinger.
> Der Name `<projektname>-backend` bleibt in **allen** Schritten identisch (Ordner, Service, Nginx, Socket).

---

## 🧭 Schritte

| # | Schritt | Wo |
|:-:|:--|:--|
| 0 | [Subdomain anlegen (DNS)](#0--subdomain-anlegen-netcup-ccp) | 🌐 netcup |
| 1 | [Lokales Backend pushen](#1--lokales-backend-vorbereiten--pushen) | 💻 PC |
| 2 | [Repository klonen & einrichten](#2--auf-dem-server-repository-klonen--einrichten) | ☁️ Server |
| 3 | [`settings.py` & `.env` anpassen](#3--django-settingspy-anpassen) | ☁️ Server |
| 4 | [Migrationen, Static Files & Accounts](#4--migrationen-static-files--gast-accounts) | ☁️ Server |
| 5 | [Gunicorn-Service einrichten](#5--gunicorn-systemd-service) | ☁️ Server |
| 6 | [Nginx & SSL einrichten](#6--nginx-konfigurieren--ssl-einrichten) | ☁️ Server |
| 7 | [Frontend anbinden & hochladen](#7--frontend-anbinden--hochladen) | 🎨 Hostinger |

---

## 0 · Subdomain anlegen (netcup CCP)

> [!IMPORTANT]
> Ohne A-Record schlägt Certbot in Schritt 6 sofort fehl. Danach ein paar Minuten warten, bis der Eintrag aktiv ist.

1. Im netcup CCP einloggen (`customercontrolpanel.de`).
2. **Domains** ➔ Lupe bei `marioramirez.de` ➔ **DNS**.
3. Neuen Eintrag hinzufügen:

| Feld | Wert |
|:--|:--|
| **Hostname** | `api<nummer>` (z. B. `api2`, `api3`) |
| **Typ** | `A` |
| **Ziel** | `213.160.75.13` |
| **TTL** | `300` |

4. **Zonenrevision speichern**.

---

## 1 · Lokales Backend vorbereiten & pushen

Im lokalen Projektordner auf deinem PC:

```bash
git add .
git commit -m "Ready for production"
git push origin main
```

---

## 2 · Auf dem Server: Repository klonen & einrichten

Per SSH verbinden:

```bash
ssh marito1010@213.160.75.13
```

In das Verzeichnis wechseln und klonen:

```bash
cd /var/www/projekte
git clone <GITHUB_REPO_URL> <projektname>-backend
cd <projektname>-backend
```

Virtuelle Umgebung anlegen und Abhängigkeiten installieren:

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn
```

---

## 3 · Django `settings.py` anpassen

```bash
nano core/settings.py
```

**✅ Checkliste**

**`ALLOWED_HOSTS`**

```python
ALLOWED_HOSTS = [
    'api<nummer>.marioramirez.de',
    'marioramirez.de',
    'www.marioramirez.de',
    '213.160.75.13',
    'localhost',
    '127.0.0.1',
]
```

**`CORS_ALLOWED_ORIGINS`**

> [!WARNING]
> Nur die Origin eintragen – **ohne Pfad und ohne Slash am Ende!**

```python
CORS_ALLOWED_ORIGINS = [
    "https://marioramirez.de",
    "https://www.marioramirez.de",
]
# Alternativ für reine Testzwecke:
# CORS_ALLOW_ALL_ORIGINS = True
```

**Statische Pfade prüfen**

> [!NOTE]
> `STATIC_ROOT` heißt `staticfiles` – genau so wird der Ordner in Nginx (Schritt 6) ausgeliefert.

```python
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

💾 Speichern: `Strg + O` → `Enter` &nbsp;|&nbsp; ❌ Schließen: `Strg + X`

**`.env` anlegen**

```bash
nano .env
```

```env
SECRET_KEY=dein-geheimer-schlüssel-aus-deinem-lokalen-pc-hier-einfügen
DEBUG=False
```

💾 Speichern: `Strg + O` → `Enter` &nbsp;|&nbsp; ❌ Schließen: `Strg + X`

---

## 4 · Migrationen, Static Files & Gast-Accounts

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

<details>
<summary><b>👤 Gast-Accounts ohne Passwort-Sperre erstellen</b></summary>

<br>

Über die Django-Shell werden die Passwort-Validierungen umgangen (z. B. „zu kurz" bei `asdasd`):

```bash
python manage.py shell -c "
from django.contrib.auth import get_user_model
User = get_user_model()

# 1. Kunde
u1, _ = User.objects.get_or_create(username='andrey')
u1.set_password('asdasd')
if hasattr(u1, 'type'): u1.type = 'customer'
u1.save()

# 2. Geschäftskonto
u2, _ = User.objects.get_or_create(username='kevin')
u2.set_password('asdasd24')
if hasattr(u2, 'type'): u2.type = 'business'
u2.save()

print('Accounts erfolgreich eingerichtet!')
"
```

</details>

---

## 5 · Gunicorn Systemd-Service

Service-Datei anlegen:

```bash
sudo nano /etc/systemd/system/<projektname>-backend.service
```

Inhalt:

```ini
[Unit]
Description=Gunicorn daemon for <projektname>-backend
After=network.target

[Service]
User=marito1010
Group=www-data
WorkingDirectory=/var/www/projekte/<projektname>-backend
ExecStart=/var/www/projekte/<projektname>-backend/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/var/www/projekte/<projektname>-backend/<projektname>-backend.sock \
          core.wsgi:application

[Install]
WantedBy=multi-user.target
```

💾 Speichern: `Strg + O` → `Enter` &nbsp;|&nbsp; ❌ Schließen: `Strg + X`

Dienst aktivieren und starten:

```bash
sudo systemctl daemon-reload
sudo systemctl start <projektname>-backend
sudo systemctl enable <projektname>-backend
sudo systemctl status <projektname>-backend --no-pager
```

---

## 6 · Nginx konfigurieren & SSL einrichten

> [!IMPORTANT]
> Reihenfolge einhalten: **1. Config anlegen → 2. Symlink → 3. Nginx testen → 4. Certbot.**
> Den SSL-Block **nicht** selbst schreiben – Certbot trägt ihn automatisch ein. Das Zertifikat existiert vorher noch nicht, Nginx würde sonst nicht starten.

Nginx-Konfigurationsdatei erstellen:

```bash
sudo nano /etc/nginx/sites-available/<projektname>-backend
```

Inhalt (Basis-Block ohne SSL):

```nginx
server {
    server_name api<nummer>.marioramirez.de;

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location /static/ {
        alias /var/www/projekte/<projektname>-backend/staticfiles/;
    }

    location /media/ {
        alias /var/www/projekte/<projektname>-backend/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/projekte/<projektname>-backend/<projektname>-backend.sock;
    }
}
```

Aktivieren und Nginx neu laden:

```bash
sudo ln -s /etc/nginx/sites-available/<projektname>-backend /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

SSL-Zertifikat via Certbot abrufen:

```bash
sudo certbot --nginx -d api<nummer>.marioramirez.de
```

Erreichbarkeit prüfen:

```bash
curl -i https://api<nummer>.marioramirez.de/admin/
```

---

## 7 · Frontend anbinden & hochladen

**`config.js` im Frontend:**

```javascript
const API_BASE_URL = 'https://api<nummer>.marioramirez.de/api/';
const STATIC_BASE_URL = 'https://api<nummer>.marioramirez.de/';
```

**Dateien hochladen:** per FileZilla / FTP in das Hostinger-Verzeichnis

```
public_html/projekte/<frontend-ordner>/
```

**Im Browser öffnen:** `https://www.marioramirez.de/projekte/<frontend-ordner>/`

> [!NOTE]
> Cache leeren mit `Strg + F5`.

---

## 🚨 Notfall-Befehle

| 🔥 Problem | 💻 Befehl |
|:--|:--|
| Code in `settings.py` oder View geändert | `sudo systemctl restart <projektname>-backend` |
| HTTP 400 Bad Request / Fehlersuche | `sudo journalctl -u <projektname>-backend -n 30 --no-pager` |
| Neuen Stand von GitHub auf Server ziehen | `git pull origin main && sudo systemctl restart <projektname>-backend` |
| Nginx-Syntax testen | `sudo nginx -t` |
| API im Terminal prüfen | `curl -i https://api<nummer>.marioramirez.de/api/base-info/` |

> [!TIP]
> `git pull` immer im Projektordner ausführen: `cd /var/www/projekte/<projektname>-backend`

---

<div align="center">

**Gebaut mit ☕ und Terminal von [Mario Ramirez](https://www.marioramirez.de)**

</div>
