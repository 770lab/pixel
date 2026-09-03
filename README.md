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

✅ Déjà fait, sauf le dernier point :

1. ✅ Projet Firebase créé : **`lab-pixel`** (forfait Spark, gratuit).
2. ✅ **Authentication** → *E-mail / mot de passe* activé → utilisateur créé : `chabadclub770@gmail.com` avec un mot de passe généré (communiqué à part, pas ici — c'est un repo public). Change-le dès ta première connexion sur `770lab.com/pixel`, ou directement dans Firebase Console → Authentication → Utilisateurs → ⋮ → Réinitialiser le mot de passe.
3. ✅ **Firestore** créé (édition Standard, région `eur3` Europe), règles de sécurité publiées (celles du fichier `firestore.rules`).
4. ✅ App Web enregistrée, config copiée dans `FIREBASE_CONFIG` (`web/index.html`) et dans `.firebaserc` (`lab-pixel`).
5. ✅ Domaine `770lab.com` ajouté aux domaines autorisés (Authentication → Paramètres).

6. ❌ **Bloqué : passage au forfait Blaze** (obligatoire pour les Cloud Functions). Tes deux comptes de facturation Google Cloud existants (« Paiement de Firebase ») ont *chacun atteint leur quota maximal de projets associés* — Firebase a refusé d'y rattacher `lab-pixel`. Pour débloquer, une des actions suivantes (je ne peux pas le faire moi-même, ça touche à la facturation) :
   - Va sur [console.cloud.google.com/billing](https://console.cloud.google.com/billing), ouvre un des deux comptes « Paiement de Firebase », onglet **Projects**, et détache un projet que tu n'utilises plus ; ou
   - Demande une augmentation de quota de projets sur ce compte de facturation ; ou
   - Crée un nouveau compte de facturation (nouvelle carte) et attache-le à `lab-pixel`.

   Une fois débloqué : Console Firebase → `lab-pixel` → ⚙️ → Utilisation et facturation → passer à **Blaze**, puis :

```bash
npm i -g firebase-tools
firebase login
cd pixel-fiche
firebase use lab-pixel

# clé API Anthropic (stockée en secret, jamais dans le code)
firebase functions:secrets:set ANTHROPIC_API_KEY

# optionnel : functions/.env  (copier .env.example) — emails autorisés, modèle
cd functions && npm install && cd ..

firebase deploy --only functions
```

Après le deploy, la fonction est appelée par le site via `firebase.app().functions("europe-west1").httpsCallable("extractFiche")` — rien d'autre à configurer.

Modèle par défaut : `claude-sonnet-5`. Pour diviser le coût par ~3 : `CLAUDE_MODEL=claude-haiku-4-5-20251001` dans `functions/.env`.

## 2. Le site sur 770lab.com/pixel

✅ Déjà fait — déployé et en ligne sur **https://770lab.com/pixel/**.

Le site est un seul fichier `web/index.html`, poussé sur le repo dédié `770lab/pixel` (branche `main`, racine) avec GitHub Pages activé (Settings → Pages → Source = `main` / `/root`). Comme tes autres démos (`dessert`, `maurice`, `goku`…), ce repo hérite automatiquement du domaine personnalisé `770lab.com` déjà configuré sur ton repo racine (`{owner}.github.io`) : pas de DNS ni de routing à toucher, chaque nouveau repo `xxx` avec Pages activé devient `770lab.com/xxx/` tout seul.

Pour mettre à jour le site : remplace `index.html` dans le repo `770lab/pixel` (commit direct ou upload) — la page se republie automatiquement en 1–2 min.

Dans la console Firebase → **Authentication → Settings → Domaines autorisés**, ajoute `770lab.com` (à faire quand tu configures Firebase, étape 1).

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
