# Frontend overrides (montés via `docker-compose.override.yml`)

Ces fichiers sont **bind-mountés** dans le conteneur `frontend` par
`docker-compose.override.yml`. Ils survivent aux recréations du conteneur
(`docker compose down/up`) — contrairement aux `docker cp` ponctuels.

| Fichier | Rôle |
|---|---|
| `openmrs-config.json` | Config SPA servie à `/openmrs/spa/openmrs-config.json`. **Généré** : config assemblée de l'image + fusion profonde de `frontend/configuration/config.json` (voir ci-dessous). |
| `importmap.json` | Importmap de l'image + l'entrée `path-drc-esm-orders-app`. |
| `routes.registry.json` | Routes registry de l'image + les routes de `path-drc-esm-orders-app` (copiées depuis `modules/path-drc-esm-orders-app-1.0.0/routes.json`). |
| `modules/path-drc-esm-orders-app-1.0.0/` | **App custom PATH-DRC** : dashboards d'accueil *Radiologie et imagerie* + *Procédures* (workflow de traitement des commandes comme le labo : prélever / rejeter / saisir résultats / imprimer) et onglet patient *Radiologie et Imagérie*. Source : `~/Desktop/EMR/openmrs-esm-patient-management/packages/path-drc-esm-orders-app`. Ses 3 panneaux order-basket sont retirés via config (doublons des general order types natifs). |
| `modules/openmrs-esm-patient-orders-app-12.2.1/` | Module **patché** buildé depuis le fork [`cerveaubleu64/openmrs-esm-patient-chart`](https://github.com/cerveaubleu64/openmrs-esm-patient-chart), branche `fix/general-order-priority-labels-v12.2.1` (libellés de priorité traduisibles + champ Latéralité). Voir `frontend/overrides/esm-patient-orders-app/README.md`. |

## Régénérer `openmrs-config.json`

⚠️ **Jamais de remplacement de bloc entier** : la config de l'image est la fusion
de *tous* les `frontend/configuration/*.json` (patient-chart-dashboard, VIH, etc.).
Écraser un bloc app avec celui de `config.json` seul perd ces données
(régression vécue : disparition du groupe VIH). Toujours **fusion profonde**.

```sh
docker run --rm --entrypoint cat ghcr.io/path-drc/path-drc-emr-frontend:latest \
  /usr/share/nginx/html/openmrs-config.json > /tmp/image-config.json
python3 - <<'PY'
import json
def deep_merge(base, overlay):
    for k, v in overlay.items():
        if isinstance(v, dict) and isinstance(base.get(k), dict):
            deep_merge(base[k], v)
        else:
            base[k] = v
    return base
image = json.load(open('/tmp/image-config.json'))
repo = json.load(open('frontend/configuration/config.json'))
json.dump(deep_merge(image, repo),
          open('frontend-overrides/openmrs-config.json', 'w'),
          ensure_ascii=False, indent=2)
PY
```

Le mount est lu en direct par nginx — pas de redémarrage nécessaire
(rechargement forcé du navigateur : Cmd+Shift+R).

## Régénérer `importmap.json` / `routes.registry.json`

Après un rebuild de l'image frontend (nouvelles versions d'apps) :

```sh
docker cp roland-emr-frontend-1:/usr/share/nginx/html/importmap.json frontend-overrides/importmap.json
docker cp roland-emr-frontend-1:/usr/share/nginx/html/routes.registry.json frontend-overrides/routes.registry.json
python3 - <<'PY'
import json
im = json.load(open('frontend-overrides/importmap.json'))
im['imports']['path-drc-esm-orders-app'] = './path-drc-esm-orders-app-1.0.0/path-drc-esm-orders-app.js'
json.dump(im, open('frontend-overrides/importmap.json', 'w'), indent=2)
r = json.load(open('frontend-overrides/routes.registry.json'))
r['path-drc-esm-orders-app'] = json.load(open('frontend-overrides/modules/path-drc-esm-orders-app-1.0.0/routes.json'))
json.dump(r, open('frontend-overrides/routes.registry.json', 'w'), indent=2)
PY
```

⚠️ Attention au chemin du module orders-app patché : si la version assemblée
change (ex. 12.3.0), rebuilder le fork sur le nouveau tag et adapter le nom du
dossier + le mount dans `docker-compose.override.yml`.

## Config associée (dans `frontend/configuration/config.json`)

- `path-drc-esm-orders-app` : UUIDs des order types / concept sets EMR2
  (imaging `9c786025…`/`319d8dfc…`, procedure `89b2923d…`/`c789511c…`,
  supply `3fefd401…`/`fbaf78be…`)
- `@openmrs/esm-patient-orders-app.extensionSlots.order-basket-slot.remove` :
  retire les 3 panneaux basket de l'app custom (doublons)
- `@openmrs/esm-home-app.extensionSlots.homepage-dashboard-slot.order` :
  ordre du menu d'accueil (… Laboratoire → Radiologie et imagerie → Procédures → Facturation)
