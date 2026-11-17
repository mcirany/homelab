# homelab

Infra as code / GitOps du homelab. Ce repo est la source de vérité : un commit sur `main`
est déployé automatiquement par [doco-cd](https://doco.cd), qui tourne sur chaque serveur et
interroge le repo toutes les 3 minutes.

## Arborescence

```
.
├── apps/                    # Applications (un dossier = un projet compose)
│   ├── actual/               # Actual Budget (budget.cineplex.fr)
│   │   └── compose.yml
│   ├── happyplex/            # Happyplex v2 (happy.cineplex.fr)
│   │   ├── compose.yml
│   │   ├── secrets.env.example
│   │   └── secrets.enc.env   # HP_SESSION_KEY / HP_SECRET_KEY, chiffrés SOPS
│   ├── plex/
│   │   └── compose.yml
│   ├── portabase/            # Dashboard de sauvegarde des bases (portabase.cineplex.fr)
│   │   ├── compose.yml
│   │   ├── secrets.env.example
│   │   └── secrets.enc.env   # PROJECT_SECRET / mot de passe postgres, chiffrés SOPS
│   ├── portabase-agent/      # Agent qui exécute les sauvegardes sur le serveur
│   │   ├── compose.yml
│   │   └── secrets.env.example   # EDGE_KEY, à chiffrer en secrets.enc.env
│   ├── seer/
│   │   └── compose.yml
│   ├── silo/
│   │   ├── compose.yml
│   │   └── secrets.enc.env   # mots de passe postgres / SECRET_KEY, chiffrés SOPS
│   ├── sure/
│   │   ├── compose.yml
│   │   └── secrets.enc.env   # mot de passe postgres / SECRET_KEY_BASE, chiffrés SOPS
│   ├── tautulli/
│   │   └── compose.yml
│   └── tracearr/
│       └── compose.yml
├── infra/                   # Briques d'infra
│   ├── doco-cd/             # Agent GitOps, bootstrap manuel une fois par serveur
│   └── traefik/             # Reverse proxy, déployé par doco-cd
│       ├── compose.yaml
│       ├── traefik.yaml          # config statique
│       ├── config/               # config dynamique (provider file)
│       ├── secrets.env.example
│       └── secrets.enc.env       # clés OVH (challenge DNS), chiffrées SOPS
├── .doco-cd.<serveur>.yml   # Liste des apps déployées sur <serveur>
└── .sops.yaml               # Règles de chiffrement des secrets
```

## Plusieurs serveurs

Chaque serveur a sa propre instance doco-cd avec un `DOCO_TARGET` (ex. `srvfr01`). Elle ne lit
que `.doco-cd.<DOCO_TARGET>.yml`. Pour ajouter un serveur : créer `.doco-cd.<nom>.yml`, puis
bootstrapper doco-cd sur la machine avec `DOCO_TARGET=<nom>`.

### App active sur un seul serveur (ex. plex)

L'app est déclarée dans les fichiers des deux serveurs, avec `destroy.enabled` comme interrupteur :

| `destroy.enabled` | Effet sur ce serveur |
|---|---|
| `false` | doco-cd déploie l'app |
| `true` | doco-cd l'arrête et la supprime (avec `remove_volumes: false`, données conservées) |

Pour basculer : passer à `true` sur le serveur actif et à `false` sur l'autre, dans le même commit.
Ne pas utiliser les `profiles` compose pour ça : sans profil actif, compose échoue avec
`no service selected`.

C'est un modèle pull : aucun serveur n'a besoin d'accès SSH ou d'un socket Docker distant sur
un autre. Les valeurs propres à un hôte (chemins, etc.) vont dans le bloc `environment:` du
fichier du serveur, pour que `apps/` reste générique.

## Secrets (SOPS + age)

Rien n'est versionné en clair. Les fichiers `*.enc.*` sont chiffrés avec SOPS et déchiffrés
par doco-cd au moment du déploiement.

```sh
brew install sops age
age-keygen -o ~/.config/sops/age/keys.txt   # garder ce fichier hors du repo, le sauvegarder
# copier la "public key: age1..." dans .sops.yaml

cp infra/traefik/secrets.env.example infra/traefik/secrets.enc.env
$EDITOR infra/traefik/secrets.enc.env       # renseigner les valeurs
sops encrypt -i infra/traefik/secrets.enc.env
sops edit infra/traefik/secrets.enc.env     # pour modifier plus tard
```

Vérifier avant chaque commit que le fichier contient bien `ENC[` et un bloc `sops_`.

## Bootstrap de doco-cd sur un serveur

1. Créer un token GitHub *fine-grained* en lecture seule : repo `homelab` uniquement,
   permission `Contents: Read-only`.
2. Sur le serveur :

   ```sh
   cd /opt/infra/doco-cd       # copie de infra/doco-cd sur le serveur
   cp .env.example .env        # régler DOCO_TARGET
   mkdir -m 700 secrets
   printf '%s' 'github_pat_...' > secrets/git_access_token
   cp /chemin/vers/keys.txt secrets/sops_age_key   # clé privée age du serveur
   # Registres privés (images ghcr.io/laulodev/…) : voir « Images privées » plus bas
   printf '{"auths":{"ghcr.io":{"auth":"%s"}}}\n' "$(printf '%s:%s' utilisateur-github ghp_... | base64 -w0)" > secrets/docker_config.json
   chmod 600 secrets/*
   docker compose up -d && docker compose logs -f
   ```

## Images privées (ghcr.io)

doco-cd télécharge les images avec les identifiants de `/root/.docker/config.json` dans son
container, fourni par le secret `secrets/docker_config.json` ([doc doco-cd](https://doco.cd/latest/Advanced/Container-Registry-Authentication/)).

1. GitHub → Settings → Developer settings → *Personal access tokens (classic)* : jeton avec
   le seul droit `read:packages`, sans expiration ou à renouveler. Le compte doit avoir accès
   aux paquets de l'organisation (ex. `laulodev`).
2. Sur le serveur, dans `/opt/infra/doco-cd` :

   ```sh
   printf '{"auths":{"ghcr.io":{"auth":"%s"}}}\n' "$(printf '%s:%s' utilisateur-github ghp_... | base64 -w0)" > secrets/docker_config.json
   chmod 600 secrets/docker_config.json
   docker compose up -d     # après avoir copié le compose.yaml à jour
   ```
3. Vérifier : `docker compose logs -f`, pas de `unauthorized` ni `pull access denied` au déploiement.

Le fichier reste sur le serveur (`infra/doco-cd/secrets/` est ignoré par git). Il contient le
jeton en clair (base64) : `chmod 600` obligatoire.

## Ajouter une app

1. `apps/<app>/compose.yaml` (images versionnées, pas de `latest`, secrets dans `secrets.enc.env`).
2. Ajouter un document dans `.doco-cd.<serveur>.yml` :

   ```yaml
   ---
   name: <app>
   working_dir: apps/<app>
   ```
3. Commit + push. Déployé au prochain poll.

Règles :
- Bind mount relatif (`./traefik.yaml`) uniquement pour de la config en lecture seule versionnée :
  chaque révision est déployée depuis sa propre copie du repo. Toute donnée écrite (base,
  certificats, uploads) va dans un volume nommé ou un chemin absolu de l'hôte.
- Retirer une app du fichier serveur ne la supprime pas : `destroy` est désactivé pour
  éviter de perdre des volumes par erreur.

## Sauvegardes (Portabase)

[Portabase](https://portabase.io) sauvegarde les bases de srvfr01. Deux projets compose, comme
dans la [doc officielle](https://portabase.io/docs/installation/docker) :

- `apps/portabase` : le dashboard et son postgres, sur `https://portabase.cineplex.fr`.
- `apps/portabase-agent` : l'agent, qui lit les bases et envoie les sauvegardes. Il est relié
  aux réseaux `silo_default` et `sure_default` et monte le dossier de données d'Actual.

Mise en service (une seule fois) :

1. Créer l'enregistrement DNS `portabase.cineplex.fr`, puis pousser : doco-cd déploie le dashboard.
2. Ouvrir le dashboard et créer le compte administrateur tout de suite (le premier compte créé).
3. **Organisation > Settings > Agents** : créer l'agent `srvfr01` et copier son Edge Key.
4. Chiffrer la clé, puis décommenter le bloc `portabase-agent` de `.doco-cd.srvfr01.yml` :

   ```sh
   cp apps/portabase-agent/secrets.env.example apps/portabase-agent/secrets.enc.env
   $EDITOR apps/portabase-agent/secrets.enc.env    # EDGE_KEY=...
   sops encrypt -i apps/portabase-agent/secrets.enc.env
   ```
5. Une fois l'agent connecté, ajouter les sources depuis sa page (**Add database**). Rien
   n'est déclaré dans le repo. Les PostgreSQL :

   | App | Hôte | Port | Base | Utilisateur | Mot de passe |
   |---|---|---|---|---|---|
   | silo | `postgres` | 5432 | `silo` | `silo` | `sops decrypt apps/silo/secrets.enc.env` |
   | sure | `db` | 5432 | `sure_production` | `sure` | `sops decrypt apps/sure/secrets.enc.env` |

   Actual Budget (dossier `/opt/apps/actual/data`, monté dans l'agent sur `/sqlite-data/actual`) :

   | Type | Chemin | Couvre |
   |---|---|---|
   | Files | `/sqlite-data/actual` | tout : comptes et budgets |
   | SQLite | `/sqlite-data/actual/server-files/account.sqlite` | comptes et mot de passe seuls |

6. Ranger chaque base dans un projet, puis régler planification, rétention et stockage.
   Le stockage local est sur srvfr01 (volume `portabase_portabase-data`) : ajouter un stockage
   externe (S3, NAS…) pour que les sauvegardes survivent à la perte du serveur.

Sauvegarder une nouvelle base : relier l'agent au réseau de l'app (postgres) ou monter son
volume ou son dossier (SQLite, fichiers) dans `apps/portabase-agent/compose.yml`, puis la
déclarer dans le dashboard. Happyplex n'est pas couvert par Portabase (il fait ses propres
sauvegardes quotidiennes dans son volume). Tracearr n'est pas couvert non plus : son
postgres est embarqué dans le conteneur et n'écoute pas sur le réseau.

## Mises à jour des images (Renovate)

[Renovate](https://docs.renovatebot.com) tourne toutes les heures dans GitHub Actions
([`.github/workflows/renovate.yml`](.github/workflows/renovate.yml)), lit tous les `compose*.yaml`
du repo et ouvre une PR par image à mettre à jour. Config : [`renovate.json`](renovate.json).
Lancement manuel : onglet Actions → renovate → Run workflow.

Le workflow utilise le secret `RENOVATE_TOKEN` : token *fine-grained* limité au repo `homelab`,
permissions `Contents`, `Pull requests`, `Issues` et `Workflows` en Read and write. À renouveler
avant son expiration.

- Patch (`v3.6.8` → `v3.6.9`) : automerge entre 3h et 7h, donc déployé par doco-cd dans la foulée.
- Mineure et majeure : PR à relire et merger à la main (une PR séparée par majeure).
- Majeures postgres : désactivées, elles demandent une migration.
- doco-cd : jamais en automerge. Après merge, sur le serveur :
  `cd /opt/infra/doco-cd && <copier compose.yaml> && docker compose up -d`.
- Vue d'ensemble : issue « Dependency Dashboard » du repo.

Rien à faire pour une nouvelle app : Renovate la détecte, à condition que le tag soit versionné
(`traefik:v3.6.8`, pas `latest`).
