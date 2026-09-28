# BEC Athlétisme

Site du **Bordeaux Étudiants Club**, section athlétisme : le doyen des clubs universitaires
français, fondé en 1897. Le site présente le club (histoire, palmarès, équipe, partenaires), son
effectif et ses records **synchronisés depuis la FFA**, son calendrier de compétitions, un magazine
(articles et galerie photo/vidéo) et un formulaire d'inscription.

En ligne : **http://34.138.67.164** (VM Google Cloud, HTTP seulement tant qu'il n'y a pas de nom de
domaine, cf. [Limites connues](#limites-connues)).

Ce dépôt est le **dépôt parent** : il ne contient pas de code, seulement les deux applications en
sous-modules Git et la documentation d'infrastructure.

| Dépôt | Rôle | Documentation |
| --- | --- | --- |
| [`bec-backend`](bec-backend) | API REST : FastAPI, SQLAlchemy, SQLite | [bec-backend/README.md](bec-backend/README.md) |
| [`bec-frontend`](bec-frontend) | Site public et pages d'administration : React, Vite, Tailwind | [bec-frontend/README.md](bec-frontend/README.md) |
| ce dépôt | Versions des deux sous-modules et procédure d'infrastructure | [DEPLOYMENT.md](DEPLOYMENT.md) |

---

## Vue d'ensemble

```
                        ┌──────────────── VM Google Cloud (us-east1, e2-micro) ──────────────┐
                        │                                                                    │
 Navigateur ──HTTP:80──▶│  container frontend : Caddy                                        │
                        │    ├─ /         site React compilé (fichiers pré-compressés brotli) │
                        │    └─ /api/*  ──▶ container backend : FastAPI (gunicorn, 2 workers) │
                        │                     └─ SQLite  ./data/bec.db (volume Docker)       │
                        └──────────────────────────────┬─────────────────────────────────────┘
                                                       │
                     ┌─────────────────────────────────┼──────────────────────────────┐
                     ▼                                 ▼                              ▼
             athle.fr (FFA)                     Cloudinary                     Brevo (SMTP)
      résultats des athlètes, scrapés   photos et vidéos envoyées depuis   e-mails du formulaire
      par les scripts de synchro        l'admin (galerie, articles, équipe)      de contact
```

- **Une seule origine.** Le navigateur ne parle qu'à Caddy : le site appelle des URL relatives
  `/api/...`, que Caddy relaie vers le backend en retirant le préfixe. Pas de CORS en production,
  et le même schéma en développement (le serveur Vite fait le même relais).
- **Deux sortes d'images.** Les photos **éditoriales** du site (bandeaux, photos du club) sont des
  fichiers du front (`bec-frontend/public/photos`), servis par Caddy en plusieurs largeurs
  pré-générées. Les **médias envoyés depuis l'admin** (galerie, images d'articles, portraits de
  l'équipe) vont sur Cloudinary, et seule leur URL est stockée en base.
- **Aucune donnée sportive saisie à la main.** Les athlètes et leurs résultats viennent de la FFA ;
  records, classements et niveaux sont calculés à partir de ces résultats.

## Technologies

| Couche | Choix | Pourquoi |
| --- | --- | --- |
| Front | **React 19** + **TypeScript**, compilé par **Vite 8** | SPA découpée par page : seul l'accueil est dans le premier chargement. |
| Style | **Tailwind CSS 4**, polices Barlow Condensed et Inter | Tokens de couleur dans `src/index.css`, thème clair par défaut et sombre selon le système. |
| Données côté front | **TanStack Query** + **Axios** | Cache et déduplication des appels à l'API. |
| Animations | **Framer Motion** | Apparitions au défilement, transitions de page. |
| Éditeur d'articles | **Tiptap** (ProseMirror) + DOMPurify | Figures, grilles de médias, vidéos, mentions d'athlètes. |
| Graphiques, PDF | Recharts, jsPDF | Progression d'un athlète, export d'un article. |
| API | **FastAPI** + **Pydantic 2** | Validation des entrées, documentation Swagger générée (`/api/docs`). |
| Base de données | **SQLAlchemy 2** sur **SQLite** | Un fichier sur le disque de la VM ; PostgreSQL possible via `DATABASE_URL`. |
| Médias | **Cloudinary** | Stockage, redimensionnement et formats (AVIF/WebP) à la livraison. |
| Données FFA | requests + BeautifulSoup | Scraping des fiches et des résultats par club sur athle.fr. |
| Sécurité du HTML | nh3 (serveur) et DOMPurify (navigateur) | Le HTML des articles est assaini deux fois. |
| Serveur web | **Caddy 2** | Fichiers statiques, reverse proxy, en-têtes de cache. |
| Outils | uv, Task, pytest (back) ; npm, oxlint, tsc (front) | |
| Déploiement | Docker, GitHub Actions, GitHub Container Registry | Un `git push` sur `main` redéploie l'application concernée. |

## D'où viennent les contenus

C'est la question la plus fréquente : « où est-ce que je modifie ça ? »

| Contenu | Source | Où le modifier |
| --- | --- | --- |
| Athlètes du club | Recherche des résultats par club sur la FFA (numéro de club `033010`) | `task import:roster` (backend) |
| Résultats, records, classements | Fiches FFA de chaque athlète | `task sync:db` ; records et classements sont calculés par l'API à partir des résultats |
| Articles du Mag | Base de données, images sur Cloudinary | `/blog/admin` sur le site (éditeur Tiptap) |
| Galerie photo/vidéo | Base de données, fichiers sur Cloudinary | `/galerie/admin` sur le site |
| Calendrier des compétitions | Base de données | API `/api/calendrier` (Swagger : `/api/docs`) |
| Bureau et encadrement | Base de données, portraits sur Cloudinary | API `/api/coachs` (Swagger : `/api/docs`) |
| Textes du site : histoire, palmarès, valeurs, créneaux, partenaires | Fichiers du front | `bec-frontend/src/data/*.ts` puis redéploiement du front |
| Photos du site : bandeaux, photos du club | Fichiers du front | `bec-frontend/public/photos`, puis `node scripts/photo-variants.mjs` |
| Demandes du formulaire de contact | Envoyées par e-mail, jamais stockées | Variables `SMTP_*` et `MAIL_TO` du backend |

Les pages d'administration (`/blog/admin`, `/galerie/admin`…) ne sont **pas protégées** : voir
[Limites connues](#limites-connues).

### Articles d'exemple

Trois faux articles montrent le Mag rempli tant que le club n'a rien publié : la rentrée, une
séance de départs et les interclubs. Ils utilisent les photos du site et des faits tirés du site
lui-même (créneaux, calendrier), sans nom ni résultat inventé. Ils sont insérés et retirés à part,
pour qu'une relance du seed ne les fasse jamais revenir :

```bash
task seed:articles                 # insère (idempotent)
task seed:articles -- --remove     # retire ces trois articles, et eux seuls
```

Sur la VM, les mêmes commandes passent par `docker compose exec`, cf. [DEPLOYMENT.md §7](DEPLOYMENT.md).

## Démarrer en local

Prérequis : [uv](https://docs.astral.sh/uv/), [Task](https://taskfile.dev/)
(`brew install go-task`) et Node 22 ou plus.

```bash
git clone --recurse-submodules git@github.com:rolandkia/bec-athletism.git
cd bec-athletism

# 1. API — http://127.0.0.1:8000, Swagger sur /docs
cd bec-backend
uv sync
task init:db          # base de test : 5 athlètes, l'équipe, le calendrier, 3 articles
task run:api

# 2. Site — http://localhost:5173 (autre terminal)
cd bec-frontend
npm install
npm run dev
```

- `task init:db` recrée `database/bec.db` de zéro. Pour avoir tout l'effectif et ses résultats,
  enchaîner avec `task import:roster` puis `task sync:db` (plusieurs minutes : un appel à la FFA
  par athlète et par saison).
- Les portraits de l'équipe sont lus dans `bec-pictures/photo_profile/`, un dossier **non
  versionné**, et envoyés vers Cloudinary (clés dans `bec-backend/.env`, cf. `.env.example`). Sans
  eux, l'équipe est insérée sans photo et le site affiche des initiales.
- Pour développer le front sans lancer l'API : `VITE_API_PROXY_TARGET=http://34.138.67.164/api npm run dev`
  branche le site local sur l'API de production. Attention : les pages d'admin écrivent alors en
  production.

## Déployer

Il n'y a rien à lancer à la main : **un `git push` sur `main` d'un sous-module le redéploie.**

```
push sur main ──▶ GitHub Actions
                    1. backend : pytest            front : oxlint + tsc + vite build
                    2. image Docker ──▶ ghcr.io/rolandkia/bec-backend | bec-frontend
                    3. SSH sur la VM ──▶ docker compose pull && docker compose up -d
```

Le dépôt parent n'est pas déployé : il enregistre quelle version de chaque sous-module va avec
laquelle. Après un changement dans un sous-module :

```bash
cd bec-frontend && git commit … && git push     # déclenche le déploiement du front
cd .. && git add bec-frontend && git commit -m "Suivre …" && git push
```

La VM (installation, `docker-compose.yml` et ses secrets, CI, commandes utiles, hygiène mémoire,
mesures de performance) est décrite pas à pas dans [DEPLOYMENT.md](DEPLOYMENT.md).

## Performance

La VM est aux États-Unis et le public à Bordeaux : environ 110 ms par aller-retour, en HTTP/1.1
(pas de TLS, donc pas de HTTP/2). Presque tous les choix du front visent à **réduire le nombre
d'allers-retours** avant le premier affichage :

- **Premier affichage en 5 fichiers** (≈165 ko en brotli), pré-compressés au build. Les autres
  pages arrivent en morceaux séparés, préchargés au survol ou après le chargement.
- **Photo d'ouverture demandée dès la lecture du HTML**, à la bonne largeur, sur chaque page.
- **Photos en plusieurs largeurs** (`srcset`) : un téléphone ne télécharge pas l'image d'un écran 4K.
- **Cache HTTP** : fichiers hachés immuables, photos 30 jours, réponses de l'API 1 à 10 minutes.

Chiffres et détails : [DEPLOYMENT.md §8](DEPLOYMENT.md), et les commentaires du code, qui donnent la
mesure derrière chaque réglage.

## Limites connues

Par ordre de priorité :

1. **L'API n'a pas d'authentification.** N'importe qui peut créer, modifier ou supprimer des
   articles, des médias, des membres de l'équipe ou des évènements, via les pages `/…/admin` ou
   directement via `/api/docs`. Les pages d'admin ne sont liées que depuis les boutons « Gérer… »
   du Mag.
2. **Pas de HTTPS.** Le formulaire de contact transmet nom, e-mail et téléphone en clair. Un nom
   de domaine suffit : Caddy obtient alors le certificat tout seul, et HTTP/2 vient avec
   (cf. DEPLOYMENT.md §9).
3. **Photos servies depuis les États-Unis.** La bascule vers Cloudinary est prête
   (`task upload:site-photos`, puis `VITE_CLOUDINARY_CLOUD_NAME` au build du front).
4. **Adresse de réception du formulaire.** `bec-athle.fr` n'existe pas en DNS : `MAIL_TO` doit
   viser une boîte réelle (cf. DEPLOYMENT.md §4).
5. Le `docker-compose.yml` de production n'existe que sur la VM (il contient les secrets).

## Conventions

- Code, commentaires, documentation et messages de commit **en français**.
- Les commentaires expliquent le **pourquoi**, avec la mesure qui a motivé le choix (octets,
  millisecondes, contraste) : c'est ce qui empêche de défaire une optimisation par mégarde.
- Backend : chaque correction arrive avec son test (`uv run pytest`, lancé par la CI avant tout
  déploiement). Front : `npm run lint` et `npm run build` (qui inclut `tsc`) doivent passer.
- Toute règle métier dupliquée entre le back et le front (saison, niveau) est signalée des deux
  côtés, et doit évoluer des deux côtés.
