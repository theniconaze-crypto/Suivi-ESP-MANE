# ESP Manager – Suivi des contrôles réglementaires des équipements sous pression

Application web autonome (un seul fichier `index.html`) : Tailwind CSS, Chart.js et Firebase Firestore via CDN.
Aucune compilation, aucun serveur : elle se publie telle quelle sur GitHub Pages.

## 1. Créer le projet Firebase

1. Allez sur <https://console.firebase.google.com> → **Ajouter un projet**.
2. **Firestore Database** → *Créer une base de données* (mode production, région `eur3` ou `europe-west`).
3. **Authentication** → *Sign-in method* → activez **Anonyme** (sert uniquement à sécuriser l'accès à la base ; les comptes utilisateurs de l'application sont gérés dans Firestore).
4. **Paramètres du projet** (roue dentée) → *Vos applications* → icône **Web `</>`** → enregistrez l'application et copiez l'objet `firebaseConfig`.
5. Ouvrez `index.html`, section **1. CONFIGURATION**, et collez vos valeurs dans `FIREBASE_CONFIG`.

## 2. Règles de sécurité Firestore

Onglet **Firestore → Règles** :

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

Puis dans **Authentication → Paramètres → Domaines autorisés**, ajoutez `VOTRE-COMPTE.github.io`.
Dans la console Google Cloud (*API et services → Identifiants*), restreignez la clé API aux référents HTTP `https://VOTRE-COMPTE.github.io/*`.

## 3. Pièces jointes

Deux modes (constante `ENABLE_FILE_UPLOAD`) :

- `false` (par défaut) : liens sécurisés vers vos documents (GED, SharePoint, Drive…). Aucun coût, aucune configuration.
- `true` : téléversement direct dans **Firebase Storage**. Storage exige aujourd'hui le plan Blaze (facturation à l'usage, avec quota gratuit). Activez Storage, puis appliquez ces règles :

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /controls/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.resource.size < 10 * 1024 * 1024;
    }
  }
}
```

## 4. Publier sur GitHub Pages

1. Créez un dépôt GitHub (privé ou public selon votre offre) et déposez `index.html` à la racine.
2. **Settings → Pages** → *Source* : `Deploy from a branch` → branche `main`, dossier `/ (root)` → **Save**.
3. Après ~1 minute, l'application est disponible sur `https://VOTRE-COMPTE.github.io/NOM-DU-DEPOT/`.

## 5. Première connexion

- Identifiant : `admin` / mot de passe : `totototo` (le compte est créé automatiquement dans Firestore à la première connexion).
- **Changez immédiatement ce mot de passe** (bouton « Mon mot de passe »).
- Onglet *Utilisateurs* : créez les comptes et cochez les permissions.

## 6. Logique réglementaire implémentée

- Deux échéances par équipement : **visite périodique** et **requalification périodique**, avec périodicité en mois modifiable par appareil.
- Une requalification remet aussi à zéro la visite périodique ; un contrôle « Non conforme » ne remet aucun compteur à zéro.
- Sans contrôle enregistré, l'échéance part de la date de mise en service (à défaut, 1er janvier de l'année de construction).
- Code couleur : rouge = échu, orange ≤ 30 j, jaune ≤ 90 j, vert = conforme. Les équipements « Hors service / Rébuté » sont exclus.
- Les périodicités par défaut (4/10 ans, 6/12 ans pour les tuyauteries) sont **indicatives**. Faites-les valider par votre responsable réglementaire au regard de l'arrêté du 20/11/2017 et de votre plan d'inspection.

## 7. Limites de sécurité à connaître

L'authentification est gérée côté navigateur avec des comptes stockés dans Firestore (mots de passe hachés PBKDF2-SHA256 avec sel). Les règles Firestore ne peuvent pas distinguer les utilisateurs de l'application : toute personne disposant de la configuration Firebase et d'une session anonyme pourrait lire ou écrire dans la base en contournant l'interface. Les permissions cochées protègent donc l'interface, pas la base elle-même. C'est acceptable pour un outil interne à faible enjeu, pas pour des données sensibles.

Pour un renforcement réel : migrer vers **Firebase Authentication (e-mail/mot de passe)** avec des *custom claims* de permissions, appliqués dans les règles Firestore, et créer les comptes via une Cloud Function (plan Blaze).
