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

⚠️ **HTTPS avant d'annoncer le formulaire** : le site est servi en HTTP simple (cf. §9). Le
formulaire collecte nom, e-mail et téléphone — ces données ne doivent pas circuler en clair.
Brancher le domaine + Let's Encrypt avant de communiquer dessus.

Le `Caddyfile` de `bec-frontend` gère le reverse-proxy et le cache. Il n'est
volontairement PAS recopié ici : il a sa propre documentation, en commentaires, au
plus près des règles qu'elle explique (plafond de corps de requête, en-têtes de
cache, fichiers pré-compressés — cf. §8). Sa structure en deux temps :

```
:80 {
	handle_path /api/*  →  reverse_proxy backend:8000   (préfixe /api retiré)
	handle              →  fichiers statiques /srv + try_files … /index.html
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

Importer les nouveaux athlètes du club trouvés sur la FFA (numéro de club BEC `033010`, résultats
des saisons 2024-2025-2026) — safe à relancer, ignore ceux déjà en base ; enchaîner avec la
synchro des résultats ci-dessus pour les nouveaux :
```bash
docker compose exec -e PYTHONPATH=. backend uv run python src/scripts/import_club_roster.py 2024-2025-2026
docker compose exec -e PYTHONPATH=. backend uv run python src/scripts/sync_db.py 2024-2025-2026
```

**Migration `club_bec`** — un résultat FFA n'indique pas sous quel club il a été couru (un athlète
qui change de club importe donc aussi ses anciens résultats) ; chaque `Resultat` porte désormais un
flag `club_bec` qui ne compte que pour le classement du club (la fiche personnelle d'un athlète
prend toutes ses perfs). À lancer **une seule fois** sur une base déjà peuplée (n'efface rien,
contrairement à `init_db.py`) :
```bash
docker compose exec -e PYTHONPATH=. backend uv run python src/scripts/migrate_add_club_bec.py
docker compose exec -e PYTHONPATH=. backend uv run python src/scripts/backfill_club_bec.py
```
`backfill_club_bec.py` peut être relancé sans risque (ex. si la logique de détection FFA évolue à
nouveau) — il revérifie chaque résultat déjà en base et recalcule son flag.

Shell interactif dans le container :
```bash
docker compose exec backend bash
```

## 8) Performance — la latence transatlantique

La VM est en `us-east1`, le public du club à Bordeaux : **~110 ms d'aller-retour**, en HTTP/1.1
(pas de domaine ⇒ pas de TLS ⇒ pas de HTTP/2) et sans CDN. Deux conséquences qui dictent tous les
arbitrages ci-dessous :

1. **Le NOMBRE de requêtes compte autant que leur poids.** Sans multiplexage, le navigateur ouvre
   au plus 6 connexions et chacune paie sa propre poignée de main. Dix fichiers de 1 ko coûtent
   plus cher qu'un fichier de 10 ko.
2. **Tout ce qui peut être évité doit l'être** : cache navigateur, cache applicatif, compression
   maximale.

### Mesures (depuis la France, septembre 2026)

| | avant | après |
|---|---|---|
| Fichiers du premier rendu | 13 | **5** |
| Poids du premier rendu | 200 691 o (zstd à la volée) | **163 177 o** (brotli 11 pré-calculé) |
| `GET /api/athletes/` (pic observé) | **14,5 s** | **0,52 s** |

### Ce qui est en place

- **Découpage des morceaux** (`bec-frontend/vite.config.ts`) : deux groupes, `vendor` et
  `app-shared`, énumérés paquet par paquet. ⚠️ Ne jamais élargir un `test` à tout `node_modules` :
  jspdf, html2canvas, recharts et Tiptap retomberaient dans le bundle du premier rendu.
- **Pré-compression au build** (`bec-frontend/scripts/precompress.mjs`) : brotli 11, zstd 19 et
  gzip 9 écrits à côté de chaque fichier texte de `dist/`. Caddy les sert via
  `file_server { precompressed br zstd gzip }` — pas besoin du greffon brotli, qui ne concerne que
  l'ENCODEUR. Gain double : -18,7 % d'octets, et plus aucune compression à faire à chaque requête
  sur le vCPU partagé.
- **Cache navigateur** (`bec-frontend/Caddyfile`) : `immutable` sur `/assets/*` (noms hachés),
  30 jours sur les photos, `no-cache` sur `index.html`.
- **Cache HTTP des lectures d'API** (`bec-backend/src/api/http_cache.py`) : `STATIC_DATA` (10 min)
  pour l'effectif, les classements et les fiches athlètes ; `EDITED_DATA` (1 min) pour ce qui
  s'édite depuis l'admin — blog, calendrier, coachs, galerie.
  `bec-backend/tests/api/test_http_cache.py` verrouille ces en-têtes : une route publique qui perd
  son `Cache-Control` fait échouer la CI, car rien dans l'interface ne le signalerait.
- **Préchargement des pages au survol** (`bec-frontend/src/lib/prefetch.ts`), volontairement
  différé après l'évènement `load` et désactivé si le navigateur annonce `saveData` ou de la 2G.

### Hygiène de la VM (à refaire si la VM est recréée)

Sur 964 Mo de RAM, les agents Google de supervision en consommaient ~150 Mo et le journal systemd
avait atteint **1,7 Go** sans plafond. Résultat : les workers gunicorn partaient en swap, et la
première requête après une période calme prenait plusieurs secondes — le fameux pic à 14,5 s.
Aucun tableau de bord Cloud Monitoring n'étant utilisé, ces agents ont été désactivés :

```bash
# Plafonner le journal (il n'a AUCUNE limite par défaut : 10 % du disque)
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/00-bec-taille.conf <<'CONF'
[Journal]
SystemMaxUse=100M
SystemMaxFileSize=10M
MaxRetentionSec=1week
CONF
sudo systemctl restart systemd-journald && sudo journalctl --vacuum-size=100M

# Agents de supervision inutilisés (~150 Mo)
sudo systemctl disable --now google-cloud-ops-agent google-osconfig-agent exim4

# Rapatrier ce qui était parti en swap (seulement si `free -m` montre la marge)
sudo swapoff -a && sudo swapon -a
```

⚠️ **Ne JAMAIS désactiver `google-guest-agent`** : c'est lui qui installe les clés SSH depuis les
métadonnées GCP. Le couper ferme l'accès à la VM.

Vérifier après coup : `free -m` (swap à 0, ~450 Mo disponibles) et `docker compose ps`.

## 9) Prochaines étapes possibles

- **HTTPS + CDN — le plus gros gain restant, et il est gratuit.** Il ne manque qu'un nom de domaine
  (un sous-domaine de `bec-bordeaux.fr` suffirait). Avec un domaine : Caddy prend le certificat
  Let's Encrypt tout seul (remplacer `:80` par le nom de domaine dans le `Caddyfile`), ce qui
  débloque **HTTP/2** — donc le multiplexage, donc la fin de la limite à 6 connexions et du coût
  par fichier. Et en plaçant le domaine derrière l'offre gratuite de Cloudflare, les fichiers
  statiques seraient servis depuis un point de présence français : **~10 ms au lieu de ~110 ms**,
  plus HTTP/3 et brotli côté edge. C'est le seul changement qui s'attaque à la distance elle-même —
  tout ce qui est décrit en §8 ne fait que limiter le nombre de fois où on la paie.
- Envisager de committer une version "template" du `docker-compose.yml` (avec des `...` à la place des vraies clés) dans un des repos pour ne pas perdre sa structure, en gardant les vraies valeurs uniquement sur la VM.
