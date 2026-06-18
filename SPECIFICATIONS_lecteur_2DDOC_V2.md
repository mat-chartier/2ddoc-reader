# Spécifications du lecteur 2D-DOC V2 (ISO 22376:2023)

Version du document : 1.0 — Spécifications fonctionnelles détaillées (SFD) et techniques détaillées (STD).

Objet : décrire de manière exhaustive et non ambiguë le comportement et les algorithmes d'un lecteur 2D-DOC V2 (format France Titres / ANTS, conforme ISO 22376:2023), afin de permettre une implémentation correcte sans interprétation. Le présent document ne contient pas de code, mais détaille chaque algorithme au niveau des octets, formules et étapes.

---

# Partie A — Spécifications fonctionnelles détaillées (SFD)

## A.1 Périmètre

Le lecteur est une application **entièrement côté client** (navigateur), sans serveur applicatif. Il prend en entrée un code 2D-DOC V2 (depuis une caméra, une image ou une chaîne texte), en décode le contenu, résout en ligne le schéma (manifest) et le certificat de signature auprès de l'ANTS, vérifie la signature, et affiche le résultat de façon lisible.

## A.2 Entrées utilisateur

Le lecteur doit accepter les quatre sources suivantes, qui aboutissent toutes au même traitement à partir de la chaîne 2D-DOC brute :

1. **Caméra** : flux vidéo (caméra arrière par défaut), scan en continu jusqu'à détection d'un QR.
2. **Photo** : capture d'une image depuis la caméra.
3. **Fichier image** : import ou glisser-déposer d'un fichier (PNG, JPEG, etc.).
4. **Texte collé** : collage direct de la chaîne 2D-DOC (commençant par le préfixe).

Pour les sources image (1, 2, 3), la détection du QR fournit une chaîne de caractères ; le traitement est ensuite identique à la source 4.

## A.3 Sorties affichées

Pour un code valide, le lecteur affiche :

1. La **chaîne brute** lue.
2. L'**en-tête décodé** : identifiant du signataire (IAC, autorité, certificat), identifiant de manifest, horodatage, longueur du payload, et le détail hexadécimal.
3. Le **payload décodé** : chaque champ avec son libellé (issu du manifest), sa valeur formatée selon son type, et le label de dictionnaire le cas échéant. Les images embarquées sont affichées en tant qu'images.
4. Le **statut de signature** : valide / invalide / non vérifiée.
5. La **chaîne de confiance** : pour le manifest et le certificat, l'URL résolue et l'environnement de provenance (production ou qualification), ou l'indication « introuvable ».

## A.4 Comportements de robustesse (dégradation gracieuse)

Exigence forte : **le contenu du QR doit toujours être affiché**, même si la résolution en ligne échoue partiellement ou totalement.

- La récupération du manifest et celle du certificat sont **indépendantes** : l'échec de l'une ne doit pas empêcher l'autre ni l'affichage du contenu.
- Si le **manifest est introuvable** : le payload est affiché en **valeurs brutes** (numéro de champ + type MessagePack + valeur), sans libellés métier.
- Si le **certificat est introuvable** : la signature est marquée **« non vérifiée »** (et non « invalide »).
- Le statut de signature a donc **trois états** : valide, invalide, non vérifiée. Une signature « invalide » (certificat présent mais signature qui ne correspond pas) est un échec réel et doit rester distincte d'une signature « non vérifiée » (impossible à contrôler).
- Un **bandeau d'avertissement** liste explicitement ce qui manque.
- Les conversions de date et d'horodatage ne doivent **jamais** provoquer d'erreur fatale sur une valeur hors borne : afficher « (date invalide) » plutôt que planter.
- Le lecteur ne doit jamais rester bloqué sur un indicateur de chargement : toute erreur inattendue aboutit à un affichage exploitable.

## A.5 Gestion des erreurs

