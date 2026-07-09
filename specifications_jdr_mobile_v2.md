# Spécifications - Application de Suivi JDR (PV & Emplacements de Sorts)

Version 3 - Document de référence pour le développement.
Ce document reprend les spécifications initiales, complétées et arbitrées suite aux échanges de cadrage. La section 7 couvre l'évolution ajoutée après la mise en production de la V1 (application fonctionnelle) : navigation multi-pages, fiche de personnage (Skills) et Grimoire.

---

## 1. Architecture Générale et Pile Technique

| Élément | Choix retenu |
|---|---|
| Architecture | Fichier unique (Single-File Architecture) : un seul `.html` contenant HTML, CSS et JavaScript |
| Langages | HTML5 et JavaScript ES6+ natif, sans framework ni outil de build |
| Style | Tailwind CSS via CDN officiel |
| Persistance | `localStorage` (API synchrone du navigateur) |
| Déploiement Android | Progressive Web App (PWA) - ouverture du fichier dans Chrome puis "Ajouter à l'écran d'accueil" |

**Justification du choix PWA** : aucune compilation, aucun SDK Android, aucun outil de build nécessaire. Limite connue et acceptée : l'application reste liée au navigateur Chrome (pas de fichier `.apk` autonome dans cette version).

**Amélioration optionnelle (non bloquante)** : ajout possible d'un `manifest.json` et d'une icône dédiée pour un rendu plus proche d'une application native une fois ajoutée à l'écran d'accueil. À traiter en phase 2 si souhaité.

---

## 2. Spécifications Fonctionnelles

### 2.1. Gestion des Points de Vie (PV)
- Affichage proéminent : `PV Actuels / PV Max`
- Champ de saisie numérique associé à deux boutons : `-` (Dégâts) et `+` (Soins)
- Champ dédié aux PV temporaires, avec logique d'absorption prioritaire en cas de dégâts

