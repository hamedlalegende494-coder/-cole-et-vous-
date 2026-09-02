# École & Vous

Un seul fichier : **`index.html`** (HTML + CSS + JavaScript réunis). Aucun serveur, aucune étape de build — parfait pour GitHub Pages.

Authentification réelle (Firebase Authentication, e-mail + mot de passe) : chaque parent a son propre compte protégé par mot de passe, et l'espace direction est réservé aux comptes créés par l'établissement.

## 1. Créer le projet Firebase (gratuit, ~10 min)

1. Aller sur https://console.firebase.google.com et créer un projet.
2. Cliquer sur l'icône **`</>`** ("Ajouter une application Web"), lui donner un nom, puis copier l'objet `firebaseConfig` affiché.
3. Ouvrir `index.html`, chercher le bloc **« 1) CONFIGURATION FIREBASE »** et remplacer les valeurs par celles obtenues.
4. Menu de gauche → **Authentication → Get started → Sign-in method → Email/Password → Activer**. (Gratuit, aucune carte bancaire requise — contrairement à l'authentification par SMS.)
5. Menu de gauche → **Firestore Database → Créer une base de données** (mode production).
6. Toujours dans Firestore, onglet **Règles**, remplacer le contenu par celui donné plus bas, en indiquant l'e-mail (ou les e-mails) de la direction.

### Créer le compte de la direction

L'espace direction n'a **pas** de formulaire d'inscription (volontairement, pour ne pas laisser n'importe qui devenir administrateur) :
1. Menu de gauche → **Authentication → Users → Add user**.
2. Renseigner l'e-mail et le mot de passe de la direction.
3. Utiliser ce même e-mail dans les règles Firestore ci-dessous (`estDirection()`), sinon le compte pourra se connecter mais aucune écriture ne sera acceptée.

### Règles Firestore à copier

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function estConnecte() { return request.auth != null; }
    function estDirection() {
      return estConnecte() && request.auth.token.email in [
        'direction@votre-ecole.fr'   // <-- remplacer par le(s) email(s) de la direction
      ];
    }
    function estMonEnfant(eleveId) {
      return estConnecte() &&
        eleveId in get(/databases/$(database)/documents/parents/$(request.auth.uid)).data.enfantIds;
    }

    match /eleves/{id} {
      allow read: if estConnecte();
      allow write: if estDirection();
    }
    match /notes/{id} {
      allow read: if estConnecte();
      allow write: if estDirection();
    }
    match /presences/{id} {
      allow read: if estConnecte();
      allow create, delete: if estDirection();
      allow update: if estDirection() || estMonEnfant(resource.data.eleveId);
    }
    match /evenements/{id} {
      allow read: if estConnecte();
      allow create, delete: if estDirection();
      allow update: if estDirection() ||
        (estConnecte() && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['confirmedBy']));
    }
    match /parents/{uid} {
      allow read, write: if estConnecte() && request.auth.uid == uid;
    }
  }
}
```

Ce que ces règles garantissent :
- Un parent ne peut justifier une absence que pour **son propre enfant** (vérifié via `estMonEnfant`).
- Seule la direction peut importer des élèves, des notes, créer/supprimer des présences et des événements.
- Un parent ne peut modifier un événement que pour y ajouter sa confirmation de présence (`confirmedBy`), rien d'autre.
- Chaque parent ne peut lire/écrire que son propre profil (`parents/{uid}`).

## 2. Héberger sur GitHub Pages

1. Créer un dépôt GitHub, y déposer `index.html` à la racine.
2. **Settings → Pages → Source : Deploy from a branch**, choisir `main` et `/ (root)`.
3. Le site est en ligne quelques secondes après à `https://<votre-compte>.github.io/<votre-depot>/`.

## Parcours de démonstration

1. **Espace direction** → se connecter avec le compte créé dans la console Firebase → *Classes* → choisir une classe → importer un PDF liste de classe (texte sélectionnable, pas un scan) au format :
   ```
   NGUYEN;Léa
   DIALLO;Karim
   ```
   → les élèves apparaissent avec leur **code de liaison**.
2. Onglet *Notes* → classe + trimestre → PDF `NOM;Prénom;Matière;Note;Coefficient`, ex. `NGUYEN;Léa;Mathématiques;15,5;4`.
3. Onglet *Présences* → classe + mois → PDF `NOM;Prénom;Jour;Statut;Motif(optionnel)`, statut ∈ `present`, `absent_justifie`, `absent_non_justifie`.
4. Onglet *Événements* → ajout manuel.
5. **Espace parent** → *Créer un compte* → nom, prénom, téléphone, e-mail, mot de passe, nombre d'enfants, puis le code de liaison de chaque enfant → tableau de bord alimenté par les vraies données.

## Comment ça marche techniquement

- **Authentification** : Firebase Authentication (e-mail + mot de passe), gratuite, sans backend.
- **Données** : Firestore, appelé directement depuis le navigateur, protégé par les règles ci-dessus.
- **Lecture des PDF** : faite dans le navigateur avec `pdf.js`.
- **Session** : la connexion persiste au rechargement de la page (gérée par Firebase Auth).

## Limites connues

- Il n'y a qu'un rôle "direction" par liste d'e-mails codée en dur dans les règles — pas de gestion fine des droits par utilisateur (professeur vs direction, par exemple).
- Pas de réinitialisation de mot de passe intégrée à l'interface (Firebase la propose nativement, on peut l'ajouter facilement si besoin).
- La messagerie n'est pas encore branchée à Firestore.