- Marqueur d'en-tête absent ou incorrect → message clair « ce n'est pas un 2D-DOC V2 ».
- Caractère invalide en Base45 → message d'erreur indiquant la position.
- Incohérence de longueur → avertissement, mais tentative d'affichage du contenu extractible.
- Échec réseau / CORS sur la résolution → message distinguant un problème réseau d'un simple « introuvable » (404).

## A.6 Configuration

Les éléments suivants sont des **constantes de configuration** isolées (modifiables sans toucher à la logique) :

- la liste ordonnée des environnements de résolution (production puis qualification) ;
- le préfixe du proxy CORS ;
- l'époque de référence des dates (1900-01-01) ;
- le préfixe SPKI de la clé EC ;
- le suffixe de chemin de chaîne associé au code format.

Aucune donnée métier (manifest, clé publique) ne doit être codée en dur dans l'application.

## A.7 Contraintes non fonctionnelles

- Livrable : **un seul fichier HTML autonome**.
- HTTPS obligatoire en production (requis pour l'accès caméra et l'API de cryptographie du navigateur).
- Interface responsive, lisible sur mobile.
- Dépendances externes limitées et chargées par CDN (voir STD).

---

# Partie B — Spécifications techniques détaillées (STD)

## B.1 Structure générale d'un 2D-DOC V2

La chaîne lue se compose de :

```
[ PRÉFIXE : 6 caractères ASCII ] [ CORPS : chaîne Base45 ]
```

Le préfixe **n'est pas** encodé en Base45 ; il est lu tel quel. Le corps, une fois décodé en Base45, donne une séquence d'octets structurée ainsi :

```
[ EN-TÊTE : 19 octets ] [ PAYLOAD : N octets ] [ SIGNATURE : 64 octets ]
```

où N est la longueur indiquée dans l'en-tête. La longueur totale décodée doit valoir `19 + N + 64`.

## B.2 Préfixe (6 caractères)

Le préfixe est composé de :

| Position | Longueur | Champ        | Exemple |
|----------|----------|--------------|---------|
| 0–2      | 3        | IAC          | `KFR`   |
| 3–4      | 2        | CIN          | `MI`    |
| 5        | 1        | Code format  | `6`     |

Le code format `6` désigne ISO 22376. Le préfixe sert à construire le **chemin de chaîne** utilisé pour la résolution (voir B.9) :

```
cheminChaîne = IAC + "/" + CIN + "/" + codeFormat + "_ISO22376"
ex. : "KFR/MI/6_ISO22376"
```

## B.3 Décodage Base45 (RFC 9285) — algorithme exact

Le Base45 **n'est pas** une base binaire (ni Base32, ni Base64). C'est de l'arithmétique en base 45 par groupes. Toute implémentation par accumulation de bits est incorrecte.

**Alphabet** (l'index d'un caractère est sa position, de 0 à 44) :

```
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ $%*+-./:
```

