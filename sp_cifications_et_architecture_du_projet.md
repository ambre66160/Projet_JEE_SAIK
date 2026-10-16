# Document d'Architecture & Spécifications Techniques

**Projet :** Plateforme de Gestion de Voyages & Mobilité Académique (Optionnelle)  
**Stack Technique :** Java, Spring Boot, Spring Data JPA, Spring MVC, Thymeleaf, Base de données H2 embarquée, API REST  
**Livrables :** Application Web MVC + Endpoints REST (/api/v1/...)

---

## 1. Présentation Générale du Projet

Le projet consiste en une plateforme de gestion intégrée des déplacements et séjours d'étudiants ou d'usagers. L'application intègre une fonctionnalité centrale : la **Mobilité Académique en tant qu'option activable**. 

Un utilisateur/étudiant classique peut utiliser la plateforme pour des réservations ou séjours standards. Dès lors que l'option **"Mobilité Académique"** est activée sur son dossier (via une case à cocher ou un état dans la logique métier), un workflow spécifique s'enclenche :
- Exigence de documents administratifs supplémentaires (convention de mobilité, visa, attestation d'assurance).
- Association obligatoire à une université/destination partenaire.
- Suivi du statut d'éligibilité et validation pédagogique.

---

## 2. Diagramme et Modélisation du Domaine (UML)

### 2.1 Les 5 Entités Principales et leurs Attributs

1. **`Etudiant` (ou `Voyageur`)**
   - `id` (Long, PK)
   - `nom` (String)
   - `prenom` (String)
   - `email` (String)
   - `optionMobiliteActivee` (Boolean) **← Flag pour l'option activable**
   - `statutDossier` (Enum: INCOMPLET, EN_COURS, VALIDE, REFUSE)

2. **`DossierMobilite`**
   - `id` (Long, PK)
   - `codeDossier` (String)
   - `dateDebut` (LocalDate)
   - `dateFin` (LocalDate)
   - `niveauEtude` (String)
   - `estValide` (Boolean)

3. **`Destination` (Université Partenaire ou Ville d'accueil)**
   - `id` (Long, PK)
   - `nom` (String)
   - `pays` (String)
   - `ville` (String)
   - `capaciteAccueil` (Integer)

4. **`Logement`**
   - `id` (Long, PK)
   - `adresse` (String)
   - `type` (Enum: RESIDENCE_UNIVERSITAIRE, APPARTEMENT, FAMILLE_ACCUEIL)
   - `loyerMensuel` (Double)
   - `disponible` (Boolean)

5. **`DocumentAdministratif`**
   - `id` (Long, PK)
   - `typeDocument` (Enum: PASSEPORT, VISA, CONVENTION_MOBILITE, ATTESTATION_ASSURANCE)
   - `nomFichier` (String)
   - `estValide` (Boolean)
   - `obligatoirePourMobilite` (Boolean)

---

### 2.2 Relations entre les Entités (Exigences du Cahier des Charges)

```
 [ Etudiant ] 1  <------ (1-1) ------> 1 [ DossierMobilite ]
      |
      | 1
      |
     (1-N)
      |
      v N
 [ DocumentAdministratif ]

 [ Destination ] 1 <------ (1-N) ------> N [ Logement ]

 [ Etudiant ] N <------ (N-N) ------> N [ Destination ]
```

1. **Relation 1-1 (OneToOne) :** `Etudiant` $\leftrightarrow$ `DossierMobilite`
   - Un étudiant possède un unique dossier de mobilité (et réciproquement).
2. **Relation 1-N (OneToMany / ManyToOne) :** 
   - `Etudiant` $\rightarrow$ `DocumentAdministratif` (Un étudiant possède plusieurs documents administratifs).
   - `Destination` $\rightarrow$ `Logement` (Une destination dispose de plusieurs logements référencés).
3. **Relation N-N (ManyToMany) :** `Etudiant` $\leftrightarrow$ `Destination`
   - Un étudiant peut émettre plusieurs vœux de destinations, et une destination accueille plusieurs étudiants.

---

## 3. Logique Métier (Règles Fonctionnelles)

L'application ne se limite pas à des formulaires CRUD. Elle embarque une logique métier basée sur le statut de l'option mobilité :

1. **Activation/Désactivation de l'Option Mobilité :**
   - Lorsque `optionMobiliteActivee` passe à `true` :
     - Le système génère automatiquement une checklist de `DocumentAdministratif` obligatoires (Visa, Convention).
     - La sélection d'une `Destination` partenaire devient obligatoire.
2. **Calcul de l'Éligibilité du Dossier :**
   - Le statut du `DossierMobilite` bascule à `VALIDE` **uniquement si** :
     - Tous les documents marqués `obligatoirePourMobilite` sont validés (`estValide == true`).
     - L'étudiant est rattaché à une destination ayant une capacité d'accueil restante $> 0$.
     - Un logement est attribué.
3. **Gestion des Capacités des Destinations :**
   - Lors de la validation de l'association N-N entre un `Etudiant` et une `Destination`, le système décrémente la capacité d'accueil de la destination.

---

## 4. Architecture de Code & Design Patterns

L'application respecte le patron d'architecture **MVC (Model-View-Controller)** préconisé par Spring Boot.

```
src/main/java/com/projet/phytrip/
├── config/             # Configuration BDD H2, Security, Initialisation de données
├── controller/         # Contrôleurs MVC (Thymeleaf) & Controllers REST
│   ├── api/            # Endpoints REST (/api/v1/...)
│   └── mvc/            # Contrôleurs HTML (Thymeleaf)
├── model/              # Entités JPA (@Entity, @OneToOne, @OneToMany, @ManyToMany)
├── repository/         # Interfaces Spring Data JPA
├── service/            # Couche Métier / Service (Logique de validation, activation d'option)
└── dto/                # Data Transfer Objects pour l'API REST
```

### Pourquoi cette structure ?
- **Séparation des responsabilités :** La couche `Service` contient toute la logique métier. Les contrôleurs ne font que rediriger les flux ou sérialiser les objets.
- **Rendu Hybride :** Les contrôleurs sous `controller/mvc/` retournent des vues Thymeleaf pour l'IHM web. Les contrôleurs sous `controller/api/` renvoient des réponses JSON pour la consommation via un client HTTP comme **Bruno** ou **Postman**.
- **Portabilité :** La base de données **H2** est configurée en mode fichier embarqué au sein du projet (`./data/projetdb`), garantissant la persistance des données et un lancement immédiat sans installation externe.

---

## 5. Interactions Graphiques & API REST

### 5.1 Association / Dissociation Graphique (Thymeleaf)
Dans l'IHM Web (réalisée avec Bootstrap ou Tailwind CSS) :
- **Gestion N-N (Étudiant <-> Destination) :** Une page de gestion permet de cocher/décocher des destinations pour un étudiant et de valider l'association.
- **Gestion 1-N (Étudiant <-> Document) :** Un bouton d'upload/ajout permet de lier dynamiquement un nouveau document au dossier de l'étudiant.
- **Toggle d'option :** Un switch HTML permet d'activer/désactiver l'option Mobilité Académique sur la fiche de l'étudiant.

### 5.2 Exemples d'Endpoints API REST (/api/v1/)

Pour répondre aux exigences de test via un client HTTP (Bruno) :

| Méthode | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/v1/students` | Liste tous les étudiants et leur statut de mobilité |
| `POST` | `/api/v1/students/{id}/toggle-mobility` | Active ou désactive l'option mobilité |
| `GET` | `/api/v1/mobilities/pending` | Liste les dossiers de mobilité en attente de validation |
| `POST` | `/api/v1/destinations/{id}/assign-student/{studentId}` | Associe un étudiant à une destination |
| `DELETE` | `/api/v1/destinations/{id}/remove-student/{studentId}` | Dissocie un étudiant d'une destination |