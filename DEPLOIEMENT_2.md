# Suivi SIRI · Institut Agro — mise en ligne (GitHub Pages + Supabase)

Trois fichiers :

| Fichier | Rôle |
|---|---|
| `index.html` | l'application complète (un seul fichier, ~1,35 Mo, bibliothèque cartographique Leaflet incluse), configurée pour le projet Supabase `aekyxbvlzpgarqhnqnks` |
| `supabase-setup.sql` | crée les deux tables (suivi et comptes), les droits par rôle et périmètre, l'abonnement temps réel et le stockage privé des documents |
| `DEPLOIEMENT.md` | ce guide |

## 1. Supabase — table, comptes et stockage

> **Avant de lancer le script sur un suivi déjà alimenté**, ouvrez l'application actuelle → onglet **Données** → **Exporter .json**. Ce script **ferme l'accès anonyme** : ensuite, plus personne n'ouvre le suivi sans compte.

1. Ouvrir le projet sur https://supabase.com/dashboard → **Authentication → Sign In / Providers** : vérifier que **Email** est activé et que « Allow new users to sign up » est coché (sinon personne ne pourra créer de compte). Si « Confirm email » est activé, chaque personne devra cliquer sur le lien reçu avant de se connecter.
2. **SQL Editor** → **New query** → coller le contenu de `supabase-setup.sql` → **Run**.
3. Vérifier dans **Table Editor** que `scan_to_bim_suivi` et `scan_to_bim_profils` existent, et dans **Storage** que le bucket `documents` apparaît (il est désormais **privé**).
4. Ouvrir la page et **créer votre compte** : le tout premier compte créé devient automatiquement administrateur, avec tous les périmètres.

Le script est idempotent : il peut être relancé sans risque. Il crée :

- la table `public.scan_to_bim_suivi` (une ligne par élément du suivi : `buildings/MTP-01`, `budget/dijon_lod100`, `pins/REN-03`, `ifc/DIJ-05`, `docs/<id>`, `qc/DIJ-05`, `scenarios/<id>`, `goal`, `approb`, `lock`, …) ;
- la table `public.scan_to_bim_profils` (un profil par compte : rôle, périmètre, actif) et le déclencheur qui la remplit à chaque inscription ;
- les fonctions de droits (`stb_role`, `stb_sites`, `stb_scope`, `stb_can_write`) et les policies RLS qui les appliquent : **une écriture hors rôle ou hors périmètre est refusée par le serveur**, pas seulement masquée à l'écran ;
- un trigger qui horodate chaque écriture côté serveur (`updated_at`) ;
- l'ajout de la table à la publication `supabase_realtime` (diffusion instantanée des modifications) ;
- le bucket **`documents`** (privé, 200 Mo par fichier) et ses policies : les fichiers ne sont lisibles que par un lien signé d'une heure, généré pour une personne connectée.

La section 7 du script, en commentaire, permet de revenir à un accès ouvert si vous changez d'avis.

Tant que la table n'existe pas, la page fonctionne quand même : l'indicateur en bas du menu affiche « Local · Supabase indisponible », l'onglet **Données** explique la cause, et les saisies restent dans le navigateur. Si seul le bucket manque, tout fonctionne sauf le dépôt de documents, qui affiche un message invitant à relancer ce script.

## 2. GitHub — publier la page

1. Créer un dépôt (par exemple `suivi-scan-to-bim`), y déposer `index.html` (Add file → Upload files → Commit).
2. **Settings → Pages → Build and deployment** : Source = *Deploy from a branch*, Branch = `main`, dossier `/ (root)` → **Save**.
3. Après une à deux minutes, la page est disponible à `https://<compte>.github.io/suivi-scan-to-bim/`.

Mise à jour ultérieure : remplacer `index.html` dans le dépôt (même nom) ; GitHub Pages redéploie automatiquement. Les données ne sont pas dans le fichier, elles restent dans Supabase.