(soit : les chiffres `0`–`9` = 0–9, les lettres `A`–`Z` = 10–35, l'espace = 36, puis `$ % * + - . / :` = 37–44).

**Règles de décodage :**

1. La longueur de la chaîne Base45 modulo 3 doit valoir **0 ou 2**. Un reste de 1 est invalide.
2. Pour chaque groupe complet de **3 caractères** `c0 c1 c2` (où `ci` désigne l'index dans l'alphabet) :
   - calculer `n = c0 + c1 × 45 + c2 × (45 × 45)` ;
   - `n` tient sur 16 bits (0–65535 pour une entrée valide) ;
   - produire **2 octets** : l'octet de poids fort `partie entière de (n / 256)`, puis l'octet de poids faible `n modulo 256`.
3. Pour un dernier groupe de **2 caractères** `c0 c1` :
   - calculer `n = c0 + c1 × 45` ;
   - produire **1 octet** : `n modulo 256`.

Point critique : **le premier caractère du groupe est le poids le plus faible** (`c0`), pas le plus fort.

**Vecteur de contrôle** : sur le corps du QR de test du manifest `000000` (préfixe `KFRMI6`, corps commençant par `03S.FJJSF…`), les premiers octets décodés doivent être `DE 03 99 18 7B A6 …`. Si le premier octet n'est pas `0xDE`, le décodage Base45 est faux.

## B.4 En-tête (19 octets) — disposition exacte

| Offset | Longueur | Champ                | Encodage / valeur                                      |
|--------|----------|----------------------|--------------------------------------------------------|
| 0      | 1        | Marqueur             | constante `0xDE` (sinon : pas un 2D-DOC V2)            |
| 1      | 1        | Version              | constante `0x03` (2D-DOC V2)                           |
| 2–9    | 8        | Bloc signataire      | C40 → 12 caractères (voir B.5)                         |
| 10–12  | 3        | Identifiant manifest | 3 octets affichés en 6 chiffres hexadécimaux           |
| 13–16  | 4        | Horodatage           | entier non signé 32 bits, gros-boutiste, secondes Unix |
| 17–18  | 2        | Longueur du payload  | entier non signé 16 bits, gros-boutiste                |

Notes :
- « gros-boutiste » signifie : valeur = `octet_de_poids_fort × 256^k + …`. Pour 2 octets : `octet[17] × 256 + octet[18]`. Pour 4 octets : `((o0 × 256 + o1) × 256 + o2) × 256 + o3`.
- L'identifiant manifest se présente en hexadécimal : par exemple les octets `00 05 00` donnent la chaîne `"000500"`.
- L'horodatage est un entier brut (il n'est **pas** encodé en MessagePack ; c'est l'horodatage de génération du code).

## B.5 Décodage C40 (ISO/IEC 16022) — algorithme exact

Le bloc signataire (8 octets) est encodé en **C40**, qui regroupe **3 symboles pour 2 octets**. Toute implémentation par accumulation de 5 bits (logique Base32) est incorrecte.

**Pour chaque paire d'octets** `(b1, b2)` du bloc :

1. calculer `V = b1 × 256 + b2` ;
2. calculer `I = V − 1` ;
3. extraire trois valeurs de symbole :
   - `U1 = partie entière de (I / 1600)`
   - `U2 = partie entière de ((I modulo 1600) / 40)`
   - `U3 = I modulo 40`

**Traduction de chaque valeur de symbole** `U` (jeu de base C40) :

| Valeur `U` | Caractère                                   |
|------------|---------------------------------------------|
| 0, 1, 2    | codes de bascule (shift) — voir note        |
| 3          | espace `' '`                                |
| 4 à 13     | chiffres `'0'` à `'9'` (4 → `'0'`)           |
| 14 à 39    | lettres `'A'` à `'Z'` (14 → `'A'`)          |

Note sur les shifts : les valeurs 0, 1, 2 commutent vers d'autres jeux de caractères (minuscules, ponctuation). Pour le bloc signataire 2D-DOC, seuls espace, chiffres et lettres apparaissent ; les shifts ne se produisent pas. Une implémentation complète conforme devrait néanmoins les gérer ; pour ce lecteur, le jeu de base suffit (à documenter comme limitation connue).

**Résultat** : 8 octets → 4 paires → **12 caractères**, découpés ainsi :

| Caractères | Champ            | Remarque                                  |
|------------|------------------|-------------------------------------------|
| 0–2        | IAC              | ex. `KFR`                                 |
| 3–6        | authorityId      | ex. `FR99` (retirer les `'0'` de fin)     |
| 7–10       | certificateId    | ex. `TEST`, `AM01` (retirer les `'0'` de fin) |
| 11         | Caractère de remplissage | ignoré                            |

**Vecteur de contrôle** : le bloc `99 18 7B A6 56 7B CD 2D` doit donner `"KFRFR99TEST0"`, soit IAC `KFR`, authorityId `FR99`, certificateId `TEST`.

## B.6 Payload — cadrage et décodage MessagePack