### 2.2. Gestion des Emplacements de Sorts (Spell Slots)
- Profil d'emplacements configurable **du niveau 1 au niveau 9**
- Chaque niveau peut être **activé ou désactivé individuellement depuis l'écran de paramètres** (ex : un personnage sans sorts de niveau élevé n'affiche pas ces niveaux sur l'écran principal)
- Chaque emplacement actif est un badge circulaire indiquant son état (disponible ou consommé)
- Interactivité bilatérale : clic sur une puce disponible = consommation ; clic sur une puce consommée = restauration (correction rapide d'erreur)

### 2.3. Gestion des Repos
- **Repos Long** : réinitialise les PV actuels au maximum, remet à zéro les PV temporaires, restaure tous les emplacements de sorts à leur maximum
- **Repos Court** : déclenche l'ouverture d'une pop-up de sélection listant les éléments réinitialisables (PV temporaires, emplacements de sorts par niveau). L'utilisateur coche les éléments à restaurer, valide, puis seuls les éléments cochés sont réinitialisés. Aucune restauration automatique ou implicite.

### 2.4. Profil de Personnage
- **Un seul profil actif dans cette version.**
- Le modèle de données doit toutefois être conçu pour permettre l'ajout ultérieur de plusieurs profils sélectionnables (ex : gestion de plusieurs personnages joués), sans nécessiter de refonte complète de la structure.
- Le nom du personnage (`characterName`) est saisi et modifiable depuis l'écran de paramètres (voir 2.6).

### 2.5bis. Ressource de Classe (Générique)
- Fonctionnalité additionnelle, sur le même principe que les emplacements de sorts : un stock d'éléments consommables (ex : Points de Ki, Inspiration Bardique, Dés de Supériorité, etc.)
- Activable ou désactivable depuis l'écran de paramètres
- Libellé personnalisable par l'utilisateur (champ texte libre, ex : "Points de Ki")
- Représentation visuelle identique aux emplacements de sorts (badges circulaires, current/max)
- Interactivité bilatérale identique : clic pour consommer, clic pour restaurer

### 2.5. Sauvegarde et Export des Données
- Persistance principale : `localStorage`, clé `jdr_character_tracker_state`
- **Ajout requis** : fonctions d'export et d'import au format JSON, pour permettre à l'utilisateur de sauvegarder manuellement son état (fichier téléchargeable) et de le restaurer en cas de perte des données locales (changement de téléphone, nettoyage du navigateur, etc.)
- Emplacement dans l'interface : voir 2.6 (écran de paramètres)

### 2.6. Écran de Paramètres (Page Secondaire)
- Accès via une icône de roue crantée, positionnée sur l'écran principal sans perturber la disposition existante (coin discret, ex : haut de l'écran)
- Cet écran constitue une deuxième vue distincte de l'écran principal de suivi
- Contenu prévu dès cette version :
  - Champ de saisie/modification du nom du personnage
  - Liste des niveaux de sorts (1 à 9), chaque ligne avec un interrupteur Activé/Désactivé et un champ numérique Max (inactif tant que le niveau n'est pas activé)
  - Section Ressource de classe : interrupteur Activé/Désactivé, champ de libellé personnalisable, champ numérique Max
  - Boutons d'export et d'import JSON
- Vocation évolutive : cet écran servira de point d'ancrage pour les futures fonctionnalités, sans devoir remanier l'écran principal à chaque ajout

---

## 3. Modèle de Données (State Management)

```json
{
  "characterName": "Nom du Personnage",
  "hp": {
    "current": 30,
    "max": 30,
    "temp": 0
  },
  "slots": {
    "1": { "enabled": true, "current": 4, "max": 4 },
    "2": { "enabled": true, "current": 3, "max": 3 },
    "3": { "enabled": true, "current": 2, "max": 2 },
    "4": { "enabled": false, "current": 0, "max": 0 },
    "5": { "enabled": false, "current": 0, "max": 0 },
    "6": { "enabled": false, "current": 0, "max": 0 },
    "7": { "enabled": false, "current": 0, "max": 0 },
    "8": { "enabled": false, "current": 0, "max": 0 },
    "9": { "enabled": false, "current": 0, "max": 0 }
  },
  "classResource": {
    "enabled": false,
    "label": "Ressource de Classe",
    "current": 0,
    "max": 0
  }
}
```

Remarque : le champ `enabled` par niveau/ressource permet à l'écran de paramètres de masquer ou afficher les éléments correspondants sur l'écran principal, sans supprimer la donnée sous-jacente (utile si l'utilisateur réactive un niveau plus tard).

Remarque structurelle : conserver ce modèle sous une clé racine identifiable (ex : `profiles: [ { ...état ci-dessus } ]`) afin de faciliter l'extension future vers plusieurs profils, même si l'interface n'exploite qu'un seul profil pour l'instant.

---

## 4. Algorithmes et Règles de Gestion Métier

### 4.1. Application des dégâts (bouton `-`)
Soit X la valeur entière positive saisie :
1. Si `hp.temp > 0` :
   - Si `hp.temp >= X` : `hp.temp -= X` ; `X = 0`
   - Sinon : `X -= hp.temp` ; `hp.temp = 0`
2. Si `X > 0` : `hp.current -= X` (plancher à 0)
3. Réinitialiser le champ de saisie
4. Déclencher la sauvegarde locale

### 4.2. Application des soins (bouton `+`)
Soit X la valeur entière positive saisie :
1. `hp.current += X` (plafond à `hp.max`)
2. Réinitialiser le champ de saisie
3. Déclencher la sauvegarde locale

### 4.3. Persistance et Auto-Save
Toute modification d'état (PV, sorts, repos) déclenche systématiquement une sérialisation JSON et un enregistrement dans `localStorage` sous la clé `jdr_character_tracker_state`.

### 4.4. Export / Import JSON (ajout)
- **Export** : génération d'un fichier `.json` téléchargeable contenant l'état courant, horodaté dans le nom de fichier (ex : `jdr_save_2026-07-07.json`)
- **Import** : lecture d'un fichier `.json` fourni par l'utilisateur, validation basique de la structure attendue, puis remplacement de l'état courant et sauvegarde dans le `localStorage`
- **Gestion d'erreur** : si le fichier importé est invalide ou mal structuré, affichage d'un message d'erreur simple (ex : "Fichier invalide"). Aucun remplacement de l'état courant n'est effectué dans ce cas ; l'état existant reste inchangé.

### 4.5. Repos Court - Pop-up de Sélection
- Ouverture d'une fenêtre modale au clic sur le bouton "Repos Court"
- Quatre catégories proposées, chacune associée à une case à cocher, non cochée par défaut :
  1. **Total de PV** : ne déclenche aucun calcul automatique (les PV récupérés lors d'un repos court sont déterminés par des dés lancés IRL) ; cette case sert d'aide-mémoire, l'ajustement effectif se fait ensuite manuellement via le champ PV existant
  2. **Emplacements de sorts** : restaure au maximum uniquement les niveaux actifs (`enabled: true`)
  3. **PV temporaires** : remet `hp.temp` à 0
  4. **Ressource de classe** : restaure `classResource.current` à `classResource.max`, uniquement si la ressource est activée
- Un bouton de validation applique la restauration aux seules catégories cochées (hors catégorie 1, qui ne modifie rien automatiquement), puis ferme la pop-up et déclenche la sauvegarde locale
- Annulation possible sans aucune modification de l'état

---

## 5. Spécifications Ergonomiques et UX Mobile

- Écran unique verrouillé sur la hauteur de vue (pas de scroll vertical pour les boutons principaux)
- Zones tactiles d'au moins 44x44 px pour tous les boutons interactifs
- Mode sombre natif et obligatoire (fond Slate-900/Zinc-900, texte Slate-100)
- Champs numériques avec `inputmode="numeric"` pour forcer le pavé numérique Android
- **Éléments à nombre variable** (emplacements de sorts actifs, ressource de classe) : affichés en ligne horizontale avec défilement horizontal si nécessaire, sans impacter la hauteur verrouillée de l'écran ; la ligne de ressource de classe est masquée entièrement si la fonctionnalité est désactivée
- **Pop-up de Repos Court** : contenu limité à 4 lignes fixes (une par catégorie), sans scroll interne

---

## 6. Points en Suspens

Aucun point bloquant sur la V1 (voir section 7 pour les points en suspens de l'évolution V3).

---

## 7. Évolution V3 - Navigation, Fiche de Personnage (Skills) et Grimoire

### 7.1. Barre de Navigation
- Ajout d'une barre de navigation à 3 pages :
  - **Skills** (gauche)
  - **Tracker** (centre - page actuelle, PV/sorts/ressource/repos, ouverte par défaut au lancement)
  - **Grimoire** (droite)
- La barre de navigation ne doit pas remettre en cause la contrainte d'écran verrouillé sans scroll sur chaque page

### 7.2. Page Skills (Pense-bête caractéristiques et compétences)
- **Haut de page** : les 6 modificateurs de caractéristiques principales (Force, Dextérité, Constitution, Intelligence, Sagesse, Charisme), affichage en lecture seule
- **Sous cette section** : liste des compétences (les 18 compétences standards de D&D 5e, noms en français), chacune affichant son modificateur final, en lecture seule
- **Champ de recherche** en haut de la liste des compétences : filtre en direct la liste par nom français au fur et à mesure de la saisie
- **Saisie des valeurs** : les modificateurs (caractéristiques et compétences) ne sont pas calculés par l'application. Ils sont saisis manuellement par l'utilisateur dans l'écran de paramètres (voir 7.4), sous forme de valeur finale déjà calculée (ex : `+5`, `-1`), en recopiant sa fiche de personnage papier/Notion. Aucune logique de maîtrise ou d'expertise n'est calculée par l'application.

Liste des 18 compétences (référence) :
Acrobaties, Arcanes, Athlétisme, Discrétion, Dressage, Escamotage, Histoire, Intimidation, Intuition, Investigation, Médecine, Nature, Perception, Persuasion, Religion, Représentation, Survie, Tromperie.

### 7.3. Page Grimoire (Différée - Work in Progress)
- **Priorité de développement : différée.** La page Skills est développée en premier ; le Grimoire sera traité dans une itération suivante.
- En attendant, la page Grimoire existe dans la navigation mais affiche un état provisoire simple : mention "Work in progress" (ou équivalent visuel cohérent avec le thème sombre de l'application), sans fonctionnalité de gestion des sorts pour l'instant
- Spécifications fonctionnelles cibles (pour la prochaine itération, non développées maintenant) : liste simplifiée des sorts connus/disponibles, affichant pour chacun **nom, niveau, mention concentration (oui/non), description courte**, avec gestion (ajout/modification/suppression) déportée dans l'écran de paramètres

### 7.4. Écran de Paramètres - Compléments
En complément du contenu existant (section 2.6), l'écran de paramètres intègre désormais :
- 6 champs numériques pour les modificateurs de caractéristiques principales
- 18 champs numériques pour les modificateurs de compétences (regroupés, avec les mêmes noms français que la page Skills)
- La gestion du Grimoire (ajout/modification/suppression de sorts) est **différée** : non incluse dans cette itération (voir 7.3)

### 7.5. Marqueur de Concentration
- Interrupteur manuel simple (actif/inactif), sans lien automatique avec un sort précis du Grimoire
- Présent à deux emplacements : sur la page **Tracker** (à proximité du bloc PV) et sur la page **Grimoire**
- Les deux interrupteurs reflètent le même état (une seule donnée partagée)
- Quand actif : icône dédiée affichée avec une animation discrète (ex : pulsation légère) pour attirer l'attention visuellement, sans gêner la lisibilité du reste de l'écran
- Choix de l'icône : non tranché à ce stade (voir points en suspens ci-dessous)

### 7.6. Modèle de Données - Compléments
```json
{
  "abilityScores": {
    "force": 0,
    "dexterite": 0,
    "constitution": 0,
    "intelligence": 0,
    "sagesse": 0,
    "charisme": 0
  },
  "skills": [
    { "name": "Acrobaties", "modifier": 0 },
    { "name": "Arcanes", "modifier": 0 },
    { "name": "Athlétisme", "modifier": 0 },
    { "name": "Discrétion", "modifier": 0 },
    { "name": "Dressage", "modifier": 0 },
    { "name": "Escamotage", "modifier": 0 },
    { "name": "Histoire", "modifier": 0 },
    { "name": "Intimidation", "modifier": 0 },
    { "name": "Intuition", "modifier": 0 },
    { "name": "Investigation", "modifier": 0 },
    { "name": "Médecine", "modifier": 0 },
    { "name": "Nature", "modifier": 0 },
    { "name": "Perception", "modifier": 0 },
    { "name": "Persuasion", "modifier": 0 },
    { "name": "Religion", "modifier": 0 },
    { "name": "Représentation", "modifier": 0 },
    { "name": "Survie", "modifier": 0 },
    { "name": "Tromperie", "modifier": 0 }
  ],
  "spells": [
    { "name": "", "level": 0, "concentration": false, "description": "" }
  ],
  "concentration": {
    "active": false
  }
}
```

### 7.7. Points en Suspens (V3)
1. Icône et animation de concentration : à définir visuellement via Claude Design, puis transfert vers Claude Code (hors texte de ce document)
2. Grimoire : différé à une itération suivante (voir 7.3) - à re-spécifier en détail une fois la page Skills livrée

---

*Document prêt à servir de base de travail pour le développement du fichier `.html` unique, en assistance directe avec Claude.*