**Vérifier quelle version est ouverte.** Le bas du menu latéral affiche une empreinte de version, par exemple `Version 18/09/2026 · cbdaa3d`, suivie de l'origine du fichier (`site GitHub Pages` ou `artefact Claude`). Si cette empreinte ne correspond pas à celle de la version livrée, la page affichée est une copie ancienne : soit le fichier `index.html` du dépôt n'a pas été remplacé, soit le navigateur sert sa copie en cache. GitHub Pages autorise la mise en cache pendant dix minutes ; un rechargement forcé (`Ctrl+F5`, ou `Cmd+Maj+R` sur Mac) affiche immédiatement la version déployée.

## 3. Partager : comptes, rôles et périmètres

Transmettez l'adresse GitHub Pages. À l'ouverture, la page demande une **adresse e-mail et un mot de passe** ; sans compte actif, rien du suivi n'est lisible.

**Comment une personne entre.** Elle clique sur « Créer un compte », saisit son nom, son adresse professionnelle et un mot de passe. Son compte existe alors, mais ne donne accès à rien : elle voit un écran « Compte en attente d'autorisation », qui se débloque tout seul dès qu'un administrateur lui a donné un rôle. Vous n'avez donc jamais de mot de passe à créer ni à transmettre.

**Ce que fait l'administrateur.** Onglet **Administration** (visible des seuls administrateurs) : pour chaque compte, un **rôle** et un **périmètre** (Dijon, Montpellier, Rennes, Topo & VRD). Attribuer un rôle active le compte ; décocher « actif » le bloque sans le supprimer.

| Rôle | Ce qu'il peut faire |
|---|---|
| **Administrateur** | tout, sur tous les sites, et la gestion des comptes |
| **AMO** | tous les sites, taux, objectif, scénarios et contrôles ; ne gère pas les comptes |
| **Responsable de site** | saisie complète sur son périmètre : suivi, données d'entrée, contrôles qualité, scénarios |
| **Contributeur** | suivi sur son périmètre : statuts, avancement, dates, commandes, factures, documents |
| **Lecteur** | consultation et exports, aucune modification |

**Ce qui est hors périmètre n'est pas affiché du tout.** Un contributeur Rennes n'a ni l'onglet Dijon, ni l'onglet Montpellier, ni Topo & VRD s'il ne l'a pas ; ses tableaux, graphiques, carte, contrôles et exports ne portent que sur Rennes, et le tableau de bord rappelle le périmètre en haut de page. Une adresse directe (`#/site/dijon`) renvoie au tableau de bord. Surtout, **le serveur ne renvoie pas les lignes hors périmètre** : la policy de lecture applique la même règle que l'écran (`stb_can_read`), donc interroger la base directement avec la clé publique ne donne rien de plus. À l'intérieur de son périmètre, ce qu'une personne peut modifier dépend ensuite de son rôle (champs grisés, et écriture refusée par le serveur).

Deux exceptions volontaires : les **réglages globaux** (taux, objectif, libellés de priorité, approbateurs) restent lisibles par tout compte actif — l'application en a besoin pour calculer les budgets ; et les **scénarios**, transverses aux sites par nature, ne sont visibles qu'à partir du rôle Responsable de site, quel que soit le périmètre, pour que le responsable du marché puisse donner son approbation.

Une saisie apparaît chez les autres en moins d'une seconde (WebSocket) ou en quelques secondes si le temps réel est indisponible sur le réseau (repli automatique par interrogation périodique, puis reconnexion). La session est mémorisée sur le poste : on ne se reconnecte pas à chaque ouverture. « Se déconnecter » se trouve en bas du menu, et « Mot de passe oublié » sur l'écran de connexion (Supabase envoie le message de réinitialisation).

**Si vous perdez l'accès administrateur**, le tableau `scan_to_bim_profils` reste modifiable depuis le Table Editor de Supabase : passez votre ligne à `role = 'admin'` et `actif = true`.

