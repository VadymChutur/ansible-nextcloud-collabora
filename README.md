# Nextcloud + Collabora Online behind Tailscale

## Prerequisites
1. Enable **MagicDNS** and **HTTPS Certificates** in the Tailscale admin console.
   Connect the VM to the tailnet manually: `sudo tailscale up` (the playbook only verifies that the node is connected). 
   To prevent the domain and certificate from expiring, disable *Key expiry* for this node in the Tailscale admin panel.
2. Place custom fonts (`.ttf`/`.otf`) into the `files/fonts/` directory (the folder can be left empty).
3. Install required Ansible collections: `ansible-galaxy collection install -r requirements.yml`
4. Add your public SSH key to the `root` user. The deployment must be executed as `root` since several tasks require direct root privileges without `sudo` prompt interruptions.

## Deployment
```bash
ansible-playbook -i inventory.ini site.yml
```

Idempotent deployment: re-running the playbook is perfectly safe. Tailscale won't reconnect, existing users are skipped, and only services with modified configurations will be restarted.

##  Architecture
- `web` (Nginx, host network mode) — listens on port 443, allowing traffic exclusively from the Tailnet and local Docker containers.
  - `/` → proxies to Nextcloud (Apache) at 127.0.0.1:8080
  - `/browser`, `/hosting/*`, `/cool/*` → proxies to Collabora at 127.0.0.1:9980
- `app` + `cron` — Nextcloud core; `db` (PostgreSQL) and `redis` run in an isolated internal Docker network without internet access.

- Nextcloud and Collabora communicate with each other via `https://<domain>.ts.net` (the domain resolves to the host internally via `extra_hosts: host-gateway`).

## Backups
- `pg-backup.timer` — runs daily at 03:00 AM, saving SQL dumps to `/opt/nextcloud/backups` and retaining them for 7 days.

- **Warning:** This only covers the database. User files are stored in the `nextcloud_data` Docker volume — they must be backed up separately using tools like Restic or Borg, preferably to an off-site host.

- `tailscale-cert.timer` — runs weekly to renew the Tailscale TLS certificate and gracefully reload Nginx.

## Updates
Upgrade the `nextcloud_image` variable strictly one major version at a time (e.g., 35 → 36 → 37). Always create a ZFS snapshot before initiating an upgrade.

## Verification
- `https://<domain>/hosting/discovery` — should return a valid XML response.
- `Nextcloud` → `Administration settings` → `Office`: status should display "Collabora Online server is reachable".

# Nextcloud + Collabora Online за Tailscale

## Підготовка
1. У Tailscale admin console увімкни **MagicDNS** та **HTTPS Certificates**.
   Підключи ВМ до tailnet вручну: `sudo tailscale up` (плейбук лише перевіряє, що вузол у мережі).
   Щоб сертифікат і домен не злетіли, в адмінці вимкни для цього вузла *Key expiry*.
2. Поклади шрифти (`.ttf`/`.otf`) у `files/fonts/` (папка може бути порожньою).
3. Встанови колекції: `ansible-galaxy collection install -r requirements.yml`
4. Додати пабкей для root користувача. Розгортати під root користувача, оскільки деякі дії потребують root прав.

## Запуск
```bash
ansible-playbook -i inventory.ini site.yml
```
Повторний запуск безпечний: Tailscale не перепідключається, існуючі користувачі пропускаються,
змінені конфіги перезапускають лише свій сервис.

## Схема
- `web` (Nginx, host network) — слухає 443, пускає лише Tailscale + контейнери.
  - `/` → Nextcloud (apache) на `127.0.0.1:8080`
  - `/browser`, `/hosting/*`, `/cool/*` → Collabora на `127.0.0.1:9980`
- `app` + `cron` — Nextcloud; `db` (Postgres) і `redis` у внутрішній мережі без інтернету.
- Nextcloud і Collabora ходять одне до одного через `https://<домен>.ts.net`
  (домен вказує на хост через `extra_hosts: host-gateway`).

## Бекапи
- `pg-backup.timer` — щодня о 03:00, дампи в `/opt/nextcloud/backups`, зберігаються 7 днів.
- **Увага:** це лише БД. Файли користувачів лежать у volume `nextcloud_data` — їх теж
  треба бекапити (restic/borg), бажано на інший хост.
- `tailscale-cert.timer` — щотижня оновлює TLS-сертифікат і перезавантажує Nginx.

## Оновлення
Змінюй `nextcloud_image` лише на наступну мажорну версію (35 → 36 → 37), перед цим — снапшот ZFS.

## Перевірка
- `https://<домен>/hosting/discovery` — має віддати XML.
- Nextcloud → Налаштування адміністратора → Office: статус «Collabora Online server is reachable».
