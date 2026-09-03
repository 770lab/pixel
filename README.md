# 770lab.com/pixel — création de fiches Pixel CRM

Tu colles du texte, tu ajoutes des captures / PDF, Claude en sort une fiche structurée, tu vérifies, tu cliques **Créer la fiche sur Pixel** : l'extension Chrome ouvre Pixel CRM (avec ta session déjà connectée) et remplit tout.

```
texte + JPG/PNG/PDF ──▶ 770lab.com/pixel ──▶ Cloud Function (Claude API) ──▶ fiche JSON
                                   │  (relecture / correction)
                                   ▼
                    extension Chrome "770lab → Pixel CRM"
                                   │  ouvre crm.pixel-crm.com/…/fiche/create
                                   ▼
        type de dossier → étape 1 (bypass) → 2 client → 3 éligibilité → 4 attributions → Enregistrer
                                   ▼
                  fiche N_EditMain : deal STEREM + surface + type logement + réf. devis → Enregistrer
                                   ▼
                    n° HMH-2026-xxxx renvoyé sur la page + historique Firestore
```

Trois morceaux, trois dossiers :

| Dossier | Rôle | Hébergement |
|---|---|---|
| `web/index.html` | le site (single-file, vanilla JS, Firebase compat) | GitHub Pages `770lab.com/pixel` |
| `functions/` | Cloud Function `extractFiche` (Claude API, texte + images + PDF → JSON) | Firebase Functions gen2, `europe-west1` |
| `extension/` | extension Chrome MV3 qui pilote Pixel CRM | ton Chrome (mode développeur) |

---

## 1. Firebase (une fois)

1. Console Firebase → nouveau projet (ou un existant) → plan **Blaze** (obligatoire pour les Cloud Functions ; ton volume restera dans le gratuit).
2. **Authentication** → activer *E-mail / mot de passe* → créer ton utilisateur (chabadclub770@gmail.com).
3. **Firestore** → créer la base (mode production, région `eur3` ou `europe-west1`).
4. **Paramètres du projet → Vos applications → Web** → copier la config et la coller dans `FIREBASE_CONFIG` en haut du `<script>` de `web/index.html`.
5. Mettre le project id dans `.firebaserc`.

```bash
npm i -g firebase-tools
firebase login
cd pixel-fiche
firebase use REMPLACER-PAR-TON-PROJECT-ID

# clé API Anthropic (stockée en secret, jamais dans le code)
firebase functions:secrets:set ANTHROPIC_API_KEY

# optionnel : functions/.env  (copier .env.example) — emails autorisés, modèle
cd functions && npm install && cd ..

firebase deploy --only functions,firestore:rules
```

Après le deploy, la fonction est appelée par le site via `firebase.app().functions("europe-west1").httpsCallable("extractFiche")` — rien d'autre à configurer.

Modèle par défaut : `claude-sonnet-5`. Pour diviser le coût par ~3 : `CLAUDE_MODEL=claude-haiku-4-5-20251001` dans `functions/.env`.

## 2. Le site sur 770lab.com/pixel

Le site est un seul fichier `web/index.html`. Dans le repo GitHub Pages de 770lab.com, crée le dossier `pixel/` et mets-y `index.html` → l'URL devient `https://770lab.com/pixel/`.

Dans la console Firebase → **Authentication → Settings → Domaines autorisés**, ajoute `770lab.com`.

(Alternative : `firebase deploy --only hosting` publie le même fichier sur `PROJECT.web.app`.)

## 3. L'extension Chrome

1. `chrome://extensions` → activer **Mode développeur** (en haut à droite).
2. **Charger l'extension non empaquetée** → choisir le dossier `extension/`.
3. Recharger `770lab.com/pixel` : le badge en haut à droite passe à **extension v1.0.0**.

Elle n'a besoin d'aucun identifiant : elle utilise ta session Pixel déjà ouverte dans Chrome. Si Pixel demande une reconnexion, la création s'arrête avec un message ; reconnecte-toi et relance.

Le popup de l'extension (icône **P** verte) montre le dernier statut et un bouton *Réinitialiser* si un job reste bloqué.

## 4. Utilisation

1. Colle le texte et/ou dépose captures + PDF (Ctrl+V colle directement une capture).
2. **Analyser avec Claude** → la fiche apparaît, avec la liste **À compléter** (email tronqué, etc.).
3. Corrige ce qu'il faut (les emails tronqués ne sont jamais devinés).
4. **Créer la fiche sur Pixel** → un onglet Pixel s'ouvre, un bandeau vert affiche la progression, puis le n° de dossier revient sur la page.

Coche **Test à blanc** pour voir le remplissage sans rien enregistrer (ferme ensuite l'onglet Pixel).

Le champ **Deal** (défaut `STEREM`) est le mot-clé cherché dans la liste des deals de la fiche.

## 5. Ce que l'extension remplit dans Pixel

| Étape | Champs |
|---|---|
| Type de dossier | Opérations diverses |
| 1 Avis fiscal | *Bypasser cette étape* |
| 2 Client | civilité, nom, prénom, (co-titulaire), adresse, complément, CP, ville, email, mobile, fixe |
| 3 Éligibilité | type chauffage, énergie, occupation, parcelle, revenu fiscal, foyers, personnes, âge du bâtiment, travaux demandés → lit la précarité calculée |
| 4 Attributions | commentaire (devis, commercial, surface, MPR, équipements) |
| Fiche | deal, surface habitable, type logement, réf. dossier externe = n° devis, date d'édition devis |

Non rempli automatiquement : **Commercial terrain** (liste Pixel ≠ tes commerciaux), produits / lignes de devis, note de dimensionnement, n° d'agrément coup de pouce (voir la skill `pixel-creer-fiche` dans Claude pour ces suites).

## 6. Si Pixel change son interface

Tous les identifiants de champs sont dans `extension/pixel-fill.js` (ids `Fiche_VM_*`, `FicheISO_VM_*`, boutons `skipAvisFiscal`, `next_step_client`, `next_step_chantier`, `enregistrer`, fiche `FicheVM_ISO_Fiche_DealId`, `surfaceHabitableInput`, `typeLogement`, `FicheVM_Fiche_RefDossierExterne`). C'est le seul fichier à adapter.

## Coût

~0,01 à 0,03 € par fiche avec Sonnet (une capture + un PDF de devis), Firebase gratuit à ce volume.