### B.6.1 Cadrage : flux de valeurs, sans tableau racine

Le payload est un **flux MessagePack** : les valeurs des champs racine du manifest sont **concaténées les unes à la suite des autres**, dans l'ordre des champs, **sans tableau MessagePack englobant**. Le décodage consiste donc à lire des valeurs MessagePack successives depuis l'offset 0 jusqu'à épuisement des N octets, et à les ranger dans une liste (une valeur par champ racine).

En revanche, les **objets et tableaux d'objets imbriqués** sont, eux, bien des tableaux MessagePack (voir B.6.3).

Tolérance recommandée : si la liste obtenue contient **exactement une** valeur, que cette valeur est un tableau, et que sa taille est égale au nombre de champs racine du manifest, alors traiter les éléments de ce tableau comme les valeurs des champs (compatibilité avec un producteur non conforme qui aurait enveloppé le payload dans un tableau racine).

### B.6.2 Table de décodage MessagePack (marqueur → valeur, et nombre d'octets consommés)

Le premier octet (marqueur) détermine le type. `BE` = lecture gros-boutiste.

| Marqueur (1er octet) | Type            | Décodage                                                                 |
|----------------------|-----------------|--------------------------------------------------------------------------|
| `0x00`–`0x7F`        | entier positif fixe | valeur = l'octet lui-même                                            |
| `0xE0`–`0xFF`        | entier négatif fixe | valeur = octet − 256                                                 |
| `0xC0`               | nil             | valeur = absente / nulle                                                 |
| `0xC2` / `0xC3`      | booléen         | `false` / `true`                                                         |
| `0xCC`               | uint8           | 1 octet suivant                                                          |
| `0xCD`               | uint16          | 2 octets suivants, BE                                                    |
| `0xCE`               | uint32          | 4 octets suivants, BE                                                    |
| `0xCF`               | uint64          | 8 octets suivants, BE (précision au-delà de 2^53 : approchée)            |
| `0xD0`               | int8            | 1 octet, signé                                                           |
| `0xD1`               | int16           | 2 octets, BE, signé                                                      |
| `0xD2`               | int32           | 4 octets, BE, signé                                                      |
| `0xD3`               | int64           | 8 octets, BE, signé                                                      |
| `0xCA`               | float32         | 4 octets, IEEE 754, BE                                                   |
| `0xCB`               | float64         | 8 octets, IEEE 754, BE                                                   |
| `0xA0`–`0xBF`        | fixstr          | longueur = 5 bits de poids faible du marqueur ; puis cette longueur en octets UTF-8 |
| `0xD9`               | str8            | longueur sur 1 octet, puis UTF-8                                         |
| `0xDA`               | str16           | longueur sur 2 octets BE, puis UTF-8                                     |
| `0xDB`               | str32           | longueur sur 4 octets BE, puis UTF-8                                     |
| `0xC4`               | bin8            | longueur sur 1 octet, puis ces octets bruts                             |
| `0xC5`               | bin16           | longueur sur 2 octets BE, puis ces octets bruts                         |
| `0xC6`               | bin32           | longueur sur 4 octets BE, puis ces octets bruts                         |
| `0x90`–`0x9F`        | fixarray        | nombre d'éléments = 4 bits de poids faible ; décoder récursivement ce nombre de valeurs |
| `0xDC`               | array16         | nombre sur 2 octets BE, puis les éléments                               |
| `0xDD`               | array32         | nombre sur 4 octets BE, puis les éléments                               |
| `0x80`–`0x8F`        | fixmap          | nombre de paires = 4 bits de poids faible ; chaque paire = clé + valeur |
| `0xDE`               | map16           | nombre de paires sur 2 octets BE                                        |
| `0xDF`               | map32           | nombre de paires sur 4 octets BE                                        |
| `0xD4`               | fixext1         | extension : 1 octet de type, 1 octet de données                         |
| `0xD5`               | fixext2         | 1 octet de type, 2 octets de données                                    |
| `0xD6`               | fixext4         | 1 octet de type, 4 octets de données                                    |
| `0xD7`               | fixext8         | 1 octet de type, 8 octets de données                                    |
| `0xD8`               | fixext16        | 1 octet de type, 16 octets de données                                   |
| `0xC7`               | ext8            | 1 octet de longueur, 1 octet de type, puis les données                  |
| `0xC8`               | ext16           | 2 octets de longueur BE, 1 octet de type, puis les données              |
| `0xC9`               | ext32           | 4 octets de longueur BE, 1 octet de type, puis les données              |