L'indicateur en bas du menu précise l'état : **Supabase · à jour**, **Enregistrement…**, **Supabase · erreur — nouvel essai** (la saisie est conservée et renvoyée toutes les 4 s). L'onglet **Données → Synchronisation Supabase** détaille projet, table, état du temps réel et nombre d'enregistrements.

## 4. Le menu latéral

La navigation est un menu vertical à gauche, regroupé en cinq sections (Pilotage, Sites, Analyses, Livrables, Paramètres), avec des compteurs sur les onglets Scénarios, Topo & VRD, Livrables IFC et Contrôles.

- Le chevron en haut du menu le **réduit en une colonne d'icônes** (et l'élargit à nouveau) ; le choix est conservé sur le poste. Raccourci : touche <kbd>m</kbd>.
- En dessous de 1100 px de large (tablette, portable en écran partagé), le menu se referme et s'ouvre par le bouton ☰ du bandeau ; <kbd>Échap</kbd> ou un clic à côté le referme.
- Le bandeau supérieur ne contient plus que les indicateurs, le filtre de priorité, le thème clair/sombre et l'export.

Les deux enveloppes y sont distinctes : **Budget bâtiments** (Scan-to-BIM) et **Topo & VRD** sont deux indicateurs séparés, et les sous-titres de *Commandé HT* et *Facturé TTC* rappellent la part de chacune. Un indicateur **Objectif** n'apparaît que si un objectif budgétaire est saisi (onglet Données) : il compare la somme des deux enveloppes à cet objectif et s'affiche en vert tant qu'il est tenu.

## 5. Carte et punaises

L'onglet **Carte** affiche les photographies aériennes ou le Plan IGN, avec en option la couche **Cadastre** (parcelles PCI Express) — trois services de la Géoplateforme IGN, gratuits et sans clé. Chaque bâtiment est représenté par une punaise :

- **pleine** (⬤) : punaise placée à la main (partagée avec tous les utilisateurs, ligne `pins/<bâtiment>` dans Supabase) ;
- **tiretée** (◐) : position estimée depuis l'emprise OpenStreetMap associée ou suggérée — « Valider la position » l'enregistre telle quelle, ou glisser la punaise pour l'ajuster ;
- **pointillée** (◌) : position *approchée* d'une annexe, déduite du numéro — voir ci-dessous ;
- absente (○ dans la liste) : sélectionner le bâtiment puis cliquer sur la carte à son emplacement.

**Annexes et bâtiments principaux (Montpellier).** Le code à quatre chiffres rattache l'annexe à son bâtiment : « BAT 0121 » est une annexe de « BAT 0120 ». OpenStreetMap ne nomme pratiquement aucune annexe ; faute d'emprise, l'annexe est donc posée **à côté** de son bâtiment principal — à une quinzaine de mètres, dans une direction arbitraire, punaise en pointillé. Cette position situe le bon îlot, **pas l'emplacement exact** : elle est à corriger en faisant glisser la punaise sur la photo aérienne. Conséquence utile : placer un bâtiment principal positionne du même coup toutes ses annexes.

**Placement guidé.** Le bouton **« Placement guidé »**, au-dessus de la liste, enchaîne les bâtiments de la zone qui n'ont ni punaise ni emprise : un clic sur la carte place la punaise et passe au suivant (boutons « Passer » et « Arrêter »). Les bâtiments principaux viennent en premier, puisqu'ils entraînent leurs annexes ; pendant le placement, les emprises OpenStreetMap encore libres ressortent en jaune et un clic sur l'une d'elles suffit à l'associer. La ligne **« 43.618123, 3.855400 »** sous le bâtiment sélectionné accepte aussi des coordonnées collées, ou un lien Google Maps / OpenStreetMap.

L'onglet **Contrôles** récapitule, site par site, les bâtiments sans position, les positions approchées d'annexes et les positions estimées à valider ; l'export CSV porte les colonnes *Latitude*, *Longitude* et *Position* (`punaise`, `estimée (OSM)`, `approchée (annexe)`).

