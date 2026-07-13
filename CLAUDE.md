# Cantrip — Suivi de Personnage JDR

PWA de suivi de personnage JDR (PV, emplacements de sorts, ressource de classe, fiche
Personnage/Skills, Grimoire) construite en juillet 2026. Le propriétaire travaille alternativement
sur deux PC (fixe et portable) : ce fichier sert de mémoire portable entre les deux, en
complément du dépôt Git qui est la seule source de vérité partagée entre les machines.

## Stack technique

- Fichier unique `index.html` : HTML/CSS/JS vanilla, **pas de framework, pas de build step**.
- Tailwind CSS via CDN (`<script src="https://cdn.tailwindcss.com">`) — utilisé de façon
  marginale, la majorité du style est en `style=""` inline directement dans les templates JS.
- Police Cinzel (Google Fonts) via `class="font-cinzel"`.
- Persistance : `localStorage`, clé `jdr_character_tracker_state` (conservée telle quelle après
  le renommage de l'app en Cantrip, pour ne pas perdre les sauvegardes existantes). Pas de
  backend.
- PWA : `manifest.json` + `icon.svg` + `sw.js` (service worker, stratégie réseau d'abord avec
  fallback cache — voir section Service Worker ci-dessous).
- Portraits des personnages : `characters/calix.jpg` + `characters/deneor.png`, fichiers statiques
  (pas de base64 inline) référencés depuis `state.characters.*.portrait` — voir section
  Personnages ci-dessous.

## Structure du code (`index.html`)

Application à état unique en IIFE (`(function () { 'use strict'; ... })()`), rendu par
réécriture complète de `innerHTML` à chaque changement (pas de diffing, pas de framework).

- `state` : données persistées (profils de personnage). Sauvegardé via `saveState()` à chaque
  mutation.
- `ui` : état éphémère non persisté (page active, champs en cours d'édition, recherche...).
  Voir `ui.view` pour la page courante.
- `render()` : dispatch vers `renderTracker()` / `renderStats()` / `renderGrimoire()` /
  `renderSettings()` (menu Paramètres) / `renderSettingsCharacter()` / `renderSettingsGrimoire()` /
  `renderSettingsApp()` / `renderSettingsLoadCharacter()` selon `ui.view`, puis ajoute
  `renderBottomNav()`. Chaque page occupe toute la hauteur disponible (`height:100%`, pas de
  scroll de la page globale — seuls les conteneurs `[data-scroll-root]` scrollent en interne).
- `bindEvents()` : re-attache tous les event listeners après chaque re-render (délégation via
  `data-action` sur les éléments cliquables).

### Pages (barre de navigation basse, 4 onglets)

1. **Tracker** (page par défaut) — portrait du personnage actif (`state.characters[activeCharacterId]
   .portrait`, cercle 38px) à gauche du nom, ligne de 3 tuiles CA / Initiative /
   Déplacement (`profile.combatStats: { ac, initiative, speed }`, lecture seule, éditées dans
   Paramètres), PV (bloc fusionné avec bouclier temporaire), emplacements de sorts groupés par
   niveau en badges circulaires, ressource(s) de classe (un bloc de badges par ressource
   configurée, voir Paramètres ci-dessous), bouton "Repos" unique.
   Deux marqueurs toggle (`profile.concentration`, `profile.inspiration`, booléens indépendants)
   alignés à droite sur la ligne du nom du personnage, dans l'ordre inspiration puis
   concentration : point d'inspiration (icône étoile
   `iconStar(filled)`, se remplit — `fill:currentColor` — quand actif, simple surbrillance ambre,
   pas de glow) et concentration (icône smiley aux sourcils froncés `ICON_CONCENTRATION`, glow
   violet animé via la classe `concentration-active` / `@keyframes concentration-pulse`).
2. **Personnage** (onglet nav, anciennement "Stats" ; le code interne — `renderStats()`,
   `ui.statsSearch`, `view: 'stats'` — garde le nom `stats`) — caractéristiques (6) et
   compétences (18, D&D 5e, noms français) en lecture seule, avec champ de recherche filtrant
   la liste en direct. Les valeurs sont saisies manuellement dans Paramètres (l'app ne calcule
   aucun modificateur).
3. **Grimoire** — affiche les capacités du personnage (sorts par niveau + capacités de classe et
   dons), regroupées par section avec badge de type d'action coloré (Action/Bonus/Réaction/
   Rituel/Passif...), niveau, portée/durée et note d'usage en italique. Contenu **statique,
   codé en dur** dans `index.html` (pas dans `state`), une constante par personnage —
   `SPELLBOOK` pour Calix Noctavel, `DENEOR_SPELLBOOK` pour Deneor Sentariel — sélectionnée via
   `activeSpellbook()` selon `state.activeCharacterId`. Pas d'UI d'édition du contenu lui-même
   (CRUD depuis Paramètres, multi-personnage générique) ; voir `specifications_jdr_mobile_v2.md`
   section 7.3 pour la spec d'origine.
   **Grimoire de Deneor (Paladin, Serment des Anciens)** : système de préparation de sorts
   quotidienne, voir sous-section dédiée plus bas.
   Pagination par onglets sous le titre "Grimoire" : un onglet par niveau de sort de 0 à
   `maxEnabledSpellLevel()` (le plus haut niveau d'emplacement activé dans Paramètres), plus un
   onglet "Classe" en dernière position pour la section "Capacités de classe et dons" (non liée
   à un niveau). Navigation par tap sur un onglet (`data-action="grimoire-tab"`) ou par swipe
   horizontal sur la zone de contenu (`#grimoireSwipe`, listeners `touchstart`/`touchend` dans
   `bindEvents()`) : swipe vers la gauche = niveau suivant, swipe vers la droite = niveau
   précédent, sans effet de bord aux extrémités (`grimoireStep()`). Le swipe (mais pas le tap sur
   un onglet) déclenche une animation de glissement directionnelle sur `#grimoireSwipe`
   (`ui.grimoireAnimDirection`, classes CSS `grimoire-anim-next`/`-prev`, consommée une seule
   fois par `renderGrimoire()` pour ne pas se rejouer aux re-renders suivants). L'onglet actif
   (`ui.grimoireTab`, état éphémère) est recalé sur 0 si le niveau affiché n'existe plus après
   un changement de config dans Paramètres (ex. désactivation du niveau en cours de visionnage).
   Filtres par type sur la même ligne que le titre "Grimoire", alignés à droite : Action / Bonus
   / Réaction (`GRIMOIRE_FILTERS`, chips `data-action="toggle-grimoire-filter"`). Plusieurs
   filtres actifs se combinent en OR (`spellMatchesGrimoireFilters()`) ; aucun filtre actif =
   tout afficher. L'état (`ui.grimoireFilters`, éphémère) est conservé en changeant d'onglet de
   niveau puisqu'il n'est jamais réinitialisé par `renderGrimoire()`/`grimoireStep()`.

   **Préparation de sorts (Deneor uniquement)** — contrairement à Calix, le Grimoire de Deneor
   n'affiche pas tout `DENEOR_SPELLBOOK` en lecture seule : dans `renderGrimoire()`, les sections
   autres que `classe` sont filtrées aux sorts ayant `alwaysAvailable: true` (sorts de serment,
   toujours disponibles) ou présents dans `profile().preparedSpells` (tableau de noms de sorts,
   persistant, migré par `sanitizeProfile()`) ; la section `classe` (capacités de classe, de
   serment, et la card `SERMENT` de résumé des préceptes) reste toujours affichée en entier.
   Une page dédiée `renderGrimoirePrepare()` (`view: 'grimoire-prepare'`) permet de choisir les
   sorts préparés, sans plafond imposé : elle réutilise les mêmes onglets de niveau
   (`grimoireTabs()`/`ui.grimoireTab`/`grimoireStep()`, y compris le swipe, `#grimoireSwipe`
   partagé puisque les deux pages ne sont jamais montées en même temps) mais affiche tous les
   sorts du niveau (pas seulement ceux préparés), rendus par `renderGrimoireSection(section,
   spells, true)` : les sorts `alwaysAvailable` portent un badge "Toujours disponible" et ne sont
   pas cliquables, les autres togglent leur présence dans `preparedSpells` au tap
   (`data-action="toggle-prepared-spell"`) avec une mise en évidence (bordure + fond teinté). Un
   compteur mis en valeur (fond plein) affiche `preparedSpells.length` en haut à droite de
   l'en-tête. Filtres dédiés `PREPARE_FILTERS` (état `ui.prepareFilters`, éphémère) : Action /
   Bonus / **Classe** — remplace Réaction (aucun sort de ce type chez Deneor) ; le chip Classe
   révèle la section `classe` (lecture seule) sous l'onglet de niveau courant sans en changer
   (no-op si on est déjà sur l'onglet Classe).
   Trois points d'entrée vers cette page, chacun fixant `ui.grimoirePrepareBackView` pour que le
   bouton retour revienne au bon endroit : le bouton flottant `#grimoirePrepareFab` en bas du
   Grimoire (visible seulement en haut de la liste, listener de scroll sur `#grimoireSwipe` dans
   `bindEvents()` qui bascule directement `style.opacity`/`pointerEvents` sans `render()`), le
   bouton "Préparer mes sorts" de `renderSettingsGrimoire()` (qui bascule aussi sur son contenu
   Deneor selon `state.activeCharacterId`), et le bouton "Se reposer et préparer des sorts" de la
   modale Repos (Deneor uniquement, `confirmRestAndPrepare()` — factorisé avec `confirmRest()` via
   `applyRestChecks()` commun).
4. **Paramètres** — depuis juillet 2026, `renderSettings()` n'affiche plus qu'un menu de trois
   boutons (`data-action="nav"`, réutilise le pattern générique de navigation) qui renvoient
   chacun vers une sous-page dédiée. Chaque sous-page a son propre `view` (`settings-character` /
   `settings-grimoire` / `settings-app`) et affiche en haut un bouton retour
   (`renderSettingsHeader()`) qui repositionne `ui.view = 'settings'` (retour au menu, pas au
   Tracker). Dans la barre de navigation basse, l'onglet Paramètres reste en surbrillance tant
   que `ui.view` commence par `"settings"` (menu ou n'importe quelle sous-page). Pas de
   multi-personnage à ce stade : cette restructuration reste mono-profil (pas de sélecteur de
   personnage, pas de changement de la structure de `state`).
   - **Paramétrer le Personnage** (`renderSettingsCharacter()`) — nom du personnage,
     CA/Initiative/Déplacement (`data-action="combat-input"`, Initiative signée via
     `formatSigned()`, CA/Déplacement non signés), config des emplacements de sorts et des
     ressources de classe, saisie des caractéristiques/compétences. Contenu en flux continu, sans
     titres de sous-section, séparé par de simples `<hr>` légers (`SETTINGS_SEPARATOR`) entre les
     blocs.
     Les emplacements de sorts s'activent dans l'ordre croissant : impossible d'activer un niveau
     si un niveau inférieur est désactivé (message d'erreur `ui.settingsLevelError`, pas
     d'auto-activation en cascade). Désactiver un niveau qui a des niveaux supérieurs actifs ouvre
     une dialog de confirmation (`ui.disableLevelDialog`) car ces niveaux supérieurs seront
     désactivés en cascade ; désactiver le niveau actif le plus haut ne demande pas de
     confirmation.
     **Ressource(s) de classe** : `profile.classResources` est un tableau (0..n éléments, pas de
     limite) d'objets `{ id, label, max, used[] }` — chacun avec son propre libellé et son propre
     nombre d'emplacements, éditables en ligne (`data-action="class-resource-label"` /
     `"class-resource-max"`), plus un bouton "+ Ajouter une ressource"
     (`data-action="add-class-resource"`, crée via `makeClassResource()`/`generateId()`) et un
     bouton de suppression par ligne (`data-action="remove-class-resource"`). Il n'y a plus de
     notion d'« activer » : une ressource existe dans le tableau ou n'existe pas. La modale Repos
     réinitialise tous les `used[]` de `classResources` d'un coup (case "Ressource(s) de classe").
     `sanitizeProfile()` migre automatiquement l'ancien format mono-ressource
     (`profile.classResource: { enabled, label, max, used }`) vers `classResources` — appliqué à
     chaque chargement, aussi bien au profil actif (`state.profiles[]`) qu'aux instantanés
     `savedProfile` de chaque personnage (voir section Personnages ci-dessous).
   - **Paramétrer le Grimoire** (`renderSettingsGrimoire()`) — branché sur
     `state.activeCharacterId` : pour Calix, contenu inchangé et non éditable (`SPELLBOOK` fixe,
     emplacement UI "à venir" pour un futur sélecteur connus/préparés côté Calix — non développé).
     Pour Deneor, affiche un résumé du système de préparation et un bouton "Préparer mes sorts"
     vers `renderGrimoirePrepare()` (voir section Grimoire plus haut).
   - **Paramétrer l'application** (`renderSettingsApp()`, volontairement pas nommée "profil" —
     ce terme désigne déjà le personnage, `state.profile`/`state.profiles[]`) — ne contient plus
     qu'une rubrique "Sauvegarde" avec un bouton "Charger un personnage" (`data-action="nav"
     data-view="settings-load-character"`), qui donne accès à l'export/import JSON par personnage,
     voir section Personnages ci-dessous. Plus de toggle de thème ici : le thème suit désormais le
     personnage chargé, voir section Thèmes (Calix / Deneor) plus bas.

## Personnages (Calix / Deneor)

Système à **deux emplacements de personnage fixes** (pas de CRUD générique, pas d'ajout/suppression
de personnage) : `state.characters` est un objet `{ calix: {...}, deneor: {...} }`, chaque entrée
`{ id, name, subtitle, level, portrait, theme, savedProfile }` où `savedProfile` est un instantané
complet d'un profil (même forme que `defaultProfile()`) et `theme` (`'calix'` | `'deneor'`) est le
thème visuel associé à ce personnage (voir section Thèmes (Calix / Deneor) plus bas).
`state.activeCharacterId` (`'calix'` | `'deneor'`) indique lequel des deux est actuellement
**chargé**. `CHARACTER_ORDER = ['calix', 'deneor']` fixe l'ordre d'affichage.

Le profil réellement affiché/édité dans toute l'app reste `state.profiles[state.activeProfileIndex]`
(inchangé, lu via `profile()`) : c'est une **copie de travail**, distincte des instantanés
`savedProfile`. Les deux ne sont jamais le même objet en mémoire (clonage systématique via
`cloneDeep()`, un round-trip JSON) pour permettre un aller-retour explicite :

- **Charger un personnage** (`renderSettingsLoadCharacter()`, `view: 'settings-load-character'`,
  accessible depuis Paramétrer l'application) — carrousel plein écran (un personnage affiché à la
  fois, portrait + nom + sous-titre + niveau si défini) avec navigation par flèches
  (`data-action="character-carousel-step"`) ou glissé façon "carte à jouer" sur
  `#characterCarouselSwipe`, implémenté via Pointer Events (souris **et** tactile, pas seulement
  tactile) dans `bindEvents()` : la carte suit le curseur/doigt 1:1 pendant le drag (rotation +
  fondu proportionnels à la distance), avec de la résistance si on dépasse le premier/dernier
  personnage. Au relâché, au-delà d'un seuil de 90px la carte part en vol vers le bord (transition
  de 360ms) puis `characterCarouselStep()` s'exécute et la carte suivante entre avec l'animation
  CSS `character-anim-next`/`-prev` (650ms, plus lente que celle du Grimoire à 420ms, pour un effet
  de feuilletage posé) ; en dessous du seuil, la carte revient se recaler avec la même transition
  de 360ms. Un badge "Personnage chargé" s'affiche sur la carte si `state.activeCharacterId`
  correspond au personnage affiché.
  Deux boutons en bas :
  - **Charger ce personnage** (`data-action="load-character"`) — toujours actif : clone
    `character.savedProfile` dans `state.profiles[state.activeProfileIndex]`, bascule
    `state.activeCharacterId`, applique le thème du personnage (`applyTheme()`, voir section
    Thèmes (Calix / Deneor) plus bas), puis renvoie directement sur le Tracker (`ui.view =
    'tracker'`).
  - **Mettre à jour le personnage** (`data-action="update-character"`) — clone le profil de
    travail actuel (`profile()`) dans `character.savedProfile`, ce qui en fait les nouvelles
    valeurs par défaut de ce personnage. **Grisé/non cliquable** (`pointer-events:none`) tant que
    ce personnage n'est pas celui actuellement chargé (`state.activeCharacterId !== charId`) —
    impossible de mettre à jour un personnage sans l'avoir chargé au préalable. Une ligne d'aide
    sous ce bouton est **toujours présente** (jamais conditionnelle) pour éviter un décalage
    vertical pendant le swipe entre les deux personnages : "Chargez ce personnage avant de pouvoir
    le mettre à jour." si non chargé, "Ce personnage est actuellement chargé." sinon.
  Un message transitoire (`ui.characterNotice = { id, text }`, propre au personnage affiché dans
  le carrousel) s'affiche après une mise à jour ou un import réussis ("Personnage mis à jour." /
  "Personnage importé."), effacé au moindre changement d'onglet du carrousel.
  Sous ces deux boutons, un bloc "Sauvegarde JSON de {nom}" propose l'export/import du **profil
  complet** du personnage affiché (indépendamment de `state.activeCharacterId` — export/import
  fonctionnent aussi bien sur le personnage chargé que sur l'autre) :
  - **Exporter** (`data-action="export-character"`, fonction `exportCharacterJson()`) — sérialise
    `character.savedProfile` en JSON (`JSON.stringify(..., null, 2)`) et déclenche un téléchargement
    via un `<a download>` éphémère (`cantrip-{charId}.json`). Aucun appel réseau, tout se passe
    côté client via un `Blob`/`URL.createObjectURL`.
  - **Importer** (`data-action="import-character"`, fonction `importCharacterJson()`) — ouvre un
    `<input type="file">` créé dynamiquement (pas de champ persistant dans le template, pour éviter
    qu'il ne soit détruit au prochain re-render), lit le fichier via `FileReader`, `JSON.parse` puis
    `sanitizeProfile()` (qui comble maintenant aussi `hp`, `characterName`, `concentration`,
    `inspiration` et `spellLevels` en plus des champs déjà couverts, pour encaisser un JSON
    partiel/corrompu — c'est le seul point d'entrée de données externes non fiables de l'app) avant
    d'écraser `character.savedProfile`. Si le personnage importé est celui actuellement chargé
    (`state.activeCharacterId === charId`), la copie de travail (`state.profiles[
    state.activeProfileIndex]`) est aussi mise à jour immédiatement pour que le Tracker reflète le
    changement sans repasser par "Charger ce personnage". En cas de JSON invalide, message d'erreur
    transitoire (`ui.characterImportError`, même durée de vie que `ui.characterNotice`) sans toucher
    à l'état existant.
  Le Grimoire affiché dépend de `state.activeCharacterId` (`activeSpellbook()`, voir section
  Grimoire plus haut) : charger Deneor bascule sur `DENEOR_SPELLBOOK` et son système de
  préparation de sorts, indépendant du contenu de Calix.

`loadState()` migre automatiquement toute sauvegarde antérieure à ce système (`parsed.characters`
absent) : le profil unique existant devient l'instantané de Calix (aucune perte de données) et
Deneor démarre sur un `defaultProfile()` vierge ; `activeCharacterId` est alors mis à `'calix'`.

## Décisions de conception (à respecter, divergent parfois du cahier des charges)

Le développement s'est appuyé sur deux sources parfois contradictoires : le cahier des charges
(`specifications_jdr_mobile_v2.md`) et une maquette visuelle Claude Design ("2a — Parchemin
révisé"). En cas de divergence, les arbitrages suivants ont été retenus :

- **Dégâts/soins** : tap direct sur `−`/`+` (1 point par tap), **pas** de champ de saisie de
  quantité — choix explicite, contrairement au doc de specs qui décrivait un champ + bouton.
- **Modale "Repos" unique** (pas de Repos Court/Long séparés) : la case "Total de PV", si
  cochée, restaure réellement les PV au max (comportement actif, pas juste un aide-mémoire).
  Ce point diverge du doc de specs et n'a pas été explicitement re-confirmé par l'utilisateur —
  à surveiller si retour futur.
- Les cases à cocher de la modale Repos sont toutes décochées par défaut à chaque ouverture.
- Les champs "Max" (sorts, ressource de classe) sont désactivés visuellement tant que le niveau
  correspondant n'est pas activé.
- Tout `<input type="text">` voit son contenu présélectionné au focus (listener délégué global
  `focusin` sur `#app`, tape directement pour remplacer la valeur), **sauf** le champ de
  recherche des compétences (`#statsSearchInput`) qui reste un filtre en direct.

## Thèmes (Calix / Deneor)

Il n'y a plus de toggle clair/sombre manuel : le thème visuel est une propriété de chaque
personnage (`state.characters[id].theme`, `'calix'` | `'deneor'`) et s'applique automatiquement
au chargement de ce personnage (`applyTheme()`, appelé au démarrage depuis `loadState()`/
`defaultState()` et lors de l'action `data-action="load-character"`). `activeTheme()` lit
`state.characters[state.activeCharacterId].theme` (repli sur `'calix'` si absent) ;
`applyTheme()` pose cette valeur sur l'attribut `data-theme` de `<html>` et met à jour le
`<meta name="theme-color">` via la table `THEME_COLOR_META`.
- **Thème Calix** (`:root`, valeurs par défaut sans attribut) — ocre/parchemin clair, hérité de
  l'ancien thème clair unique (pas blanc, pour garder l'aspect médiéval).
- **Thème Deneor** (`:root[data-theme="deneor"]`) — palette sombre vert forêt construite autour
  de Pantone 3435 C (`#154734`, utilisé comme `--border-strong`), cohérente avec le thème de
  Paladin sous Serment des Anciens de Deneor Sentariel.
Les deux jeux de variables CSS custom properties (fonds, bordures, textes, couleurs d'accent des
badges) sont définis en tête de fichier dans le bloc `<style>`. **Toute nouvelle couleur ajoutée
dans un template JS doit utiliser `var(--...)` plutôt qu'un hex en dur** pour rester compatible
avec les deux thèmes.
Le `<meta name="theme-color">` initial dans `<head>` et `background_color`/`theme_color` dans
`manifest.json` reflètent la teinte Calix par défaut (`#ecdcb0`), puisque Calix reste le
personnage chargé par défaut sur un état vierge.

## Service Worker (`sw.js`)

Stratégie réseau d'abord avec fallback cache (pas de stale-while-revalidate). `CACHE_NAME` est
versionné (`cantrip-v33` au 2026-07-12) — **incrémenter cette constante à chaque changement
significatif des assets statiques** (`index.html`, `manifest.json`, `icon.svg`,
`characters/*.jpg|png`) pour forcer l'invalidation du cache côté client.

## Déploiement

- Dépôt : https://github.com/Nerdash/cantrip (public, compte GitHub "Nerdash")
- **Prod** (GitHub Pages, HTTPS, installable en PWA) : https://nerdash.github.io/cantrip/ —
  déploiement automatique à chaque push sur `master`, qui sert directement `index.html` à la
  racine (pas de pipeline CI, pas de dossier `dist`). `.nojekyll` présent pour désactiver le
  traitement Jekyll.
- **Preprod** (Netlify) : https://ben-cantrip.netlify.app/ — déploiement automatique à chaque
  push sur `preprod`, configuré depuis le dashboard Netlify (pas de `netlify.toml` dans le
  dépôt).

## Git — spécifique à cette machine, à reconfigurer sur toute nouvelle installation

L'identité Git de ce dépôt est configurée **localement** (pas globalement), pour ne pas exposer
l'email réel dans l'historique public :
```
git config user.name "Nerdash"
git config user.email "124379495+Nerdash@users.noreply.github.com"
```
Cette config est dans `.git/config`, donc **pas transmise par un `git clone`** — à relancer sur
toute nouvelle machine avant de committer.

## Reste à faire / pistes non traitées

- Sélecteur connus/préparés pour le Grimoire de Calix : emplacement UI réservé dans
  `renderSettingsGrimoire()` mais logique non développée (Deneor a son propre système de
  préparation de sorts, voir section Grimoire plus haut — indépendant de celui-ci).
- Personnages : système actuellement limité à deux emplacements fixes (Calix/Deneor), pas de CRUD
  générique (ajout/suppression d'un personnage) — voir section Personnages plus haut.
- Génération d'un `.apk` installable : voie recommandée — passer l'URL GitHub Pages dans
  PWABuilder.com pour générer un APK signé sans installer Android Studio.
- Icône et animation de concentration : validées visuellement mais pourraient encore évoluer
  (voir historique des commits sur `concentration`).

## Fichiers sources du brief original

- `specifications_jdr_mobile_v2.md` (à la racine du dépôt) : cahier des charges fonctionnel
  complet, y compris la section 7 (évolution V3 : navigation, Stats/Skills, Grimoire) — à
  relire avant toute évolution fonctionnelle.
