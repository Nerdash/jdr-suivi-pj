# JDR Tracker — Suivi de Personnage

PWA de suivi de personnage JDR (PV, emplacements de sorts, ressource de classe, fiche
Stats/Skills, Grimoire) construite en juillet 2026. Le propriétaire travaille alternativement
sur deux PC (fixe et portable) : ce fichier sert de mémoire portable entre les deux, en
complément du dépôt Git qui est la seule source de vérité partagée entre les machines.

## Stack technique

- Fichier unique `index.html` : HTML/CSS/JS vanilla, **pas de framework, pas de build step**.
- Tailwind CSS via CDN (`<script src="https://cdn.tailwindcss.com">`) — utilisé de façon
  marginale, la majorité du style est en `style=""` inline directement dans les templates JS.
- Police Cinzel (Google Fonts) via `class="font-cinzel"`.
- Persistance : `localStorage`, clé `jdr_character_tracker_state`. Pas de backend.
- PWA : `manifest.json` + `icon.svg` + `sw.js` (service worker, stratégie réseau d'abord avec
  fallback cache — voir section Service Worker ci-dessous).

## Structure du code (`index.html`)

Application à état unique en IIFE (`(function () { 'use strict'; ... })()`), rendu par
réécriture complète de `innerHTML` à chaque changement (pas de diffing, pas de framework).

- `state` : données persistées (profils de personnage). Sauvegardé via `saveState()` à chaque
  mutation.
