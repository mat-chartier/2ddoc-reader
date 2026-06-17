# Lecteur 2D-DOC V2 (ISO 22376:2023)

Lecteur **100 % navigateur** de codes 2D-DOC V2 (norme ISO 22376:2023, génération France Titres / ANTS). Il scanne un QR, décode l'en-tête et le payload MessagePack, résout en ligne le manifest et le certificat dans la chaîne de confiance de l'ANTS, puis vérifie la signature ECDSA P-256 — le tout dans une seule page HTML statique, sans backend.

**Démo en ligne : https://mat-chartier.github.io/2ddoc-reader/**

> ⚠️ Outil indépendant, à but de test et de compréhension du format. Ce n'est pas un produit officiel de l'ANTS. Les données de qualification publiées par l'ANTS sont marquées « POUR INTÉGRATION UNIQUEMENT » et ne constituent pas de vrais titres.

## Fonctionnalités

- **Plusieurs entrées** : caméra (scan continu), photo, glisser-déposer ou import de fichier image, et collage d'une chaîne 2D-DOC brute.
- **Décodage de l'en-tête** : marqueur `0xDE`, version, identifiant du signataire (C40), identifiant de manifest, horodatage, longueur du payload.
- **Décodage du payload** en flux MessagePack avec prise en charge complète des types : chaînes, dates, entiers (jusqu'à 64 bits), flottants, booléens, **binaires**, **timestamps**, objets, et tous les tableaux.
- **Résolution en ligne** du manifest (schéma des champs) et du certificat de signature dans la chaîne de confiance ANTS, avec essai de la **production d'abord** puis repli automatique sur la **qualification**.
- **Vérification de signature** ECDSA P-256 selon le schéma 2D-DOC V2 : `ECDSA( SHA-256( en-tête ‖ SHA-256(payload) ) )`, avec la clé publique extraite du certificat téléchargé.
- **Rendu lisible** : chaque champ est affiché avec son libellé, son type et, le cas échéant, le label issu des dictionnaires du manifest.
- **Images embarquées** : les champs binaires reconnus comme images (WEBP, PNG, JPEG, GIF) sont affichés directement.
- **Affichage dégradé** : si le manifest et/ou le certificat sont introuvables (réseau, manifest non encore publié…), le contenu du QR reste affiché en valeurs brutes, avec un avertissement clair et le statut de chaque maillon de la chaîne.

## Utilisation

Ouvrez simplement `index.html` dans un navigateur récent, ou utilisez la [démo en ligne](https://mat-chartier.github.io/2ddoc-reader/).

Deux prérequis pour les fonctions natives :

- **HTTPS** est nécessaire pour accéder à la caméra et à l'API `crypto.subtle` (la vérification de signature). GitHub Pages le fournit ; en local, `http://localhost` convient aussi.
- Un **proxy CORS** est nécessaire pour récupérer manifest et certificat (voir ci-dessous).

## Configuration

Deux constantes en tête du `<script>` de `index.html` :

```js
// Environnements de résolution, essayés DANS L'ORDRE : prod d'abord, qualif en repli.
var ENVIRONMENTS = [
  { name: 'production',    base: 'https://pub.ants.gouv.fr/2D-DOC/V2/PRD' },
  { name: 'qualification', base: 'https://pub.ants.gouv.fr/2D-DOC/V2/QLF'  }
];

// Préfixe d'un proxy CORS (voir section ci-dessous). Laisser '' pour un accès direct.
var FETCH_PREFIX = 'https://<votre-proxy>/?url=';
```

### Pourquoi un proxy CORS ?

`pub.ants.gouv.fr` sert bien les manifests et certificats, mais **sans en-tête `Access-Control-Allow-Origin`**. Un navigateur bloque donc la lecture de ces réponses par le JavaScript d'une autre origine (par exemple depuis GitHub Pages). Le proxy, exécuté côté serveur (où CORS ne s'applique pas), récupère la ressource et la réémet avec l'en-tête CORS attendu.

N'importe quel relais convient, à condition de le restreindre à `pub.ants.gouv.fr/2D-DOC/` pour ne pas créer un proxy ouvert. Un Cloudflare Worker minimal fait très bien l'affaire ; renseignez ensuite son URL dans `FETCH_PREFIX` (forme `https://<worker>/?url=`).

## Chaîne de confiance (résolution)

À partir du préfixe et de l'en-tête, le lecteur reconstitue le chemin :

```
PREFIX (ex. KFRMI6)  →  IAC / CIN / formatCode
  → LOTL Racine        00_LOTL_RACINE/
  → LOTL OSC            01_LOTLs/<IAC>/
  → TSL                 02_TSLs/<IAC>/<CIN>/<format>/
  → Certificat          03_CERTIFICATS/.../<authorityId>/<certificateId>.cer   (clé publique de vérification)
  → Manifest            04_MANIFESTS/.../<manifestId>.xml                       (schéma du payload)
```

Le certificat et le manifest sont cherchés en production puis en qualification ; un badge indique pour chacun l'environnement d'où il provient, ou s'il est introuvable.

## Format du payload

Le payload est un **flux MessagePack** : les valeurs des champs de tête sont concaténées, **sans tableau racine englobant** (conforme au spécimen normatif de la spécification ANTS V2). Les objets et tableaux d'objets imbriqués, eux, sont bien des tableaux MessagePack de taille fixe.

## Dépendances

Chargées depuis un CDN (aucune installation) :

- [`jsQR`](https://github.com/cozmo/jsQR) — détection du QR dans l'image.
- [`elliptic`](https://github.com/indutny/elliptic) — vérification ECDSA P-256.

## Données de test

Le jeu de données officiel de l'ANTS (JDD) fournit des vecteurs signés et conformes pour chaque manifest de test. Les cas « cas 1 » sont signés par le certificat de test `FR99/TEST` (publié), donc vérifiables de bout en bout par le lecteur.

## Limitations connues

- Outil de test/diagnostic : ne remplace pas un lecteur certifié.
- La résolution en ligne dépend de la disponibilité des ressources ANTS et du proxy CORS.
- La détection d'image embarquée couvre WEBP, PNG, JPEG et GIF.

## Licence

À définir par l'auteur du dépôt (par exemple MIT).
