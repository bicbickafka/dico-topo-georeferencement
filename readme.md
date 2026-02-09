<div style="display: flex; gap: 10px; width: 100%; margin-bottom: 10px;">
  <img src="image_6.png" style="width: 32.5%; border-radius: 4px; object-fit: cover;">
  <img src="image_5.png" style="width: 32.5%; border-radius: 4px; object-fit: cover;">
  <img src="image_7.png" style="width: 32.5%; border-radius: 4px; object-fit: cover;">
</div>
<div style="display: flex; gap: 10px; width: 100%; margin-bottom: 20px;">
  <img src="image_8.png" style="width: 49.3%; border-radius: 4px; object-fit: cover;">
  <img src="image_12.png" style="width: 49.3%; border-radius: 4px; object-fit: cover;">
</div>

==Table des matières==

# Présentation

Les dictionnaires topographiques suivent des conventions éditoriales communes bien respectées, ce qui a permis l'encodage du corpus au format XML. Ce dépôt GitHub regroupe les scripts qui enrichissent ces données avec les codes du Code officiel géographique (COG) de 2011. Ce travail de géoréférencement permet de cartographier les toponymes et d'assurer l'interrogation[^1]  et l'interopérabilité du corpus à travers sa dimension spatiale.

À titre d'exemple, la chaîne de traitement est illustrée par le Dictionnaire topographique du département de la Mayenne, disponible sur [Gallica](https://gallica.bnf.fr/ark:/12148/bpt6k204189z/f55.item).

![[image_2.png]]

Vue d'ensemble de la chaîne de traitement :

| étape              | input                                                                                      | output               |
| ------------------ | ------------------------------------------------------------------------------------------ | -------------------- |
| parse.py           | DT53.xml                                                                                   | DT53_parsed.xlsx     |
| classify.py        | DT53_parsed.xlsx                                                                           | DT53_classified.xlsx |
| recognize.py       | DT53_classified.xlsx                                                                       | DT53_recognized.xlsx |
| match.py           | DT53_recognized.xlsx<br>DT03_COG_2011.xlsx                                                 | DT53_matched.xlsx    |
| validation experte | DT53_matched.xlsx                                                                          | DT53_validated.xlsx  |
| inject.py          | DT53_validated.xlsx                                                                        | DT53_injected.xml    |
| control.py         | DT53.xml<br>DT53_injected.xml<br>DT03_validated.xlsx<br>DT03_COG_2011.xlsx<br>dicotopo.rng | DT53_controlled.xlsx |

# Chaîne de traitement



## 1. parse.py

Ce premier script transforme la structure XML en tableau. Il extrait cinq champs de chaque article : l'identifiant, la vedette (le nom du lieu), la définition, la typologie et la localisation (au format JSON).

Par exemple, cet extrait XML :
![[image_15.png|400]]
devient :

| champ          | contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |
### 2. classify.py

Ce deuxième script distingue les communes des autres types de lieux. Cette distinction est essentielle car les communes s'apparient directement au COG par leur vedette, tandis que les lieux-dits (hameaux, fermes, bois, etc.) doivent être géolocalisés indirectement via les communes mentionnées dans leur localisation. Le script `classify.py` analyse la typologie de chaque entrée : si elle mentionne « arrondissement », « canton », « chef-lieu » ou « commune », l'entrée est identifiée comme une commune (`is_commune: true`). Dans tous les autres cas, elle ne l'est pas (`is_commune: false`). Attention : cette règle ne couvre pas tous les cas, notamment lorsque les typologies sont vides, une validation experte reste nécessaire.
![[image_3.png]]
Exemple : Pour l'entrée « Villeneuve », la typologie indique « hameau ». Le script enregistre donc `is_commune: false`, ce qui signale que ce lieu devra être géoréférencé via les communes mentionnées dans sa localisation.

| champ          | contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |
| `is_commune`   | false                                                                    |
### 3. recognize.py

Il nous faut maintenant extraire les noms de lieux qui permettront l'appariement avec le COG. Pour les communes (`is_commune: true`), le nom est la vedette elle-même. Pour les autres lieux, il faut identifier la ou les communes mentionnées dans le champ `localisation`.
#### Reconnaissance d'entités nommées

Le script utilise la bibliothèque SpaCy pour détecter automatiquement les toponymes : entités géopolitiques (GPE) comme les communes, et lieux géographiques (LOC) comme les rivières. Un système de secours basé sur des expressions régulières capture les mots à majuscule initiale en cas d'échec.
#### Normalisation des toponymes

Les toponymes extraits sont normalisés (suppression des articles, apostrophes, tirets et accents, conversion en minuscules) puis les doublons sont éliminés. Les noms normalisés sont enregistrés dans la colonne `commune_norm` au format JSON.

| champ          | contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |
| `is_commune`   | false                                                                    |
| `commune_norm` | ["bazouges", "chateau gontier"]                                          |
### 4. [[match.py]]

Ce script associe chaque toponyme normalisé (la 'clé'/'key') à un nom du COG (normalisé aussi) en trois étapes, chaque étape annotant les résultats avec le nom du COG (`NCCENR`), le code du COG (`INSEE`) et la méthode utilisée (`match`).

3 étapes :

