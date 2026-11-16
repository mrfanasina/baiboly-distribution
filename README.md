# Baiboly — distribution (mises à jour et téléchargements)

Ce dépôt ne contient **pas** l'APK. Il sert uniquement à publier les fichiers que
l'application lit à distance :

| Fichier | Rôle | Lien à mettre dans l'app |
| --- | --- | --- |
| `update.json` | dernière version de l'APK + notes de mise à jour | fixe, ne change jamais |
| `downloads.json` | catalogue des contenus téléchargeables (audio, vidéo, BD, JSON de contenu) | fixe, ne change jamais |

Règle d'or : **on ne change jamais le nom de ces fichiers ni leur emplacement**.
Le contenu évolue, l'URL reste identique. L'app peut donc coder ces deux URL en dur.

---

## 1. Règles communes

- Encodage **UTF-8 sans BOM**, JSON valide (pas de commentaires, pas de virgule finale).
- Toutes les URL doivent être en **HTTPS** et pointer directement sur le fichier.
- Dates au format ISO 8601 : `AAAA-MM-JJ` (`2026-11-16`).
- Les identifiants (`id`, `version_code`) sont des chaînes/entiers **stables** : on n'en réutilise jamais un.
- L'app **ignore les champs inconnus** et **tolère les champs optionnels absents**.
- `schema_version` indique la version du format. Si l'app ne connaît pas cette valeur,
  elle doit refuser le fichier plutôt que de deviner.
- Après chaque modification, vérifier que le fichier reste valide :
  `python3 -m json.tool update.json` et `python3 -m json.tool downloads.json`.

---

## 2. `update.json` — mises à jour de l'application

| Champ | Type | Requis | Description |
| --- | --- | --- | --- |
| `schema_version` | entier | oui | version du format, actuellement `1` |
| `version_code` | entier | oui | `versionCode` Android ; sert à comparer avec l'app installée |
| `version_name` | chaîne | oui | nom lisible de la version (`"4.0"`) |
| `apk_url` | chaîne | oui | URL HTTPS directe de l'APK |
| `file_size` | entier | non | taille de l'APK en octets |
| `sha256` | chaîne | non | empreinte SHA-256 de l'APK (hexadécimal minuscule) |
| `published_at` | chaîne | non | date de publication |
| `forced` | booléen | non | `true` = mise à jour imposée (défaut `false`) |
| `min_supported_version_code` | entier | non | en dessous, l'app doit exiger la mise à jour |
| `changelog` | tableau de chaînes | non | notes affichées à l'utilisateur |

Règle de décision côté app :
`mise à jour disponible` si `version_code > versionCode installé`.
`mise à jour obligatoire` si `forced == true` ou `versionCode installé < min_supported_version_code`.

```json
{
  "schema_version": 1,
  "version_code": 4,
  "version_name": "4.0",
  "apk_url": "https://exemple.com/app-debug.apk",
  "file_size": 33469634,
  "sha256": "750ee87a941795d0fa23dfc25a2f93304a2018903a346767d4590fcc47c65499",
  "published_at": "2026-11-15",
  "forced": false,
  "min_supported_version_code": 1,
  "changelog": ["Ajout de programme liturgique personnalisable."]
}
```

---

## 3. `downloads.json` — contenus téléchargeables

Trois niveaux : `downloads.json` → catégories → éléments.

### Racine

| Champ | Type | Requis | Description |
| --- | --- | --- | --- |
| `schema_version` | entier | oui | version du format, actuellement `1` |
| `updated_at` | chaîne | non | date de dernière mise à jour du catalogue |
| `categories` | tableau | oui | liste des catégories (voir ci-dessous) |

### Catégorie (`categories[]`)

| Champ | Type | Requis | Description |
| --- | --- | --- | --- |
| `id` | chaîne | oui | identifiant unique, slug (`audio`, `video`, `bd`, `contenu`) |
| `title` | chaîne | oui | libellé affiché |
| `icon_url` | chaîne | non | icône de la catégorie |
| `order` | entier | non | ordre d'affichage croissant |
| `items` | tableau | oui | éléments de la catégorie (peut être vide) |

### Élément (`categories[].items[]`)

| Champ | Type | Requis | Description |
| --- | --- | --- | --- |
| `id` | chaîne | oui | identifiant **unique dans tout le fichier** (ex. `audio-baiboly-mg`) |
| `title` | chaîne | oui | libellé affiché |
| `description` | chaîne | non | texte court |
| `type` | chaîne | oui | `audio`, `video`, `bd`, `document`, `json` |
| `url` | chaîne | oui | URL HTTPS directe du fichier |
| `mime_type` | chaîne | non | type MIME (`audio/mpeg`, `video/mp4`, `application/pdf`, `application/json`) |
| `file_name` | chaîne | non | nom proposé au téléchargement |
| `file_size` | entier | non | taille en octets |
| `sha256` | chaîne | non | empreinte SHA-256 du fichier |
| `version` | entier | non | version du contenu, incrémentée à chaque remplacement (défaut `1`) |
| `published_at` | chaîne | non | date de publication |
| `cover_url` | chaîne | non | vignette / pochette |
| `language` | chaîne | non | code langue (`mg`, `fr`, …) |
| `enabled` | booléen | non | `false` = masqué dans l'app (défaut `true`) |

Règles côté app :
- afficher uniquement les éléments `enabled != false` ;
- considérer un contenu comme « à mettre à jour » si le `version` local est inférieur
  au `version` distant, ou si le `sha256` local diffère ;
- ne remplacer un fichier local qu'après vérification du `sha256` (si fourni) ;
- si `type == "json"`, traiter `url` comme un **contenu structuré** décrit en section 4.

Exemple complet (structure identique au fichier de ce dépôt) :