**Numérotation des punaises.** Le numéro porté par la punaise est celui du libellé du bâtiment, pas un numéro d'ordre : à Montpellier, « BAT 0070 » donne **07** et « BAT 0071 » donne **07.1** (les trois premiers chiffres font le numéro, le quatrième le sous-numéro) ; à Rennes, « BAT 18 » donne **18**, « BAT 9 BIS » donne **9b**, « BATIMENT A » donne **A**. Les bâtiments dont le libellé ne porte aucun numéro (Dijon, une partie de Rennes) gardent leur numéro d'ordre dans le site, précédé d'un « n » si ce numéro est déjà celui d'un autre bâtiment. La liste à droite de la carte et le filtre utilisent le même numéro.

Les punaises se déplacent en les faisant glisser ; double-clic ouvre la fiche. Le bouton **France** montre les cinq campus. Sans connexion, ou dans la version hébergée sur claude.ai (images externes bloquées), un plan simplifié dessiné depuis les données OpenStreetMap embarquées remplace le fond de carte.

## 6. Documents Topo & VRD (PDF et IFC)

En bas de l'onglet **Topo & VRD**, la carte **Documents** reçoit les plans de récolement, levés et maquettes VRD : glisser les fichiers sur la zone prévue ou utiliser « Ajouter des fichiers… ». Formats acceptés : **.pdf** et **.ifc**, jusqu'à 200 Mo par fichier.

- Les fichiers partent dans le bucket `documents` du projet Supabase (chemin `topo/<secteur>/<id>-<nom>`), leur fiche (nom, taille, secteur, commentaire, date) dans la table partagée : **tous les utilisateurs de la page voient et ouvrent les mêmes documents**.
- Les chips en haut de la carte filtrent par secteur ; la liste déroulante « Secteur du dépôt » choisit le secteur de rattachement des fichiers ajoutés.
- **Voir** ouvre la visionneuse intégrée : les PDF dans un cadre de lecture, les IFC en 3D avec le nombre d'éléments et de triangles, le filtrage par niveau et par famille (architecture, structure, MEP, espaces, mobilier, objets non typés) et le clic sur un élément pour ses informations. Boutons *Ajuster*, *Vue de dessus*, *Face* et *Image* (capture PNG). <kbd>Échap</kbd> ferme.
- Au-delà de 80 Mo, une maquette reste téléchargeable mais n'est pas affichée en 3D (limite du rendu en navigateur) ; le fichier n'est alors pas téléchargé inutilement.
- **✕** retire le document pour tout le monde et supprime le fichier du stockage.
- Dans la version hébergée sur claude.ai (sans Supabase), un fichier ouvert ici reste visible **pour la session en cours uniquement** : il porte la mention « session », n'est pas partagé et disparaît au rechargement.

## 7. Contrôle qualité des livrables IFC et verrouillage de l'onglet

L'onglet **Contrôles** réunit deux choses : le contrôle qualité des maquettes livrées et les contrôles de cohérence des données du suivi (inchangés).

**Contrôle qualité.** Dès qu'une maquette est rattachée à un bâtiment (fiche du bâtiment → Livrables IFC), elle apparaît dans le tableau des livrables. « Contrôler » déplie sa fiche :

- **17 contrôles automatiques**, recalculés à chaque affichage depuis la lecture du fichier : schéma déclaré, vue de définition (MVD), unité de longueur, géoréférencement, unicité de l'IfcBuilding, nombre de niveaux comparé à la fiche du bâtiment, altimétrie des niveaux, présence de géométrie, part d'objets non typés (proxy), espaces, jeux de propriétés, quantités, nom de fichier portant le code du bâtiment, auteur et date d'export, unicité de l'empreinte SHA-256, indice de version, lien GED. Chacun est classé *conforme*, *à vérifier* ou *bloquant*.
- **10 contrôles manuels** de réception (géoréférencement, nomenclature, recalage sur le nuage, niveau de détail, classification, espaces et surfaces, propriétés, réseaux, cohérence du modèle, livrables associés) en trois états — ✓ conforme, ✕ non conforme, — sans objet — avec une observation par point, le nom du contrôleur et un avis global. « Tout marquer conforme » complète les points restés vides.
- L'**avis** de chaque livrable se déduit de l'ensemble : *Conforme*, *Conforme avec réserves*, *Non conforme*, ou *Contrôle à finir*. Tout est partagé (lignes `qc/<bâtiment>`), repris dans l'export JSON et détaillé dans une feuille « C-Q livrables » du classeur Excel.

