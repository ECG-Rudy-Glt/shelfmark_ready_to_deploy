# Shelfmark — prêt à déployer

Déploiement conteneurisé de [Shelfmark](https://github.com/calibrain/shelfmark) prêt à
l'emploi pour la recherche/téléchargement direct via Anna's Archive (LibGen, Z-Library et
Welib en plus) — aucun indexeur torrent ni débrideur requis pour ce mode. Zéro code custom :
tout passe par l'image officielle et ses variables d'environnement.

## Démarrage rapide

```bash
git clone https://github.com/ECG-Rudy-Glt/shelfmark_ready_to_deploy.git
cd shelfmark_ready_to_deploy
cp .env.example .env
# Édite .env si besoin (langue, port, auth...) — les mirroirs Anna's Archive
# par défaut sont déjà renseignés et fonctionnels.
docker compose up -d
```

Puis ouvre `http://localhost:8084` (ou le port choisi via `SHELFMARK_PORT`).

## Déploiement via Ansible (alternative à `docker compose` direct)

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
cp ../.env.example ../.env && $EDITOR ../.env
ansible-playbook deploy-shelfmark.yml
```

Cible `localhost` par défaut (déploiement local, `connection: local`). Pour déployer sur une
autre machine (Docker déjà installé dessus — ce playbook ne l'installe pas, volontairement
portable vers n'importe quelle distro) :

```bash
ansible-playbook deploy-shelfmark.yml -e target=mon-hote -e ansible_user=mon-user
```

Pour arrêter et supprimer la stack :

```bash
ansible-playbook deploy-shelfmark.yml -e shelfmark_state=absent
```

## Fichiers de ce repo

```
docker-compose.yml   service shelfmark — image officielle, shm_size 2gb (voir plus bas)
.env.example          toutes les variables utiles, commentées, valeurs par défaut sûres
ansible/
  deploy-shelfmark.yml  déploiement idempotent via community.docker.docker_compose_v2
  requirements.yml      collection Ansible requise
data/                  volumes bind-mount (books, config) — créé au premier lancement, ignoré par git
```

## Le piège `/dev/shm` (pourquoi `shm_size: "2gb"` est déjà dans le compose)

Le bypasser anti-bot **intégré** à Shelfmark (utilisé pour passer la protection DDoS-Guard
des mirroirs Anna's Archive) lance un Chromium headless. Avec les 64 Mo de `/dev/shm` que
Docker alloue par défaut, ce navigateur ne démarre pas (`Pure CDP browser startup failed`)
— **non documenté par le projet upstream**, trouvé en déploiement réel. `docker-compose.yml`
règle déjà ça (`shm_size: "2gb"`), rien à faire de plus.

Piège associé, déjà évité ici : `USING_EXTERNAL_BYPASSER=true` (bascule vers un service
externe type FlareSolverr) **ne corrige pas** le problème et casse même la recherche Anna's
Archive — FlareSolverr est câblé pour résoudre des challenges Cloudflare, pas DDoS-Guard
([issue upstream FlareSolverr #886](https://github.com/FlareSolverr/FlareSolverr/issues/886),
fermée "not planned"). Le bypasser interne (`USE_CF_BYPASS=true`,
`USING_EXTERNAL_BYPASSER=false`, les valeurs par défaut de `.env.example`) est la seule
option qui fonctionne pour cette source.

## Aller plus loin

- **Sortie des livres** : par défaut les fichiers atterrissent dans `./data/books`
  (`BOOKS_OUTPUT_MODE=folder`). Deux autres modes disponibles côté Shelfmark : `email` (envoi
  SMTP direct, pratique pour une Kindle) et `booklore` (upload API vers une instance
  Grimmory/Calibre-Web Automated) — variables `EMAIL_*`/`BOOKLORE_*` documentées dans la
  référence upstream ci-dessous.
- **Mode `universal`** (indexeurs torrent + débrideur, en plus d'Anna's Archive) : section
  commentée en bas de `.env.example` (`PROWLARR_*`/`ALLDEBRID_API_KEY`/`REALDEBRID_API_KEY`).
  Nécessite ta propre instance Prowlarr, pas fournie par ce repo.
- **Authentification** : `AUTH_METHOD=none` par défaut (usage local/LAN mono-machine).
  Passe à `builtin` (comptes locaux) si l'instance devient accessible à plusieurs personnes
  ou au-delà de ta seule machine — voir aussi `proxy`/`oidc` dans la doc upstream si tu la
  mets derrière un reverse proxy avec SSO.
- **Référence complète des variables** :
  [docs/environment-variables.md](https://github.com/calibrain/shelfmark/blob/main/docs/environment-variables.md)
  du projet upstream.

## Notes

- Pas d'app native : Shelfmark est une web UI (navigateur, ou raccourci écran d'accueil).
- Ne commite jamais `.env` (déjà dans `.gitignore`) — il contient les clés/mots de passe une
  fois les intégrations optionnelles renseignées.
- Usage personnel : respecte le droit d'auteur applicable dans ta juridiction pour les livres
  que tu télécharges.