```json
{
  "schema_version": 1,
  "updated_at": "2026-11-16",
  "categories": [
    {
      "id": "audio",
      "title": "Audio",
      "order": 1,
      "items": [
        {
          "id": "audio-baiboly-mg",
          "title": "Baiboly audio (malagasy)",
          "type": "audio",
          "url": "https://exemple.com/downloads/baiboly-audio.mp3",
          "mime_type": "audio/mpeg",
          "file_size": 0,
          "sha256": "",
          "version": 1,
          "published_at": "2026-11-16",
          "language": "mg",
          "enabled": true
        }
      ]
    }
  ]
}
```

---

## 4. Contenu JSON reconnu par l'application (`type: "json"`)

Un élément de type `json` pointe vers un fichier de données que l'app charge
directement. Ce fichier a **sa propre structure**, volontairement enveloppée pour
que l'app sache quoi en faire sans deviner.

### Enveloppe obligatoire

| Champ | Type | Requis | Description |
| --- | --- | --- | --- |
| `schema_version` | entier | oui | version du format de contenu (actuellement `1`) |
| `id` | chaîne | oui | identifiant du contenu, identique au `id` déclaré dans `downloads.json` |
| `type` | chaîne | oui | type de contenu : `programme_liturgique`, `playlist`, `texte`… |
| `version` | entier | oui | version du contenu, à incrémenter à chaque modification |
| `updated_at` | chaîne | non | date de mise à jour |
| `title` | chaîne | non | titre affiché |
| `language` | chaîne | non | code langue |
| `data` | objet | oui | données propres au `type` (voir ci-dessous) |

Contrat côté app :
- vérifier `schema_version` **avant** de lire `data` ; refuser les valeurs inconnues ;
- router sur `type` : un `type` inconnu est ignoré, pas une erreur fatale ;
- conserver la copie locale et ne la remplacer que si `version` distante > locale ;
- ignorer les champs non documentés pour rester compatible avec les ajouts futurs.

### Exemple : `type: "programme_liturgique"` (version 4)

```json
{
  "schema_version": 1,
  "id": "contenu-programme-liturgique",
  "type": "programme_liturgique",
  "version": 1,
  "updated_at": "2026-11-16",
  "title": "Programme liturgique 2026",
  "language": "mg",
  "data": {
    "annee": 2026,
    "programmes": [
      {
        "id": "lit-2026-11-22",
        "date": "2026-11-22",
        "titre": "Fetin'ny Kristy Mpanjaka",
        "lectures": [
          {
            "ordre": 1,
            "reference": "Jeremia 23:1-6",
            "texte": ""
          }
        ],
        "chants": ["..."]
      }
    ]
  }
}
```

`data.annee` (entier) et `data.programmes` (tableau) sont obligatoires pour ce type.
Chaque programme expose `id`, `date`, `titre`, `lectures[]` et `chants[]`.
Les champs `texte` et `chants` peuvent être vides : l'app doit alors utiliser
le contenu local.

> Les autres `type` (`playlist`, `texte`, …) suivent le même principe : enveloppe
> identique, et `data` documenté au moment où le type est ajouté. Tout changement
> incompatible impose d'incrémenter `schema_version`.

---

## 5. Où héberger ces fichiers

L'app télécharge des fichiers statiques : aucun serveur applicatif n'est nécessaire.

### Option A — GitHub Releases + raw (gratuit, lien fixe)

1. Publier l'APK dans une release avec un tag stable :
   `gh release create v4 app-debug.apk --title "Version 4"`
2. URL de l'APK : `https://github.com/<user>/<repo>/releases/download/v4/app-debug.apk`
   (garder le même tag et le même nom de fichier d'une version à l'autre, et
   remplacer l'asset ; ou utiliser `.../releases/latest/download/app-debug.apk`).
3. URL des manifests : `https://raw.githubusercontent.com/<user>/<repo>/main/update.json`
   et `.../main/downloads.json`. Ces liens restent fixes tant que la branche
   `main` et les noms de fichiers ne changent pas.
4. Limite : 2 Go par asset, largement suffisant.

### Option B — Cloudflare Pages / Netlify (glisser-déposer)

Publier `update.json`, `downloads.json`, l'APK et les médias dans le dossier du site :
`https://mon-site.pages.dev/update.json`, `.../app-debug.apk`. CDN + HTTPS gratuits,
et possibilité d'utiliser un domaine personnel.

### Option C — VPS nginx

Placer les fichiers dans `/var/www/baiboly/` et servir ce dossier. Prévoir
`client_max_body_size 50m;` pour les vidéos, et servir les APK avec
`application/vnd.android.package-archive`. Plus de maintenance (TLS, sauvegardes).

### Option D — Firebase App Distribution / Google Play

Utile pour des testeurs, mais ne fournit pas un simple fichier public à URL fixe :
moins adapté à ce dépôt.

---

## 6. Publier une nouvelle version (checklist)

1. Construire l'APK et noter son `versionCode`.
2. Calculer la taille et l'empreinte : `stat -c '%s' app.apk` et `sha256sum app.apk`.
3. Mettre à jour `update.json` : `version_code`, `version_name`, `apk_url`,
   `file_size`, `sha256`, `published_at`, `changelog`.
4. Publier l'APK à l'URL déclarée (même nom de fichier qu'avant).
5. Pour un contenu : ajouter/modifier l'élément dans `downloads.json`, incrémenter
   son `version`, mettre à jour `sha256`, puis `updated_at` du catalogue.
6. Vérifier le JSON (`python3 -m json.tool`) et ouvrir les URL dans un navigateur
   pour confirmer qu'elles répondent bien en HTTPS.
7. Ne jamais changer les liens de `update.json` et `downloads.json` : l'app les a
   en dur.