Important : chaque décodage doit renvoyer, en plus de la valeur, l'**offset de fin** (position de l'octet suivant), pour permettre la lecture en flux.

### B.6.3 Extension Timestamp (type d'extension = `0xFF`, soit −1)

Pour les extensions dont l'octet de type vaut `0xFF` :

- **données sur 4 octets** : secondes Unix = entier non signé 32 bits BE.
- **données sur 8 octets** : valeur 64 bits BE ; les **34 bits de poids faible** sont les secondes, les 30 bits de poids fort sont les nanosecondes. (Calcul des secondes : `(les 2 bits de poids faible des 4 premiers octets) × 2^32 + (les 4 octets suivants)`.)
- **données sur 12 octets** : 4 octets de nanosecondes (BE) puis 8 octets de secondes (BE).

Pour l'affichage, seules les secondes sont nécessaires.

### B.6.4 Correspondance type de champ (manifest) → MessagePack et rendu

| Type de champ (manifest) | Encodage MessagePack                | Rendu attendu                                             |
|--------------------------|-------------------------------------|-----------------------------------------------------------|
| String                   | chaîne (ou nil si absent)           | texte ; distinguer chaîne vide et nil                     |
| Date                     | entier = nb de jours depuis 1900-01-01 | date convertie (voir B.7), format `AAAA-MM-JJ`         |
| Integer                  | entier                              | nombre                                                    |
| Float                    | flottant                            | nombre                                                    |
| Boolean                  | booléen                             | « Oui » / « Non »                                         |
| Binary                   | binaire                             | image si reconnue (voir B.10), sinon « N octets »         |
| Timestamp                | extension Timestamp                 | date-heure                                                |
| Object                   | **tableau** des valeurs des sous-champs, dans l'ordre du type | rendu récursif des sous-champs            |
| *Array (String/Date/Integer/Float/Boolean/Binary/Timestamp/Object) | **tableau** dont chaque élément suit le type de base | liste de valeurs rendues selon le type de base |

Règles transverses :
- Les champs « nillable » absents sont encodés par le nil MessagePack (`0xC0`).
- Un champ peut référencer un **dictionnaire** : la valeur stockée (chaîne ou entier) est une **clé** dont le libellé lisible figure dans le dictionnaire du manifest. Afficher la clé et son libellé.

## B.7 Conversion des dates

Le type Date est un entier `D` = nombre de jours depuis le **1900-01-01 (UTC)**.

- date = 1900-01-01 + `D` jours.
- Implémentation robuste : si `D` n'est pas un nombre fini, ou si la date résultante est hors plage représentable, renvoyer « (date invalide) » au lieu de provoquer une erreur.

## B.8 Signature (ECDSA P-256) — algorithme exact

- La **signature** est constituée des **64 derniers octets** de la séquence décodée : `r` (32 octets, BE) concaténé à `s` (32 octets, BE).
- Le **condensé signé** se calcule en **double passe** :
  1. `h1 = SHA-256(payload)` (les N octets du payload) ;
  2. `condenséFinal = SHA-256( en-tête (19 octets) ‖ h1 )`.
- La vérification ECDSA P-256 porte sur `condenséFinal` **tel quel**, sans hachage supplémentaire (sémantique « NONEwithECDSA » : on vérifie `(r, s)` contre un condensé déjà calculé).