| étape                               | exemple                                                                  | match |
| ----------------------------------- | ------------------------------------------------------------------------ | ----- |
| key = COG                           | "chateau gontier" correspond à "chateau gontier"                         | exact |
| first_token(key) = first_token(COG) | "couesmes" correspond à "couesmes  vauce" ("couesmes"  = "couesmes" )    | fuzzy |
| key ∈ tokens(COG)                   | "vauce" correspond à "couesmes  vauce" ("vauce" ∈ ["couesmes", "vauce"]) | fuzzy |
2 exceptions :

| exception                                                | exemple                                                                       | match          |
| -------------------------------------------------------- | ----------------------------------------------------------------------------- | -------------- |
| il peut y avoir une écart d'une lettre (levenshtein ≤ 1) | "bazouges" correspond à "bazougers"                                           | fuzzy          |
| plusieurs communes correspondent                         | "deneuille" correspond à "deneuille les chantelle" et à "deneuille les mines" | fuzzy_multiple |

Pour notre exemple :

| champ          | contenu                                                                  |
| -------------- | ------------------------------------------------------------------------ |
| `id`           | DT53-23764                                                               |
| `vedette`      | Villeneuve                                                               |
| `definition`   | hameau, commune de Bazouges, réuni à la ville de Château-Gontier en 1862 |
| `typologie`    | hameau                                                                   |
| `localisation` | ["commune de Bazouges", "ville de Château-Gontier"]                      |
| `is_commune`   | false                                                                    |
| `commune_norm` | ["bazouges", "chateau gontier"]                                          |
| `NCCENR`       | ["Bazougers", "Château-Gontier"]                                         |
| `INSEE`        | ["53025", "53062"]                                                       |
| `match`        | ["fuzzy", "exact"]                                                       |
### 5. inject.py

**inject.py** enrichit selon `is_commune` : a commune article (the article itself gets an attribute and a child tag) or a lieu-dit article (the names inside the `<localisation>` tag get attributes).

==(schema maken ipv tableau)==

![[image_15.png|400]]

| is_commune | input                                                                     | output                                                                                                                                                     |
| ---------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| true       | `<article id="DT53-13633" pg="189">`                                      | `<article id="DT53-13633" pg="189" type="commune">`<br><br>`<insee>53147</insee>`                                                                          |
| false      | `<localisation>communes de Bazouges et de Château-Gontier</localisation>` | `<localisation>communes de <commune insee="53025">Bazouges</commune></localisation> et de <commune insee="53062">Château-Gontier</commune></localisation>` |
### 6. control.py

Ce dernier script garantit qu'aucune information n'a été perdue ou corrompue lors des enrichissements. Il génère un rapport de contrôle (`DT53_controlled.xlsx`) organisé en trois onglets qui vérifient respectivement l'intégrité du fichier XML, la validité des enrichissements et la conformité XML.
#### Intégrité

Cet onglet compare le fichier XML enrichi (`DT53_injected.xml`) au fichier source (`DT53.xml`) pour détecter toute altération du contenu original :

| id         | DT53.xml                                                       | DT53_injected.xml                                               |
| ---------- | -------------------------------------------------------------- | --------------------------------------------------------------- |
| DT03-00001 | `<definition>hameau, commune de Saint-Martinien.</definition>` | `<definition>hameau., commune de Saint-Martinien.</definition>` |
| DT03-00001 | pg="1"                                                         | pg="2"                                                          |
| DT03-00001 | id="DT03-00001"                                                | id="DT03-00002"                                                 |
| DT03-00001 | id="DT03-00001"                                                | n/a                                                             |
#### Validité

Cet onglet croise le fichier XML enrichi (`DT53_injected.xml`) avec le tableau validé (`DT53_validated.xlsx`) et la liste du COG 2011 (`DT53_COG_2011.xlsx`) pour identifier les enrichissements manquants ou invalides :

| **id**     | **problem**             | **DT03_injected.xml**                | **correction**                                      |
| ---------- | ----------------------- | ------------------------------------ | --------------------------------------------------- |
| DT03-00001 | attribute_insee_missing | `<commune>Saint-Martinien</commune>` | `<commune insee="3246">Saint-Martinien</commune>`   |
| DT03-13255 | attribute_type_missing  | `<article id="DT03-13255" pg="207">` | `<article id="DT03-13255" pg="207" type="commune">` |
| DT03-13255 | balise_insee_missing    | `n/a`                                | `<insee>3002</insee>`                               |
| DT03-00002 | insee_invalid           | `insee="4059"`                       | `insee="3059"`                                      |
| DT03-13256 | insee_invalid           | `<insee>4003</insee>`                | `<insee>3003</insee>`                               |
#### Conformité

Cet onglet vérifie la conformité XML du fichier enrichi en s'assurant qu'il est bien formé et valide selon les règles définies par le schéma RelaxNG dicotopo.rng.

| **check**       | **status** | **error**                                                  |
| --------------- | ---------- | ---------------------------------------------------------- |
| well-formedness | passed     | n/a                                                        |
| dico-topo.rng   | invalid    | line 8: element DICTIONNAIRE failed to validate attributes |
| dico-topo.rng   | invalid    | line 9: element article failed to validate attributes      |
| dico-topo.rng   | invalid    | line 10: element vedette has extra content: pg             |



[^1]: trier et filtrer les résultats selon un découpage administratif, regrouper les lieux par commune d'appartenance, travailler aisément à échelle nationale ce que la fragmentation en dictionnaires départementaux rendait complexe et laborieux.