**Verrouillage.** La carte *Protection de cet onglet*, en bas de l'onglet, permet de le réserver par un mot de passe :

- « Protéger par un mot de passe… » demande le mot de passe deux fois ; seule une **empreinte SHA-256 salée** est enregistrée (ligne `lock`), jamais le mot de passe lui-même.
- Les autres navigateurs voient alors un écran de saisie à la place de l'onglet ; la saisie n'est demandée **qu'une fois par navigateur** (elle y est mémorisée tant que le mot de passe ne change pas). Changer le mot de passe redemande la saisie à tout le monde.
- L'option « Masquer aussi l'onglet du menu » le fait disparaître de la navigation. Il reste atteignable en ajoutant `#controls` à la fin de l'adresse, ou avec la touche <kbd>0</kbd> : le mot de passe est alors demandé.
- « Verrouiller maintenant » reverrouille le poste courant ; « Retirer la protection » rouvre l'onglet pour tout le monde.

> **Portée de cette protection.** C'est un verrou d'affichage côté navigateur. Depuis la mise en place des comptes (§3), ce qui protège réellement l'onglet est le **rôle** : les contrôles qualité ne sont modifiables que par un responsable de site sur son périmètre, un AMO ou un administrateur. Le mot de passe reste utile si vous voulez, en plus, que l'onglet ne s'ouvre pas d'un simple clic ; il fait autrement double emploi et peut être retiré.

## 8. Scénarios et intégration au suivi

L'onglet **Scénarios** sert aux arbitrages : il simule des priorités et des prestations sans rien modifier dans le suivi.