Point d'attention critique sur l'implémentation : l'API de cryptographie native du navigateur (`crypto.subtle.verify` pour ECDSA) **re-hache** le message avec la fonction de hachage indiquée ; elle **ne peut donc pas** vérifier directement un condensé déjà calculé. Il faut utiliser une bibliothèque ECDSA capable de vérifier `(r, s)` contre un condensé fourni (par exemple une bibliothèque de courbes elliptiques). En revanche, `crypto.subtle.digest('SHA-256', …)` convient pour calculer les deux SHA-256 (en gardant à l'esprit qu'il est **asynchrone** : attendre le résultat avant de l'utiliser).

- **Clé publique** : point de la courbe P-256 (secp256r1 / prime256v1), forme non compressée de 65 octets : `0x04 ‖ X (32 octets) ‖ Y (32 octets)`, extrait du certificat (voir B.9.3).

Résultat : valide / invalide ; et « non vérifiée » si le certificat n'a pas pu être récupéré.

## B.9 Résolution en ligne (chaîne de confiance ANTS)

### B.9.1 Environnements

Liste **ordonnée**, essayée dans l'ordre, **production d'abord puis repli sur qualification** :

| Nom            | URL de base                                   |
|----------------|-----------------------------------------------|
| Production     | `https://pub.ants.gouv.fr/2D-DOC/V2/PRD`      |
| Qualification  | `https://pub.ants.gouv.fr/2D-DOC/V2/QLF`      |

### B.9.2 URL des ressources

À partir du chemin de chaîne (B.2), de l'identifiant manifest (6 chiffres hex) et des identifiants d'autorité/certificat (B.5) :

```
Manifest    : <base>/04_MANIFESTS/<cheminChaîne>/<idManifest>.xml
Certificat  : <base>/03_CERTIFICATS/<cheminChaîne>/<authorityId>/<certificateId>.cer
```

Pour chaque ressource : essayer l'environnement Production ; en cas d'échec (404, erreur réseau), essayer la Qualification ; mémoriser l'environnement ayant répondu. **Manifest et certificat sont résolus indépendamment.**

### B.9.3 Proxy CORS

Le serveur `pub.ants.gouv.fr` **n'émet pas d'en-tête CORS** ; un navigateur bloque donc la lecture par le JavaScript d'une autre origine. Toutes les requêtes doivent passer par un **relais (proxy) CORS** : l'URL effectivement appelée est `FETCH_PREFIX + urlCible`, où `FETCH_PREFIX` est de la forme `https://<proxy>/?url=`. Le proxy doit être restreint à `pub.ants.gouv.fr/2D-DOC/` pour ne pas constituer un proxy ouvert.

### B.9.4 Extraction de la clé publique du certificat

Le certificat est récupéré en binaire (DER). La méthode pragmatique retenue : rechercher dans les octets le **préfixe SPKI** de prime256v1 :

```
3059301306072a8648ce3d020106082a8648ce3d030107034200
```

Les **65 octets** qui suivent immédiatement ce préfixe sont la clé publique au format `0x04 ‖ X ‖ Y`. Vérifier que le premier de ces octets vaut bien `0x04` et que la longueur est 65.

## B.10 Détection et affichage des images embarquées

Pour un champ Binary, déterminer s'il s'agit d'une image par les **octets magiques** de début :

| Format | Octets de début |
|--------|-----------------|
| PNG    | `89 50 4E 47`   |
| JPEG   | `FF D8 FF`      |
| GIF    | `47 49 46 38`   |
| WEBP   | `52 49 46 46` (RIFF) aux positions 0–3 **et** `57 45 42 50` (WEBP) aux positions 8–11 |

Si une image est reconnue : l'afficher via un URI de données (`data:<type MIME>;base64,<contenu>`) dans une balise image. Sinon : afficher la taille en octets. L'encodage base64 d'un gros contenu doit être fait par tranches pour éviter tout dépassement de pile.