- `ui` : état éphémère non persisté (page active, champs en cours d'édition, recherche...).
  Voir `ui.view` pour la page courante.
- `render()` : dispatch vers `renderTracker()` / `renderStats()` / `renderGrimoire()` /
  `renderSettings()` selon `ui.view`, puis ajoute `renderBottomNav()`. Chaque page occupe toute
  la hauteur disponible (`height:100%`, pas de scroll de la page globale — seuls les conteneurs
  `[data-scroll-root]` scrollent en interne).
- `bindEvents()` : re-attache tous les event listeners après chaque re-render (délégation via
  `data-action` sur les éléments cliquables).

### Pages (barre de navigation basse, 4 onglets)

1. **Tracker** (page par défaut) — PV (bloc fusionné avec bouclier temporaire), emplacements de
   sorts groupés par niveau en badges circulaires, ressource(s) de classe (un bloc de badges par
   ressource configurée, voir Paramètres ci-dessous), marqueur de concentration, bouton "Repos"
   unique.
2. **Stats** — caractéristiques (6) et compétences (18, D&D 5e, noms français) en lecture
   seule, avec champ de recherche filtrant la liste en direct. Les valeurs sont saisies
   manuellement dans Paramètres (l'app ne calcule aucun modificateur).
3. **Grimoire** — affiche les capacités du personnage (sorts par niveau + capacités de classe et
   dons), regroupées par section avec badge de type d'action coloré (Action/Bonus/Réaction/
   Rituel/Passif...), niveau, portée/durée et note d'usage en italique. Contenu **statique,
   codé en dur** dans la constante `SPELLBOOK` (`index.html`, pas dans `state`) pour le
   personnage Calix Noctavel — pas d'UI d'édition. Une gestion générique (CRUD depuis
   Paramètres, multi-personnage) reste à faire si besoin ; voir `specifications_jdr_mobile_v2.md`
   section 7.3 pour la spec d'origine.
   Pagination par onglets sous le titre "Grimoire" : un onglet par niveau de sort de 0 à
   `maxEnabledSpellLevel()` (le plus haut niveau d'emplacement activé dans Paramètres), plus un
   onglet "Classe" en dernière position pour la section "Capacités de classe et dons" (non liée
   à un niveau). Navigation par tap sur un onglet (`data-action="grimoire-tab"`) ou par swipe
   horizontal sur la zone de contenu (`#grimoireSwipe`, listeners `touchstart`/`touchend` dans
   `bindEvents()`) : swipe vers la gauche = niveau suivant, swipe vers la droite = niveau
   précédent, sans effet de bord aux extrémités (`grimoireStep()`). L'onglet actif
   (`ui.grimoireTab`, état éphémère) est recalé sur 0 si le niveau affiché n'existe plus après
   un changement de config dans Paramètres (ex. désactivation du niveau en cours de visionnage).
4. **Paramètres** — nom du personnage, config des emplacements de sorts et des ressources de
   classe, saisie des caractéristiques/compétences, export/import JSON.
   Les emplacements de sorts s'activent dans l'ordre croissant : impossible d'activer un niveau
   si un niveau inférieur est désactivé (message d'erreur `ui.settingsLevelError`, pas
   d'auto-activation en cascade). Désactiver un niveau qui a des niveaux supérieurs actifs ouvre
   une dialog de confirmation (`ui.disableLevelDialog`) car ces niveaux supérieurs seront
   désactivés en cascade ; désactiver le niveau actif le plus haut ne demande pas de
   confirmation.
   **Ressource(s) de classe** : `profile.classResources` est un tableau (0..n éléments, pas de
   limite) d'objets `{ id, label, max, used[] }` — chacun avec son propre libellé et son propre
   nombre d'emplacements, éditables en ligne dans Paramètres (`data-action="class-resource-label"`
   / `"class-resource-max"`), plus un bouton "+ Ajouter une ressource"
   (`data-action="add-class-resource"`, crée via `makeClassResource()`/`generateId()`) et un
   bouton de suppression par ligne (`data-action="remove-class-resource"`). Il n'y a plus de
   notion d'« activer » : une ressource existe dans le tableau ou n'existe pas. La modale Repos
   réinitialise tous les `used[]` de `classResources` d'un coup (case "Ressource(s) de classe").
   `sanitizeProfile()` migre automatiquement l'ancien format mono-ressource
   (`profile.classResource: { enabled, label, max, used }`) vers `classResources` au chargement
   et à l'import JSON — les deux formats sont acceptés par `isValidProfile()` pour la
   rétrocompatibilité des sauvegardes exportées avant ce changement.

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

## Thème clair/sombre

L'app supporte un thème sombre (par défaut) et un thème clair "ocre/parchemin" (pas blanc, pour
garder l'aspect médiéval), basculable via un toggle dans Paramètres. Persisté dans
`state.theme` (`'dark'` | `'light'`), appliqué via l'attribut `data-theme` sur `<html>`
(`applyTheme()`), qui pilote un jeu de variables CSS custom properties définies dans `:root` /
`:root[data-theme="light"]` (fonds, bordures, textes, couleurs d'accent des badges). **Toute
nouvelle couleur ajoutée dans un template JS doit utiliser `var(--...)` plutôt qu'un hex en dur**
pour rester compatible avec les deux thèmes — voir le bloc `<style>` en tête de fichier pour la
liste des variables disponibles.

## Service Worker (`sw.js`)

Stratégie réseau d'abord avec fallback cache (pas de stale-while-revalidate). `CACHE_NAME` est
versionné (`jdr-tracker-v5` au 2026-07-11) — **incrémenter cette constante à chaque changement
significatif des assets statiques** (`index.html`, `manifest.json`, `icon.svg`) pour forcer
l'invalidation du cache côté client.

## Déploiement

- Dépôt : https://github.com/Nerdash/jdr-suivi-pj (public, compte GitHub "Nerdash")
- App en ligne (GitHub Pages, HTTPS, installable en PWA) : https://nerdash.github.io/jdr-suivi-pj/
- Déploiement automatique : push sur `master` → GitHub Pages sert directement `index.html` à la
  racine (pas de pipeline CI, pas de dossier `dist`).
- `.nojekyll` présent pour désactiver le traitement Jekyll par GitHub Pages.

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

- Grimoire : gestion générique (CRUD, multi-personnage) si besoin au-delà du contenu statique
  actuel de Calix Noctavel — voir section Grimoire plus haut.
- Génération d'un `.apk` installable : voie recommandée — passer l'URL GitHub Pages dans
  PWABuilder.com pour générer un APK signé sans installer Android Studio.
- Icône et animation de concentration : validées visuellement mais pourraient encore évoluer
  (voir historique des commits sur `concentration`).

## Fichiers sources du brief original

- `specifications_jdr_mobile_v2.md` (à la racine du dépôt) : cahier des charges fonctionnel
  complet, y compris la section 7 (évolution V3 : navigation, Stats/Skills, Grimoire) — à
  relire avant toute évolution fonctionnelle.