- **Règles par entité** (Dijon, Montpellier, Rennes) : une cible (tous les bâtiments du site, ou seulement les P1/P2/P3/non priorisés), une priorité de remplacement, un type de numérisation, et pour chacune des 8 prestations « inchangée / incluse / exclue ». Une prestation *incluse* qui n'était pas chiffrée est valorisée au taux de l'onglet Taux.
- **Réglages par bâtiment** : ils l'emportent sur la règle de son site — priorité, pastille par prestation (clic : inchangée → inverse de la règle → autre état), numérisation, exclusion du programme.
- **Comparaison** : budget par priorité (année), par site et par prestation, scénario contre suivi actuel, écart à l'objectif consolidé, et un tableau comparant tous les scénarios entre eux.
- **Approbation obligatoire** : « Intégrer au suivi » reste bloqué tant que **le responsable du marché** et **l'AMO** n'ont pas signé (par défaut Magali PELLCUER et Laurent COLOMBERO ; les noms et les comptes autorisés à signer se règlent dans l'onglet Administration). Chaque signature ne peut être donnée que par le compte désigné, elle est horodatée et nominative, et elle porte sur une **empreinte du contenu du scénario** : si une règle ou un bâtiment change ensuite, les signatures deviennent caduques et doivent être renouvelées. Tant que les deux ne sont pas réunies, les boutons d'intégration — global et par site — sont inactifs.
- **Intégration** : une fois les deux signatures acquises, « Valider et intégrer au suivi » écrit les options cochées (priorités, prestations et numérisation, exclusions → statut « Non retenu ») pour les 3 sites ou pour un seul. Les valeurs de base restent récupérables dans chaque fiche, le scénario garde la trace de ce qui a été intégré et quand.

Les scénarios sont partagés comme le reste du suivi (une ligne `scenarios/<id>`), repris dans l'export JSON et résumés dans une feuille « Scénarios » du classeur Excel.

## 9. Reprendre les saisies existantes

La version hébergée sur claude.ai et la version GitHub/Supabase utilisent deux stockages distincts. Pour transférer le suivi actuel :

1. dans la version claude.ai : **Données → Sauvegarde du suivi (JSON) → Exporter .json** ;
2. dans la version GitHub Pages : **Données → Importer un .json** → confirmer.

Tout est repris (statuts, avancement, commandes, factures, données d'entrée modifiées, taux, registre IFC, punaises et associations de la carte, scénarios, contrôles qualité, fiches des documents). L'import remplace l'ensemble des saisies présentes dans Supabase. Les fichiers déposés dans le bucket ne sont pas concernés par l'export JSON : ils restent dans le stockage Supabase.

## 10. Sécurité — à connaître

- **L'accès est fermé.** Sans compte actif, la table du suivi et le bucket des documents ne renvoient rien, même en interrogeant Supabase directement avec la clé publique de la page : les policies RLS n'ouvrent la lecture qu'au rôle `authenticated` porteur d'un profil actif. La page GitHub Pages reste publiquement *accessible*, mais elle ne montre qu'un écran de connexion.
- **Le périmètre est appliqué côté serveur, en lecture comme en écriture.** Les fonctions `stb_can_read` et `stb_can_write` rejouent dans la base la règle affichée à l'écran : un contributeur Rennes ne voit et n'écrit aucune ligne d'un autre site, quel que soit l'outil utilisé.
- **Une limite à connaître.** Le référentiel de base — la liste des 137 bâtiments avec leurs libellés, surfaces et budgets de référence — est embarqué dans `index.html` lui-même. Il est donc masqué à l'écran hors périmètre, mais quelqu'un qui ouvrirait le code source de la page le retrouverait. Ce qui relève du **suivi** (statuts, avancement, commandes, factures, contrôles qualité, documents, punaises) est, lui, réellement inaccessible hors périmètre. Pour fermer aussi le référentiel, il faudrait le sortir du fichier et le charger depuis Supabase derrière les mêmes policies.
- La clé `sb_publishable_…` reste une clé **publique** prévue pour être embarquée dans une page ; elle identifie le projet, elle n'ouvre aucun droit par elle-même.
- **Documents** : le bucket est privé, les fichiers ne sont servis que par un lien signé valable une heure, généré pour une personne connectée. Un lien copié reste valable jusqu'à son expiration.
- Les fichiers IFC rattachés aux bâtiments (onglet Livrables IFC) ne sont, eux, jamais téléversés : seule leur fiche (schéma, niveaux, éléments, empreinte SHA-256, lien GED) est enregistrée. Les maquettes que vous voulez rendre consultables passent par l'onglet Topo & VRD.
- Le mot de passe de l'onglet Contrôles (§7) fait maintenant double emploi avec les rôles : il reste disponible, mais c'est le rôle qui protège réellement. Vous pouvez retirer cette protection sans rien perdre.
- **Mots de passe** : ils sont gérés par Supabase (hachés côté serveur, réinitialisation par courriel). L'application ne les voit jamais et n'en conserve aucun.
- Sauvegarde : l'export JSON depuis l'onglet **Données** reste le moyen simple d'archiver un état complet ; les sauvegardes automatiques de Supabase n'existent que sur les plans payants.

## 11. Changer de projet ou de table

Les trois valeurs sont en tête de `index.html` (lignes 6–8) :

```html
<script>window.SUPABASE_CONFIG = { "url": "https://aekyxbvlzpgarqhnqnks.supabase.co",
                                  "key": "sb_publishable_…", "table": "scan_to_bim_suivi" };</script>
```

Si la table est renommée, adapter aussi son nom dans `supabase-setup.sql` avant de l'exécuter. Le nom du bucket (`documents`) est fixé dans les deux fichiers.