## B.11 Lecture du manifest (XML)

Le manifest est un document XML décrivant le schéma du payload. Le lecteur doit en extraire :

- l'identifiant, le nom et la description du manifest ;
- la **liste ordonnée des champs** du payload ; pour chacun : son **type** (déduit du nom d'élément XML ou d'un attribut de type : String, Date, Integer, Float, Boolean, Binary, Timestamp, Object, et les variantes *Array), son **libellé** (à prendre dans un libellé localisé en français s'il existe, sinon l'attribut `name`), sa contrainte de nullité (« Nillable »), et l'éventuelle référence à un dictionnaire ou à un type complexe ;
- les **types complexes nommés** (définitions d'objets : la liste ordonnée de leurs sous-champs), référencés par les champs Object / ObjectArray ;
- les **dictionnaires** (listes nommées de correspondances clé → libellé) ;
- pour un champ Binary, l'éventuelle **extension Image** et son format.

Recommandation d'implémentation : lire les éléments par leur nom local (sans dépendre des préfixes de namespace) et prévoir des valeurs par défaut tolérantes (libellé = attribut `name` à défaut de libellé localisé). Les noms exacts d'éléments et d'attributs doivent être alignés sur le schéma XSD officiel des manifests ANTS.

## B.12 Dépendances et environnement d'exécution

- **Détection de QR** : une bibliothèque de lecture de QR à partir des pixels d'une image (par ex. jsQR). Elle attend un tableau de pixels RGBA (données issues d'un canvas), **pas** un objet image : pour toutes les sources image, dessiner d'abord l'image/vidéo sur un canvas puis fournir les données de pixels.
- **ECDSA** : une bibliothèque de courbes elliptiques capable de vérifier une signature contre un condensé fourni (voir B.8).
- **SHA-256** : l'API native du navigateur convient (asynchrone).
- Toutes les dépendances sont chargées par CDN ; aucune installation côté serveur.

## B.13 Constantes de configuration (récapitulatif)

| Constante            | Valeur / forme                                                  |
|----------------------|-----------------------------------------------------------------|
| Environnements       | Production `…/V2/PRD`, puis Qualification `…/V2/QLF`            |
| Préfixe proxy        | `https://<proxy>/?url=`                                         |
| Époque des dates     | 1900-01-01 (UTC)                                                |
| Préfixe SPKI P-256   | `3059301306072a8648ce3d020106082a8648ce3d030107034200`         |
| Suffixe chemin (format 6) | `_ISO22376`                                                |

---

# Partie C — Jeu de validation

L'implémentation doit être validée contre le **jeu de données de test officiel de l'ANTS** (JDD 2D-DOC V2). Le JDD fournit, pour chaque manifest de test, des vecteurs en trois représentations (hexadécimale, base32, base45) ; la représentation **base45** est exactement le contenu d'un QR.

Critères d'acceptation, à vérifier au minimum sur le manifest `000000` (cas signé par `FR99/TEST`) :

1. Décodage Base45 : premier octet `0xDE`.
2. En-tête : marqueur `0xDE`, version `0x03`, signataire C40 = `KFRFR99TEST0` (IAC `KFR`, autorité `FR99`, certificat `TEST`), identifiant manifest `000000`.
3. Payload : décodé comme un **flux** de valeurs (par ex. `nil`, `"2D-DOC V2"`, `"A"`, `"B"`), et non comme une valeur unique.
4. Signature : récupération du certificat `TEST` en ligne, extraction de la clé publique, et vérification **valide** selon le schéma `SHA-256( en-tête ‖ SHA-256(payload) )`.
5. Affichage dégradé : en simulant un manifest et/ou un certificat introuvables, le contenu reste affiché avec l'avertissement adéquat.
6. Image embarquée : sur un vecteur contenant une image (par ex. manifest `000406`), l'image s'affiche.

Toute non-conformité à l'un de ces points indique une erreur d'implémentation à corriger avant livraison.
