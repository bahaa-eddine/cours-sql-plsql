# Partie 1 — Introduction aux bases de données relationnelles et modèle relationnel

> Pr. BE. ELBAGHAZAOUI — ENSA BM

## Sommaire

1. [Qu'est-ce qu'une base de données ?](#1-quest-ce-quune-base-de-données-)
2. [Fichier plat vs base relationnelle](#2-fichier-plat-vs-base-de-données-relationnelle)
3. [Le SGBD](#3-le-sgbd-système-de-gestion-de-base-de-données)
4. [Le modèle relationnel (Codd, 1970)](#4-le-modèle-relationnel-codd-1970)
5. [Clés primaires et clés étrangères](#5-clés-primaires-et-clés-étrangères)
6. [Contraintes d'intégrité](#6-contraintes-dintégrité)
7. [Types de relations (1-1, 1-N, N-N)](#7-types-de-relations-et-cardinalités)
8. [Exemple de schéma : université](#8-exemple-complet--schéma-université)

---

## 1. Qu'est-ce qu'une base de données ?

**Définition :** une base de données est un système organisé pour **stocker, gérer et manipuler** des données.

**Exemples :** gestion des étudiants d'une école, système bancaire, site e-commerce.

### Où est stockée une base de données ?

| Emplacement | Rôle |
|---|---|
| **Disque dur / SSD** | Stockage **permanent** dans des fichiers physiques (tables, index, journaux de transactions). |
| **Mémoire vive (RAM)** | Stockage **temporaire** : le SGBD y charge une partie des données (cache, buffers) pour accélérer les requêtes. |

> 💡 Les données permanentes restent toujours sur le disque ; la RAM n'est qu'un accélérateur.

---

## 2. Fichier plat vs base de données relationnelle

| Fichier plat (CSV, Excel, texte) | Base de données relationnelle |
|---|---|
| Données dans un seul fichier | Données organisées en **tables reliées** |
| Redondance et incohérences fréquentes | Redondance réduite, cohérence garantie |
| Aucune relation entre les données | Relations via clés primaires / étrangères |
| Recherche et mise à jour difficiles | Accès rapide et sécurisé via **SQL** |

### Exemple simple

**Fichier plat** `commandes.csv` :

```text
client,ville,produit
Ali,Rabat,PC
Ali,Rabat,Souris
Ali,Rabbat,Clavier      <-- faute de frappe : incohérence !
```

➡️ La ville d'Ali est répétée 3 fois (redondance) et une erreur s'est glissée.

**Base relationnelle** : on sépare en deux tables.

```text
CLIENTS                    COMMANDES
id | nom | ville           id | id_client | produit
1  | Ali | Rabat           1  | 1         | PC
                           2  | 1         | Souris
                           3  | 1         | Clavier
```

➡️ La ville n'est stockée **qu'une seule fois**. Pour la corriger, on modifie une seule ligne.

---

## 3. Le SGBD (Système de Gestion de Base de Données)

Un **SGBD** est un logiciel qui permet de **créer, stocker, organiser et manipuler** les données.
Il assure la **sécurité**, la **cohérence** et la gestion de la **concurrence** (plusieurs utilisateurs en même temps).

**Fonctions principales :**
- Définir les structures (tables, schémas)
- Manipuler les données (ajout, suppression, modification, recherche)
- Gérer les utilisateurs et les permissions
- Sauvegarder et restaurer

**Exemples de SGBD :** MySQL, PostgreSQL, Oracle, SQL Server, SQLite.

---

## 4. Le modèle relationnel (Codd, 1970)

Proposé par **Edgar Frank Codd** (IBM) en 1970. Les données sont représentées sous forme de **tables** (appelées *relations*).

### Vocabulaire

| Terme | Synonyme | Signification | Exemple |
|---|---|---|---|
| **Table** | Relation | Ensemble de données sur un même sujet | `ETUDIANTS` |
| **Attribut** | Colonne | Une caractéristique | `Nom`, `Prenom`, `Age` |
| **Tuple** | Ligne / enregistrement | Une instance (un élément) | `(1, 'MBOUZANG', 'Frank', 18)` |

### Exemple : table ETUDIANTS

| ID_Etudiant | Nom | Prenom | Age |
|---|---|---|---|
| 1 | MBOUZANG | Frank | 18 |
| 2 | ALAOUI | Sara | 20 |
| 3 | BENNANI | Ali | 19 |

- La table a **4 attributs** (colonnes) et **3 tuples** (lignes).

**Avantages du modèle relationnel :** structure simple et logique, moins de redondance, cohérence des données, puissance du langage SQL.

---

## 5. Clés primaires et clés étrangères

### Clé primaire (PK – Primary Key)
- Identifie **de manière unique** chaque ligne.
- Toujours **unique** et **non nulle** (`NOT NULL`).
- **Une seule** clé primaire par table (mais elle peut être composée de plusieurs colonnes).

> Exemple : `ID_Etudiant` dans `ETUDIANTS`. Deux étudiants peuvent s'appeler « Ali », mais ils n'auront jamais le même `ID_Etudiant`.

### Clé étrangère (FK – Foreign Key)
- Attribut qui **référence la clé primaire d'une autre table**.
- Sert à créer un **lien** entre deux tables.
- Garantit l'**intégrité référentielle** : impossible d'insérer une valeur qui n'existe pas dans la table référencée.

### Exemple

```text
COURS (ID_Cours, Intitule)                 INSCRIPTIONS (ID_Etudiant, ID_Cours)
       ▲ PK                                                  │ FK
       └─────────────────────────────────────────────────────┘
```

| COURS | |
|---|---|
| **ID_Cours** | Intitule |
| 10 | Bases de données |
| 20 | Java |

| INSCRIPTIONS | |
|---|---|
| ID_Etudiant | ID_Cours |
| 1 | 10 |
| 2 | 10 |
| 2 | 20 |

➡️ Insérer `(3, 99)` dans INSCRIPTIONS serait **refusé** : le cours 99 n'existe pas.

---

## 6. Contraintes d'intégrité

| Contrainte | Règle | Exemple |
|---|---|---|
| **Intégrité d'entité** | La clé primaire ne peut pas être nulle | Un étudiant sans `ID_Etudiant` est interdit |
| **Intégrité référentielle** | Une clé étrangère doit correspondre à une clé primaire existante | Impossible d'inscrire un étudiant à un cours inexistant |
| **Intégrité de domaine** | Chaque valeur respecte son type et ses règles | `age > 0`, date valide |

---

## 7. Types de relations et cardinalités

### Symboles (notation « patte de corbeau »)

| Symbole | Signification |
|---|---|
| `\|\|` | Un et un seul (obligatoire) |
| `o\|` | Zéro ou un |
| `o{` | Zéro, un ou plusieurs |
| `\|\|--\|\|` | Relation 1–1 |
| `\|\|--o{` | Relation 1–N |
| `o{--o{` | Relation N–N |

### 1-1 (Un à Un)
Chaque ligne d'une table correspond à **une seule** ligne de l'autre table. Rare (souvent fusionné en une seule table).

> Exemple : un étudiant possède **une seule** carte étudiante.

```text
ETUDIANT ||--|| CARTE_ETUDIANT
```

### 1-N (Un à Plusieurs)
Une ligne d'une table peut être liée à **plusieurs** lignes de l'autre table.

> Exemple : un professeur enseigne **plusieurs** cours, mais chaque cours a **un seul** professeur.
> ➡️ La clé étrangère `ID_Professeur` se place dans la table **COURS** (côté « N »).

```text
PROFESSEUR ||--o{ COURS
```

### N-N (Plusieurs à Plusieurs)
Plusieurs lignes d'une table sont liées à plusieurs lignes de l'autre. **Nécessite une table d'association.**

> Exemple : un étudiant suit plusieurs cours, un cours accueille plusieurs étudiants.
> ➡️ On crée la table **INSCRIPTIONS** qui contient les deux clés étrangères.

```text
ETUDIANT ||--o{ INSCRIPTION }o--|| COURS
```

---

## 8. Exemple complet : schéma université

```text
ETUDIANTS    (ID_Etudiant, Nom, Prenom, DateNaissance)
PROFESSEURS  (ID_Professeur, Nom, ID_Departement)
COURS        (ID_Cours, Intitule, Credit, #ID_Professeur)
INSCRIPTIONS (ID_Inscription, #ID_Etudiant, #ID_Cours, DateInscription)
```
*(souligné/PK en premier, `#` = clé étrangère)*

**Relations :**
- Un étudiant peut s'inscrire à plusieurs cours (N-N via INSCRIPTIONS).
- Un cours peut avoir plusieurs étudiants.
- Un professeur peut enseigner plusieurs cours (1-N).

---

<p align="center">
🏠 <a href="README.md">Sommaire</a> · <a href="02-diagramme-er.md">Partie 2 : Diagramme ER</a> ➡️
</p>
