# Déploiement BEC — récap complet

VM Google Cloud (free tier) : e2-micro, us-east1-d, Debian 13 (trixie), 1 vCPU / 1 Go RAM / 30 Go disque.

Architecture : un seul point d'entrée HTTP (port 80) géré par le container **frontend** (Caddy), qui sert le site statique et reverse-proxy `/api/*` vers le container **backend** (FastAPI). Base de données SQLite persistée sur le disque de la VM via un volume Docker.

```
Internet ──:80──▶ [frontend: Caddy]
                      ├─ / *          → fichiers statiques React (SPA)
                      └─ /api/* → backend:8000 (prefix /api strippé)
                                       [backend: FastAPI/gunicorn]
                                          └─ SQLite : ./data/bec.db (volume)
```

## 1) Setup initial de la VM

```bash
sudo apt update && sudo apt upgrade -y

# Swap obligatoire : 1 Go de RAM seul suffit pas (docker + 2 containers → OOM sans ça)
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # reco-nnecte-toi après ça
```

**Piège rencontré** : si `apt update`/`upgrade` échoue avec une erreur de clé GPG Docker manquante (`Sub-process /usr/bin/sqv returned an error code`), un dépôt Docker cassé traînait déjà. Fix :
```bash
sudo rm -f /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.sources
sudo apt update
```
Et si `dpkg` reste bloqué après une interruption (`dpkg was interrupted`) :
```bash
sudo dpkg --configure -a
sudo apt --fix-broken install -y
```
(`google-cloud-cli` peut légitimement prendre 10-20 min à se configurer sur un e2-micro — patience avant de paniquer, vérifier `top`/`free -h` avant d'agir.)

## 2) IP externe statique (gratuite tant qu'attachée à la VM qui tourne)

Compute Engine → l'instance → **Modifier** → Interfaces réseau → passer l'IP externe de "Éphémère" à "Réserver une adresse IP externe statique" (garde le même niveau de réseau que l'instance, déjà pré-rempli).

⚠️ Si un jour la VM est arrêtée en gardant l'IP statique réservée mais non attachée, elle est facturée à l'heure — libérer l'IP si arrêt prolongé.

## 3) Utilisateur SSH dédié pour le déploiement CI (`github-actions`)

Ne jamais réutiliser la clé SSH personnelle pour le CI. Sur le Mac :
```bash
ssh-keygen -t ed25519 -f ~/.ssh/bec_deploy_key -C "github-actions" -N ""
```
Coller le contenu de `bec_deploy_key.pub` dans **Compute Engine → VM → Modifier → Sécurité et accès → Clés SSH → Ajouter un élément**. GCP crée automatiquement l'utilisateur Linux `github-actions` (déduit du commentaire final de la clé).

```bash
# Une fois l'utilisateur créé (id github-actions pour vérifier) :
sudo usermod -aG docker github-actions
```

**Piège rencontré** : le home de l'utilisateur principal (`kia_pro_roland`) était en `700` (`drwx------`), donc `github-actions` ne pouvait même pas *traverser* le dossier pour atteindre `~/bec-deploy` (`Permission denied` dans le CI). Fix :
```bash
chmod o+x /home/kia_pro_roland
```
(donne juste le droit de traversée, pas de lister le contenu — suffisant, `bec-deploy` a déjà les bons droits en dessous)

`github-actions` fait partie du groupe `google-sudoers` → a les droits `sudo` par défaut sur les VM GCP.

## 4) Dossier de déploiement `~/bec-deploy` (sous `/home/kia_pro_roland/bec-deploy`)

```yaml
# docker-compose.yml
services:
  frontend:
    image: ghcr.io/rolandkia/bec-frontend:latest
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend

  backend:
    image: ghcr.io/rolandkia/bec-backend:latest
    restart: unless-stopped
    environment:
      - ROOT_PATH=/api
      - CLOUDINARY_CLOUD_NAME=...
      - CLOUDINARY_API_KEY=...
      - CLOUDINARY_API_SECRET=...
      # Formulaire de contact du site (POST /api/demandes) — Brevo
      - SMTP_HOST=smtp-relay.brevo.com
      - SMTP_PORT=587
      - SMTP_SECURITY=starttls   # ou ssl si port 465
      - SMTP_USER=...            # « adresse email de connexion » Brevo
      - SMTP_PASSWORD=...        # clé SMTP (pas une clé API v3)
      - MAIL_FROM=...            # expéditeur VÉRIFIÉ dans Brevo
      - MAIL_TO=...              # boîte réelle qui reçoit les demandes
    volumes:
      - ./data:/app/database
```
Ce fichier vit **uniquement sur la VM**, jamais commité (contient les vraies clés Cloudinary et le mot de passe SMTP en clair).

**Ajouter/changer une variable d'environnement** ⇒ éditer ce fichier à la main en SSH, puis :
```bash
cd /home/kia_pro_roland/bec-deploy
docker compose up -d backend   # recrée le container avec le nouvel environnement
```
Il n'y a ni gestionnaire de secrets ni `env_file:` : le CI ne déploie que les images, pas la config.

⚠️ **SMTP** : sans `SMTP_HOST` ni `MAIL_TO`, `POST /api/demandes` répond `503` et le formulaire du
site affiche un repli — le reste du site fonctionne normalement. Deux pièges Brevo :
- **`MAIL_FROM` doit être un expéditeur vérifié** dans Brevo (*Senders, Domains & Dedicated IPs →
  Senders*), sinon le relais refuse le message. L'« adresse email de connexion » SMTP n'est pas
  automatiquement un expéditeur valide — donc ne pas compter sur le repli `MAIL_FROM = SMTP_USER`.
- **`MAIL_PASSWORD` = la clé SMTP**, pas une clé API v3. Attention aux espaces en fin de copie
  (erreur `535`).

⚠️ **`MAIL_TO` doit être une boîte réellement joignable.** Au 29/07/2026, `bec-athle.fr` **n'existe
pas en DNS** (aucun A, MX ni NS) : tout envoi vers `contact@bec-athle.fr` rebondit *après* que le
site a affiché « demande envoyée ». Le domaine actif du club omnisports est `bec-bordeaux.fr` (MX
OVH). À trancher avant la mise en service, et à répercuter dans
`bec-frontend/src/data/club.ts` (`contact.email`, l'adresse affichée sur le site).

Vérifier après déploiement : `docker compose logs backend --tail=50`, un envoi réel depuis
`/contact`, **et la réception effective** dans la boîte `MAIL_TO`.

⚠️ **HTTPS avant d'annoncer le formulaire** : le site est servi en HTTP simple (cf. §8). Le
formulaire collecte nom, e-mail et téléphone — ces données ne doivent pas circuler en clair.
Brancher le domaine + Let's Encrypt avant de communiquer dessus.

Le `Caddyfile` de `bec-frontend` gère le reverse-proxy :
```
:80 {
	encode gzip

	handle_path /api/* {
		reverse_proxy backend:8000
	}

	handle {
		root * /srv
		file_server
		try_files {path} /index.html
	}
}
```

## 5) CI/CD GitHub Actions

Chaque repo (`bec-backend`, `bec-frontend`) a son propre `.github/workflows/deploy.yml` : `test` → build l'image Docker → push sur GHCR → SSH sur la VM → `docker compose pull && up -d`.

**Secrets à ajouter sur CHAQUE repo** (Settings → Secrets and variables → Actions — les secrets ne sont PAS partagés entre repos) :
| Secret | Valeur |
|---|---|
| `VM_HOST` | IP statique de la VM |
| `VM_SSH_USER` | `github-actions` |
| `VM_SSH_KEY` | contenu de `~/.ssh/bec_deploy_key` (clé **privée**) |

**Piège rencontré** : le script SSH faisait `cd ~/bec-deploy`, qui pour `github-actions` pointe vers un home vide (`/home/github-actions`) au lieu de `/home/kia_pro_roland/bec-deploy`. Le `cd` échouait silencieusement et comme la dernière commande du script (`docker image prune -f`) réussissait toujours, le job s'affichait "vert" sans avoir rien déployé. Fix appliqué dans les deux `deploy.yml` : chemin absolu + `set -e` pour ne plus jamais avoir de faux positif :
```yaml
script: |
  set -e
  cd /home/kia_pro_roland/bec-deploy
  docker compose pull backend   # ou frontend
  docker compose up -d backend
  docker image prune -f
```

**Images GHCR privées par défaut** → rendre publiques sur `github.com/rolandkia?tab=packages` → package → Package settings → Change visibility → Public (sinon `docker compose pull` échoue en 403 sur la VM).

Une fois tout ça en place : chaque `git push` sur `main` redéploie automatiquement, sans rien faire de manuel.

## 6) Swagger FastAPI derrière le proxy

FastAPI ne sait pas nativement qu'il est servi sous `/api` par Caddy. Ajouté dans `bec-backend/src/main.py` :
```python
app = FastAPI(title="BEC API", root_path=os.getenv("ROOT_PATH", ""))
```
`ROOT_PATH=/api` est défini uniquement dans le `docker-compose.yml` de la VM (pas en local, où le backend tourne directement sur `:8000`). Accès : **`http://<IP_VM>/api/docs`**.

## 7) Commandes utiles sur la VM

Se connecter :
```bash
ssh -i ~/.ssh/bec_deploy_key github-actions@<IP_VM>
cd /home/kia_pro_roland/bec-deploy
```

État des containers / logs :
```bash
docker compose ps
docker compose logs backend --tail=100
docker compose logs frontend --tail=100
```

Initialiser le schéma + données de seed (⚠️ **supprime et recrée `bec.db`** — à ne lancer qu'une fois, jamais une fois qu'il y a des vraies données en prod) :
```bash
docker compose exec -e PYTHONPATH=. backend uv run python src/scripts/init_db.py
```
(nécessite que les photos des coachs soient accessibles au chemin absolu `/bec-pictures/photo_profile/*.png` dans le container — voir volume temporaire ci-dessous si besoin de les re-uploader vers Cloudinary)

Synchroniser les résultats FFA des athlètes déjà en base (safe à relancer autant de fois que voulu) :
```bash
docker compose exec -e PYTHONPATH=. backend uv run python src/scripts/sync_db.py 2024-2025-2026
```

Shell interactif dans le container :
```bash
docker compose exec backend bash
```

## 8) Prochaines étapes possibles

- **HTTPS** : dès qu'un nom de domaine pointe vers l'IP de la VM, Caddy gère le certificat Let's Encrypt automatiquement — remplacer `:80` par le nom de domaine dans le `Caddyfile`.
- Envisager de committer une version "template" du `docker-compose.yml` (avec des `...` à la place des vraies clés) dans un des repos pour ne pas perdre sa structure, en gardant les vraies valeurs uniquement sur la VM.

