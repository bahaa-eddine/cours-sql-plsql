# Partie 3 — SQL fondamental

> Pr. BE. ELBAGHAZAOUI — ENSA BM
> DDL, DML, DQL, agrégations, fonctions, jointures, transactions, index et vues.
> Chaque notion est illustrée par un exemple **exécutable** et son **résultat**.
> SGBD de référence : **PostgreSQL** (les variantes MySQL, SQL Server et Oracle sont signalées quand elles diffèrent).

---

## Sommaire

0. [La base d'exemple](#0-la-base-dexemple)
1. [Introduction à SQL](#1-introduction-à-sql)
2. [Les types de données](#2-les-types-de-données)
3. [DDL — créer et modifier la structure](#3-ddl--créer-et-modifier-la-structure)
4. [Les contraintes](#4-les-contraintes)
5. [DML — ajouter, modifier, supprimer des données](#5-dml--ajouter-modifier-supprimer-des-données)
6. [SELECT — interroger les données](#6-select--interroger-les-données)
7. [Les fonctions d'agrégation et GROUP BY](#7-les-fonctions-dagrégation-et-group-by)
8. [Les fonctions utiles](#8-les-fonctions-utiles)
9. [Les jointures](#9-les-jointures)
10. [Les transactions](#10-les-transactions)
11. [Index et vues](#11-index-et-vues)
12. [Récapitulatif](#12-récapitulatif)
13. [Exercices](#13-exercices)

---

## 0. La base d'exemple

Tous les exemples du chapitre utilisent la même petite base « **magasin** ».
Copiez ce script dans PostgreSQL (pgAdmin, psql ou [DB Fiddle](https://www.db-fiddle.com/)) pour tester chaque requête vous-même.

```sql
CREATE TABLE Clients (
    id    SERIAL PRIMARY KEY,
    nom   VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    ville VARCHAR(50),
    age   INT CHECK (age > 0)
);

CREATE TABLE Produits (
    id        SERIAL PRIMARY KEY,
    nom       VARCHAR(100) NOT NULL,
    prix      DECIMAL(10,2) CHECK (prix > 0),
    categorie VARCHAR(50)
);

CREATE TABLE Commandes (
    id            INT PRIMARY KEY,
    client_id     INT REFERENCES Clients(id),
    produit_id    INT REFERENCES Produits(id),
    quantite      INT DEFAULT 1,
    date_commande DATE DEFAULT CURRENT_DATE
);

INSERT INTO Clients (nom, email, ville, age) VALUES
    ('Ali',     'ali@gmail.com',    'Rabat',     25),
    ('Sara',    'sara@gmail.com',   'Casa',      32),
    ('Youssef', 'youssef@yahoo.fr', 'Marrakech', 17),
    ('Amine',   NULL,               'Casa',      40),
    ('Eli',     'eli@gmail.com',    'Rabat',     28);

INSERT INTO Produits (nom, prix, categorie) VALUES
    ('PC',        5000, 'Informatique'),
    ('Souris',     150, 'Informatique'),
    ('Téléphone', 3000, 'Téléphonie'),
    ('Casque',     400, 'Audio');

INSERT INTO Commandes (id, client_id, produit_id, quantite, date_commande) VALUES
    (101, 1, 1, 1, '2025-01-10'),
    (102, 2, 3, 1, '2025-02-05'),
    (103, 2, 2, 2, '2025-02-05'),
    (104, 5, 4, 1, '2025-03-12'),
    (105, 1, 2, 3, '2025-03-20');
```

**Table `Clients`**

| id | nom | email | ville | age |
|---|---|---|---|---|
| 1 | Ali | ali@gmail.com | Rabat | 25 |
| 2 | Sara | sara@gmail.com | Casa | 32 |
| 3 | Youssef | youssef@yahoo.fr | Marrakech | 17 |
| 4 | Amine | NULL | Casa | 40 |
| 5 | Eli | eli@gmail.com | Rabat | 28 |

**Table `Produits`**

| id | nom | prix | categorie |
|---|---|---|---|
| 1 | PC | 5000.00 | Informatique |
| 2 | Souris | 150.00 | Informatique |
| 3 | Téléphone | 3000.00 | Téléphonie |
| 4 | Casque | 400.00 | Audio |

**Table `Commandes`**

| id | client_id | produit_id | quantite | date_commande |
|---|---|---|---|---|
| 101 | 1 | 1 | 1 | 2025-01-10 |
| 102 | 2 | 3 | 1 | 2025-02-05 |
| 103 | 2 | 2 | 2 | 2025-02-05 |
| 104 | 5 | 4 | 1 | 2025-03-12 |
| 105 | 1 | 2 | 3 | 2025-03-20 |

> 💡 Sauf indication contraire, **chaque exemple repart de ces données d'origine**.

---

## 1. Introduction à SQL

**SQL** (*Structured Query Language*) est le langage standard pour dialoguer avec une base relationnelle.
On l'utilise pour **créer** des tables, **manipuler** les données et **interroger** la base.

### 1.1 Les familles de commandes

| Famille | Rôle | Commandes | Exemple |
|---|---|---|---|
| **DDL** (*Data Definition Language*) | Définir la **structure** | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | `CREATE TABLE Clients (...);` |
| **DML** (*Data Manipulation Language*) | Manipuler les **données** | `INSERT`, `UPDATE`, `DELETE` | `INSERT INTO Clients ...;` |
| **DQL** (*Data Query Language*) | **Lire** les données | `SELECT` | `SELECT * FROM Clients;` |
| **TCL** (*Transaction Control Language*) | Gérer les **transactions** | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` | `COMMIT;` |

> 🧠 Pour s'en souvenir : **DDL** agit sur la **boîte** (la table), **DML** agit sur le **contenu** (les lignes), **DQL** **regarde** le contenu.

### 1.2 Règles d'écriture

```sql
-- Ceci est un commentaire sur une ligne
/* Ceci est un commentaire
   sur plusieurs lignes */

SELECT nom FROM Clients;      -- une instruction se termine par ;
select nom from clients;      -- les mots-clés ne sont pas sensibles à la casse
SELECT nom FROM Clients WHERE ville = 'Rabat';  -- les textes s'écrivent entre apostrophes ' '
```

| Règle | ✅ Correct | ❌ Incorrect |
|---|---|---|
| Texte entre apostrophes simples | `'Rabat'` | `"Rabat"` (les guillemets servent aux noms de colonnes) |
| Apostrophe dans un texte : on la double | `'L''Oréal'` | `'L'Oréal'` |
| Nombre sans apostrophes | `age = 25` | `age = '25'` (fonctionne parfois mais à éviter) |
| Point-virgule à la fin | `SELECT * FROM Clients;` | — |

### 1.3 Architecture d'une base

```text
Serveur PostgreSQL
 └── Base de données  (ex. : ecommerce)
      └── Schéma      (ex. : public, shop)
           └── Tables (ex. : Clients, Produits)
                ├── Colonnes (id, nom, email…)
                └── Lignes   (1, 'Ali', 'ali@gmail.com'…)
```

### 1.4 Les schémas

Un **schéma** est un « dossier logique » qui regroupe des tables, des vues, des fonctions…
Par défaut, tout est créé dans le schéma `public`.

**Exemple 1 : créer une base et un schéma**

```sql
CREATE DATABASE ecommerce;                               -- crée la base
\c ecommerce                                             -- (psql) se connecte à la base
CREATE SCHEMA IF NOT EXISTS shop AUTHORIZATION postgres; -- crée le schéma shop, propriétaire postgres
```

**Exemple 2 : deux tables de même nom dans deux schémas**

```sql
CREATE SCHEMA crm;
CREATE TABLE shop.clients (id INT, nom VARCHAR(50));
CREATE TABLE crm.clients  (id INT, nom VARCHAR(50), telephone VARCHAR(20));

SELECT * FROM shop.clients;   -- les clients de la boutique
SELECT * FROM crm.clients;    -- les clients du service commercial
```

➡️ Aucun conflit : `shop.clients` et `crm.clients` sont deux tables différentes.

**Exemple 3 : ne plus écrire le préfixe grâce à `search_path`**

```sql
SET search_path TO shop, public;   -- chercher d'abord dans shop, puis dans public

CREATE TABLE produits (id INT);    -- crée shop.produits
SELECT * FROM clients;             -- lit shop.clients
```

---

## 2. Les types de données

Chaque colonne a un **type** qui définit ce qu'elle peut contenir. Bien choisir le type permet d'**économiser de la place**, d'**accélérer les requêtes** et d'**empêcher les données invalides**.

### 2.1 Les nombres

| Type | Contenu | Exemple de colonne | Exemple de valeur |
|---|---|---|---|
| `SMALLINT` | Petit entier (−32 768 à 32 767) | `nb_places SMALLINT` | 45 |
| `INT` / `INTEGER` | Entier (≈ ±2 milliards) | `age INT` | 25 |
| `BIGINT` | Très grand entier | `nb_vues BIGINT` | 8 500 000 000 |
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | Nombre **exact** : `p` chiffres au total, dont `s` après la virgule | `prix DECIMAL(10,2)` | 12345678.90 |
| `REAL` / `FLOAT` / `DOUBLE PRECISION` | Nombre **approché** (calcul scientifique) | `temperature REAL` | 36.6 |

**Exemple 1 : ce que `DECIMAL(5,2)` accepte**

`DECIMAL(5,2)` = 5 chiffres au total, dont 2 après la virgule → maximum **999.99**.

```sql
CREATE TABLE Test (montant DECIMAL(5,2));

INSERT INTO Test VALUES (999.99);   -- ✅ OK
INSERT INTO Test VALUES (12.5);     -- ✅ stocké 12.50
INSERT INTO Test VALUES (3.14159);  -- ✅ arrondi à 3.14
INSERT INTO Test VALUES (1000.00);  -- ❌ ERREUR : numeric field overflow
```

**Exemple 2 : pourquoi `DECIMAL` pour l'argent ?**

```sql
SELECT 0.1::FLOAT   + 0.2::FLOAT   AS avec_float,     -- 0.30000000000000004
       0.1::DECIMAL + 0.2::DECIMAL AS avec_decimal;   -- 0.3
```

| avec_float | avec_decimal |
|---|---|
| 0.30000000000000004 | 0.3 |

➡️ `FLOAT` fait de petites erreurs d'arrondi. Pour des **montants**, on utilise toujours `DECIMAL`.

### 2.2 Le texte

| Type | Contenu | Exemple |
|---|---|---|
| `CHAR(n)` | Longueur **fixe** : toujours `n` caractères (complété par des espaces) | `code_pays CHAR(2)` → `'MA'` |
| `VARCHAR(n)` | Longueur **variable**, `n` caractères maximum | `nom VARCHAR(100)` |
| `TEXT` | Texte long, sans limite pratique | `description TEXT` |

**Exemple 1 : `CHAR` vs `VARCHAR`**

```sql
CREATE TABLE Test (c CHAR(5), v VARCHAR(5));
INSERT INTO Test VALUES ('Ali', 'Ali');

SELECT OCTET_LENGTH(c) AS taille_char, OCTET_LENGTH(v) AS taille_varchar
FROM Test;
```

| taille_char | taille_varchar |
|---|---|
| 5 | 3 |

| Valeur insérée | Dans `CHAR(5)` | Dans `VARCHAR(5)` |
|---|---|---|
| `'Ali'` | `'Ali  '` (5 caractères, 2 espaces ajoutés) | `'Ali'` (3 caractères) |

➡️ `CHAR` convient aux codes de taille fixe (`'MA'`, `'FR'`) ; `VARCHAR` convient aux noms, emails, adresses…

**Exemple 2 : texte trop long**

```sql
INSERT INTO Test (v) VALUES ('Youssef');
-- ❌ ERREUR : value too long for type character varying(5)
```

### 2.3 Les dates et heures

| Type | Contenu | Exemple de valeur |
|---|---|---|
| `DATE` | Une date (année-mois-jour) | `'2025-08-21'` |
| `TIME` | Une heure | `'14:30:00'` |
| `TIMESTAMP` | Date + heure | `'2025-08-21 14:30:00'` |
| `INTERVAL` (PostgreSQL) | Une durée | `'3 days'`, `'1 year 2 months'` |

**Exemple :**

```sql
CREATE TABLE Rendez_vous (
    id         SERIAL PRIMARY KEY,
    jour       DATE,
    heure      TIME,
    created_at TIMESTAMP DEFAULT NOW()    -- date/heure de création remplie automatiquement
);

INSERT INTO Rendez_vous (jour, heure) VALUES ('2025-09-15', '09:30');
SELECT * FROM Rendez_vous;
```

| id | jour | heure | created_at |
|---|---|---|---|
| 1 | 2025-09-15 | 09:30:00 | 2025-09-01 10:12:45 |

> ⚠️ Format des dates : toujours **`'AAAA-MM-JJ'`** (`'2025-09-15'`). Le format `'15/09/2025'` peut être mal interprété.

### 2.4 Les booléens

| Type | Valeurs | Exemple |
|---|---|---|
| `BOOLEAN` | `TRUE`, `FALSE` (ou `NULL`) | `actif BOOLEAN DEFAULT TRUE` |

```sql
CREATE TABLE Comptes (
    login VARCHAR(30) PRIMARY KEY,
    actif BOOLEAN DEFAULT TRUE
);

INSERT INTO Comptes VALUES ('ali', TRUE), ('sara', FALSE);
INSERT INTO Comptes (login) VALUES ('omar');   -- actif = TRUE par défaut

SELECT login FROM Comptes WHERE actif = TRUE;  -- ali, omar
SELECT login FROM Comptes WHERE NOT actif;     -- sara
```

> 💡 En MySQL, `BOOLEAN` est un alias de `TINYINT(1)` : `TRUE` = 1, `FALSE` = 0.

### 2.5 Une table qui utilise tous les types

```sql
CREATE TABLE Etudiants (
    id             SERIAL PRIMARY KEY,   -- entier auto-incrémenté
    cne            CHAR(10),             -- code de taille fixe
    nom            VARCHAR(100),         -- texte court
    moyenne        DECIMAL(4,2),         -- ex. 15.75
    date_naissance DATE,
    boursier       BOOLEAN,
    remarque       TEXT,
    created_at     TIMESTAMP DEFAULT NOW()
);

INSERT INTO Etudiants (cne, nom, moyenne, date_naissance, boursier, remarque)
VALUES ('R130000001', 'Ali Bennani', 15.75, '2004-05-10', TRUE, 'Délégué de classe');
```

| id | cne | nom | moyenne | date_naissance | boursier | remarque | created_at |
|---|---|---|---|---|---|---|---|
| 1 | R130000001 | Ali Bennani | 15.75 | 2004-05-10 | true | Délégué de classe | 2025-09-01 10:15:00 |

---

## 3. DDL — créer et modifier la structure

Le DDL agit sur la **structure** (tables, colonnes, schémas), pas sur les lignes.
⚠️ Ces commandes sont souvent **irréversibles** (validation automatique dans la plupart des SGBD).

### 3.1 `CREATE TABLE` — créer une table

**Syntaxe**

```sql
CREATE TABLE nom_table (
    colonne1 TYPE [CONTRAINTES],
    colonne2 TYPE [CONTRAINTES],
    ...
    [CONTRAINTES DE TABLE]
);
```

**Exemple 1 : table simple**

```sql
CREATE TABLE Villes (
    id  SERIAL PRIMARY KEY,
    nom VARCHAR(50) NOT NULL
);
```

**Exemple 2 : table avec plusieurs contraintes**

```sql
CREATE TABLE Clients (
    id    SERIAL PRIMARY KEY,        -- identifiant auto-incrémenté (1, 2, 3…)
    nom   VARCHAR(100) NOT NULL,     -- obligatoire
    email VARCHAR(100) UNIQUE,       -- pas de doublon
    ville VARCHAR(50),               -- facultatif
    age   INT CHECK (age > 0)        -- doit être positif
);
```

**Exemple 3 : ne pas provoquer d'erreur si la table existe déjà**

```sql
CREATE TABLE IF NOT EXISTS Villes (
    id  SERIAL PRIMARY KEY,
    nom VARCHAR(50) NOT NULL
);
-- Si Villes existe déjà : simple avertissement, pas d'erreur.
```

**Exemple 4 : créer une table à partir d'une requête**

```sql
CREATE TABLE Clients_Rabat AS
SELECT id, nom, email FROM Clients WHERE ville = 'Rabat';
```

| id | nom | email |
|---|---|---|
| 1 | Ali | ali@gmail.com |
| 5 | Eli | eli@gmail.com |

➡️ La nouvelle table contient une **copie** des données (les contraintes ne sont pas copiées).

### 3.2 L'auto-incrément

Une colonne auto-incrémentée reçoit **automatiquement** un numéro unique à chaque insertion.

**Exemple :**

```sql
CREATE TABLE Villes (
    id  SERIAL PRIMARY KEY,
    nom VARCHAR(50)
);

INSERT INTO Villes (nom) VALUES ('Rabat'), ('Casa'), ('Fès');   -- on ne donne pas l'id
SELECT * FROM Villes;
```

| id | nom |
|---|---|
| 1 | Rabat |
| 2 | Casa |
| 3 | Fès |

**La syntaxe selon le SGBD :**

| SGBD | Syntaxe |
|---|---|
| PostgreSQL | `id SERIAL PRIMARY KEY` ou `id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` (standard) |
| MySQL / MariaDB | `id INT AUTO_INCREMENT PRIMARY KEY` |
| SQL Server | `id INT IDENTITY(1,1) PRIMARY KEY` |
| Oracle 12c et + | `id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY` |
| Oracle avant 12c | `CREATE SEQUENCE seq_villes START WITH 1 INCREMENT BY 1;` puis `seq_villes.NEXTVAL` |

### 3.3 `ALTER TABLE` — modifier une table

**Exemple 1 : ajouter une colonne**

```sql
ALTER TABLE Clients ADD COLUMN telephone VARCHAR(20);
```

| id | nom | … | age | telephone |
|---|---|---|---|---|
| 1 | Ali | … | 25 | NULL |
| 2 | Sara | … | 32 | NULL |

➡️ Les lignes existantes reçoivent `NULL` dans la nouvelle colonne.

**Exemple 2 : ajouter une colonne avec une valeur par défaut**

```sql
ALTER TABLE Clients ADD COLUMN pays VARCHAR(50) DEFAULT 'Maroc';
```

| id | nom | … | pays |
|---|---|---|---|
| 1 | Ali | … | Maroc |
| 2 | Sara | … | Maroc |

➡️ Les lignes existantes reçoivent directement `'Maroc'`.

**Exemple 3 : ajouter plusieurs colonnes en une fois**

```sql
ALTER TABLE Clients
    ADD COLUMN date_naissance DATE,
    ADD COLUMN actif BOOLEAN DEFAULT TRUE;
```

**Exemple 4 : renommer une colonne**

```sql
ALTER TABLE Clients RENAME COLUMN telephone TO tel;
```

**Exemple 5 : changer le type d'une colonne**

```sql
-- PostgreSQL
ALTER TABLE Clients ALTER COLUMN ville TYPE VARCHAR(100);
-- MySQL
ALTER TABLE Clients MODIFY ville VARCHAR(100);
```

**Exemple 6 : rendre une colonne obligatoire / lui donner une valeur par défaut**

```sql
ALTER TABLE Clients ALTER COLUMN ville SET NOT NULL;
ALTER TABLE Clients ALTER COLUMN ville SET DEFAULT 'Inconnue';
```

**Exemple 7 : supprimer une colonne**

```sql
ALTER TABLE Clients DROP COLUMN tel;
```

**Exemple 8 : renommer une table**

```sql
ALTER TABLE Clients RENAME TO Acheteurs;
```

**Exemple 9 : ajouter ou supprimer une contrainte**

```sql
ALTER TABLE Produits ADD CONSTRAINT chk_prix CHECK (prix < 100000);
ALTER TABLE Produits DROP CONSTRAINT chk_prix;
```

**Vérifier la structure après modification :**

| SGBD | Commande |
|---|---|
| PostgreSQL (psql) | `\d Clients` |
| MySQL / Oracle | `DESCRIBE Clients;` |
| SQL Server | `EXEC sp_help 'Clients';` |

### 3.4 `DROP` — supprimer un objet

**Exemple 1 : supprimer une table**

```sql
DROP TABLE Clients_Rabat;   -- la table et toutes ses données disparaissent
```

**Exemple 2 : éviter l'erreur si la table n'existe pas**

```sql
DROP TABLE Clients_Rabat;             -- ❌ ERREUR : table "clients_rabat" does not exist
DROP TABLE IF EXISTS Clients_Rabat;   -- ✅ simple avertissement
```

**Exemple 3 : suppression bloquée par une clé étrangère**

```sql
DROP TABLE Clients;
-- ❌ ERREUR : cannot drop table clients because other objects depend on it
-- (la table Commandes contient une clé étrangère vers Clients)
```

Deux solutions :

```sql
-- Solution 1 : supprimer d'abord la table « enfant », puis la table « parent »
DROP TABLE Commandes;
DROP TABLE Clients;

-- Solution 2 (PostgreSQL) : supprimer aussi les contraintes qui dépendent de la table
DROP TABLE Clients CASCADE;
```

**Exemple 4 : supprimer d'autres objets**

```sql
DROP SCHEMA crm CASCADE;        -- supprime le schéma et tout son contenu
DROP DATABASE ecommerce;        -- supprime toute la base
```

### 3.5 `TRUNCATE` — vider une table

`TRUNCATE` supprime **toutes les lignes** d'un coup, mais **garde la table** (colonnes, contraintes).

**Exemple 1 : vider une table**

```sql
TRUNCATE TABLE Villes;
SELECT * FROM Villes;    -- 0 ligne, mais la table existe toujours
```

**Exemple 2 : remettre le compteur à 1 (PostgreSQL)**

```sql
TRUNCATE TABLE Villes;
INSERT INTO Villes (nom) VALUES ('Tanger');   -- id = 4 (le compteur continue)

TRUNCATE TABLE Villes RESTART IDENTITY;
INSERT INTO Villes (nom) VALUES ('Tanger');   -- id = 1
```

**Exemple 3 : table référencée par une clé étrangère**

```sql
TRUNCATE TABLE Clients;
-- ❌ ERREUR : cannot truncate a table referenced in a foreign key constraint

TRUNCATE TABLE Clients, Commandes;   -- ✅ on vide les deux tables ensemble
TRUNCATE TABLE Clients CASCADE;      -- ✅ (PostgreSQL) vide aussi les tables liées
```

### 3.6 `DELETE` vs `TRUNCATE` vs `DROP`

| | `DELETE FROM t` | `TRUNCATE TABLE t` | `DROP TABLE t` |
|---|---|---|---|
| Famille | DML | DDL | DDL |
| Supprime les lignes | ✅ | ✅ (toutes) | ✅ |
| Garde la table | ✅ | ✅ | ❌ |
| `WHERE` possible | ✅ | ❌ | ❌ |
| Vitesse | Lent sur beaucoup de lignes | Très rapide | Très rapide |
| Compteur auto-incrément | Continue | Remis à zéro (selon SGBD) | Supprimé |

**Exemple :** la table `Villes` contient 1 000 000 de lignes.

```sql
DELETE FROM Villes WHERE nom = 'Rabat';  -- supprime seulement les lignes 'Rabat'
DELETE FROM Villes;                      -- supprime tout, ligne par ligne (lent)
TRUNCATE TABLE Villes;                   -- supprime tout d'un coup (rapide)
DROP TABLE Villes;                       -- supprime la table elle-même
```

---

## 4. Les contraintes

Une **contrainte** est une règle que la base vérifie **automatiquement** à chaque `INSERT` ou `UPDATE`.
Si la règle n'est pas respectée, la base **refuse** l'opération.

| Contrainte | Rôle |
|---|---|
| `NOT NULL` | Valeur obligatoire |
| `UNIQUE` | Pas de doublon |
| `CHECK` | Condition à respecter |
| `DEFAULT` | Valeur automatique si rien n'est donné |
| `PRIMARY KEY` | Identifiant unique et obligatoire |
| `FOREIGN KEY` | Lien vers une autre table |

### 4.1 `NOT NULL` — valeur obligatoire

```sql
CREATE TABLE Clients (
    id  SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL
);

INSERT INTO Clients (nom) VALUES ('Ali');   -- ✅ OK
INSERT INTO Clients (nom) VALUES (NULL);    -- ❌ ERREUR
INSERT INTO Clients (id)  VALUES (10);      -- ❌ ERREUR (nom non fourni = NULL)
-- ERREUR : null value in column "nom" violates not-null constraint
```

> ⚠️ Une chaîne vide `''` n'est **pas** `NULL` : `INSERT INTO Clients (nom) VALUES ('')` est accepté. Pour l'interdire, ajouter `CHECK (nom <> '')`.

### 4.2 `UNIQUE` — pas de doublon

```sql
CREATE TABLE Clients (
    id    SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE
);

INSERT INTO Clients (email) VALUES ('ali@gmail.com');   -- ✅ OK
INSERT INTO Clients (email) VALUES ('sara@gmail.com');  -- ✅ OK
INSERT INTO Clients (email) VALUES ('ali@gmail.com');   -- ❌ ERREUR
-- ERREUR : duplicate key value violates unique constraint "clients_email_key"

INSERT INTO Clients (email) VALUES (NULL);              -- ✅ OK
INSERT INTO Clients (email) VALUES (NULL);              -- ✅ OK (plusieurs NULL sont permis)
```

**Unicité sur un couple de colonnes :**

```sql
CREATE TABLE Salles (
    batiment CHAR(1),
    numero   INT,
    UNIQUE (batiment, numero)        -- le COUPLE doit être unique
);

INSERT INTO Salles VALUES ('A', 1);  -- ✅
INSERT INTO Salles VALUES ('B', 1);  -- ✅ (numéro 1 mais bâtiment différent)
INSERT INTO Salles VALUES ('A', 1);  -- ❌ ERREUR : (A, 1) existe déjà
```

### 4.3 `CHECK` — condition à respecter

```sql
CREATE TABLE Produits (
    id    SERIAL PRIMARY KEY,
    nom   VARCHAR(100),
    prix  DECIMAL(10,2) CHECK (prix > 0),
    stock INT CHECK (stock >= 0),
    note  INT CHECK (note BETWEEN 1 AND 5)
);

INSERT INTO Produits (nom, prix, stock, note) VALUES ('PC', 5000, 10, 4);  -- ✅
INSERT INTO Produits (nom, prix, stock, note) VALUES ('PC', -50, 10, 4);   -- ❌ prix négatif
INSERT INTO Produits (nom, prix, stock, note) VALUES ('PC', 5000, 10, 7);  -- ❌ note hors [1,5]
-- ERREUR : new row for relation "produits" violates check constraint "produits_prix_check"
```

**`CHECK` sur plusieurs colonnes :**

```sql
CREATE TABLE Locations (
    id         SERIAL PRIMARY KEY,
    date_debut DATE,
    date_fin   DATE,
    CHECK (date_fin >= date_debut)          -- la fin ne peut pas être avant le début
);

INSERT INTO Locations (date_debut, date_fin) VALUES ('2025-07-01', '2025-07-10');  -- ✅
INSERT INTO Locations (date_debut, date_fin) VALUES ('2025-07-10', '2025-07-01');  -- ❌
```

**`CHECK` avec une liste de valeurs :**

```sql
CREATE TABLE Commandes (
    id     SERIAL PRIMARY KEY,
    statut VARCHAR(20) CHECK (statut IN ('en attente', 'expédiée', 'livrée'))
);

INSERT INTO Commandes (statut) VALUES ('livrée');    -- ✅
INSERT INTO Commandes (statut) VALUES ('perdue');    -- ❌
```

### 4.4 `DEFAULT` — valeur par défaut

```sql
CREATE TABLE Clients (
    id         SERIAL PRIMARY KEY,
    nom        VARCHAR(100) NOT NULL,
    ville      VARCHAR(50) DEFAULT 'Inconnue',
    actif      BOOLEAN DEFAULT TRUE,
    date_ajout DATE DEFAULT CURRENT_DATE
);

INSERT INTO Clients (nom) VALUES ('Omar');
INSERT INTO Clients (nom, ville) VALUES ('Nadia', 'Fès');
SELECT * FROM Clients;
```

| id | nom | ville | actif | date_ajout |
|---|---|---|---|---|
| 1 | Omar | Inconnue | true | 2025-09-01 |
| 2 | Nadia | Fès | true | 2025-09-01 |

➡️ `DEFAULT` n'est utilisé que si la colonne **n'est pas mentionnée**. Si on écrit explicitement `NULL`, c'est `NULL` qui est stocké.

### 4.5 `PRIMARY KEY` — clé primaire

La clé primaire = `NOT NULL` + `UNIQUE`. Une seule par table.

**Exemple 1 : clé primaire simple**

```sql
CREATE TABLE Clients (
    id  INT PRIMARY KEY,
    nom VARCHAR(100)
);

INSERT INTO Clients VALUES (1, 'Ali');    -- ✅
INSERT INTO Clients VALUES (1, 'Sara');   -- ❌ ERREUR : l'id 1 existe déjà
INSERT INTO Clients VALUES (NULL, 'Eli'); -- ❌ ERREUR : l'id ne peut pas être NULL
```

**Exemple 2 : clé primaire composite (sur plusieurs colonnes)**

Un étudiant ne peut avoir qu'**une seule note par matière**.

```sql
CREATE TABLE Notes (
    id_etudiant INT,
    id_matiere  INT,
    note        DECIMAL(4,2),
    PRIMARY KEY (id_etudiant, id_matiere)
);

INSERT INTO Notes VALUES (1, 10, 14.5);   -- ✅ étudiant 1, matière 10
INSERT INTO Notes VALUES (1, 20, 12.0);   -- ✅ même étudiant, autre matière
INSERT INTO Notes VALUES (2, 10, 16.0);   -- ✅ autre étudiant, même matière
INSERT INTO Notes VALUES (1, 10, 18.0);   -- ❌ ERREUR : le couple (1, 10) existe déjà
```

**Exemple 3 : clé primaire nommée (ajoutée après création)**

```sql
ALTER TABLE Clients ADD CONSTRAINT pk_clients PRIMARY KEY (id);
```

### 4.6 `FOREIGN KEY` — clé étrangère

Une clé étrangère garantit qu'une valeur **existe** dans une autre table.

**Exemple 1 : les trois façons de l'écrire**

```sql
-- a) En ligne
CREATE TABLE Commandes (
    id        INT PRIMARY KEY,
    client_id INT REFERENCES Clients(id)
);

-- b) En fin de table
CREATE TABLE Commandes (
    id        INT PRIMARY KEY,
    client_id INT,
    FOREIGN KEY (client_id) REFERENCES Clients(id)
);

-- c) Avec un nom de contrainte (recommandé : messages d'erreur plus clairs)
CREATE TABLE Commandes (
    id        INT PRIMARY KEY,
    client_id INT,
    CONSTRAINT fk_commandes_client FOREIGN KEY (client_id) REFERENCES Clients(id)
);
```

**Exemple 2 : ce qui est accepté et refusé** (avec la base d'exemple)

```sql
INSERT INTO Commandes (id, client_id, produit_id) VALUES (106, 3, 1);   -- ✅ le client 3 existe
INSERT INTO Commandes (id, client_id, produit_id) VALUES (107, 99, 1);  -- ❌ le client 99 n'existe pas
-- ERREUR : insert or update on table "commandes" violates foreign key constraint

INSERT INTO Commandes (id, client_id, produit_id) VALUES (108, NULL, 1); -- ✅ NULL est permis (sauf si NOT NULL)

DELETE FROM Clients WHERE id = 1;   -- ❌ Ali a des commandes (101, 105)
-- ERREUR : update or delete on table "clients" violates foreign key constraint on table "commandes"

DELETE FROM Clients WHERE id = 4;   -- ✅ Amine n'a aucune commande
```

**Exemple 3 : clé étrangère composite**

```sql
CREATE TABLE Examens (
    id_etudiant INT,
    id_matiere  INT,
    date_exam   DATE,
    FOREIGN KEY (id_etudiant, id_matiere) REFERENCES Notes(id_etudiant, id_matiere)
);
```

**Exemple 4 : ajouter une clé étrangère après coup**

```sql
ALTER TABLE Commandes
    ADD CONSTRAINT fk_commandes_produit FOREIGN KEY (produit_id) REFERENCES Produits(id);
```

### 4.7 Que faire quand le parent est supprimé ou modifié ?

On précise le comportement avec `ON DELETE` et `ON UPDATE`. Situation de départ pour les exemples :

**Clients**

| id | nom |
|---|---|
| 1 | Ali |
| 2 | Sara |

**Commandes**

| id | client_id | produit |
|---|---|---|
| 101 | 1 | PC |
| 102 | 1 | Souris |
| 103 | 2 | Casque |

#### `ON DELETE CASCADE` — les enfants suivent le parent

```sql
FOREIGN KEY (client_id) REFERENCES Clients(id) ON DELETE CASCADE
```

```sql
DELETE FROM Clients WHERE id = 1;
```

**Commandes après :**

| id | client_id | produit |
|---|---|---|
| 103 | 2 | Casque |

➡️ Les commandes 101 et 102 d'Ali ont été **supprimées automatiquement**.

#### `ON DELETE SET NULL` — les enfants restent, sans lien

```sql
FOREIGN KEY (client_id) REFERENCES Clients(id) ON DELETE SET NULL
```

```sql
DELETE FROM Clients WHERE id = 1;
```

**Commandes après :**

| id | client_id | produit |
|---|---|---|
| 101 | NULL | PC |
| 102 | NULL | Souris |
| 103 | 2 | Casque |

➡️ Les commandes sont **conservées** (utile pour la comptabilité), mais ne sont plus liées à un client.

#### `ON DELETE RESTRICT` — interdit tant qu'il y a des enfants

```sql
FOREIGN KEY (client_id) REFERENCES Clients(id) ON DELETE RESTRICT
```

```sql
DELETE FROM Clients WHERE id = 1;   -- ❌ ERREUR : Ali a encore des commandes
DELETE FROM Commandes WHERE client_id = 1;
DELETE FROM Clients WHERE id = 1;   -- ✅ maintenant c'est possible
```

➡️ C'est le comportement **par défaut** (`NO ACTION`, quasi identique).

#### `ON UPDATE CASCADE` — les enfants suivent le changement d'identifiant

```sql
FOREIGN KEY (client_id) REFERENCES Clients(id) ON UPDATE CASCADE
```

```sql
UPDATE Clients SET id = 10 WHERE id = 1;
```

**Commandes après :**

| id | client_id | produit |
|---|---|---|
| 101 | 10 | PC |
| 102 | 10 | Souris |
| 103 | 2 | Casque |

#### Combiner les options

```sql
CREATE TABLE Commandes (
    id        INT PRIMARY KEY,
    client_id INT,
    produit   VARCHAR(50),
    FOREIGN KEY (client_id) REFERENCES Clients(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

> 🧠 **À retenir :** `CASCADE` = tout suit · `SET NULL` = on coupe le lien · `RESTRICT` = on interdit.

### 4.8 Exemple complet avec toutes les contraintes

```sql
CREATE TABLE Clients (
    id         SERIAL PRIMARY KEY,
    nom        VARCHAR(100) NOT NULL,
    email      VARCHAR(100) UNIQUE,
    ville      VARCHAR(50) DEFAULT 'Inconnue',
    age        INT CHECK (age > 0),
    date_ajout DATE DEFAULT CURRENT_DATE
);

CREATE TABLE Produits (
    id        SERIAL PRIMARY KEY,
    nom       VARCHAR(100) NOT NULL,
    prix      DECIMAL(10,2) NOT NULL CHECK (prix > 0),
    categorie VARCHAR(50)
);

CREATE TABLE Commandes (
    id            SERIAL PRIMARY KEY,
    client_id     INT NOT NULL,
    produit_id    INT NOT NULL,
    quantite      INT DEFAULT 1 CHECK (quantite > 0),
    date_commande DATE DEFAULT CURRENT_DATE,
    CONSTRAINT fk_cmd_client  FOREIGN KEY (client_id)  REFERENCES Clients(id)  ON DELETE CASCADE,
    CONSTRAINT fk_cmd_produit FOREIGN KEY (produit_id) REFERENCES Produits(id) ON DELETE RESTRICT
);
```

---

## 5. DML — ajouter, modifier, supprimer des données

Le DML agit sur les **lignes**. Dans une transaction, ces opérations peuvent être annulées avec `ROLLBACK` (voir [section 10](#10-les-transactions)).

### 5.1 `INSERT` — ajouter des lignes

**Syntaxe**

```sql
INSERT INTO table (col1, col2, ...) VALUES (val1, val2, ...);
```

**Exemple 1 : insérer une ligne en précisant les colonnes (recommandé)**

```sql
INSERT INTO Clients (nom, email, ville, age)
VALUES ('Nadia', 'nadia@gmail.com', 'Fès', 35);
```

| id | nom | email | ville | age |
|---|---|---|---|---|
| … | … | … | … | … |
| 5 | Eli | eli@gmail.com | Rabat | 28 |
| **6** | **Nadia** | **nadia@gmail.com** | **Fès** | **35** |

➡️ L'`id` 6 est généré automatiquement.

**Exemple 2 : insérer sans préciser les colonnes**

```sql
INSERT INTO Clients VALUES (7, 'Omar', 'omar@gmail.com', 'Tanger', 29);
```

⚠️ Il faut donner **toutes** les colonnes, **dans l'ordre exact** de la table. Si la table change (nouvelle colonne), la requête casse. À éviter.

**Exemple 3 : insérer seulement certaines colonnes**

```sql
INSERT INTO Clients (nom, ville) VALUES ('Karim', 'Agadir');
```

| id | nom | email | ville | age |
|---|---|---|---|---|
| 8 | Karim | NULL | Agadir | NULL |

➡️ Les colonnes non mentionnées reçoivent `NULL` (ou leur valeur `DEFAULT`).

**Exemple 4 : insérer plusieurs lignes d'un coup**

```sql
INSERT INTO Produits (nom, prix, categorie) VALUES
    ('Clavier',  250, 'Informatique'),
    ('Écran',   1800, 'Informatique'),
    ('Enceinte', 600, 'Audio');
```

➡️ 3 lignes ajoutées en une seule requête (plus rapide que 3 requêtes).

**Exemple 5 : insérer le résultat d'une requête (`INSERT ... SELECT`)**

```sql
CREATE TABLE Clients_Archive (id INT, nom VARCHAR(100), ville VARCHAR(50));

INSERT INTO Clients_Archive (id, nom, ville)
SELECT id, nom, ville FROM Clients WHERE ville = 'Rabat';
```

**Clients_Archive**

| id | nom | ville |
|---|---|---|
| 1 | Ali | Rabat |
| 5 | Eli | Rabat |

**Exemple 6 : récupérer l'identifiant généré (PostgreSQL)**

```sql
INSERT INTO Clients (nom, ville) VALUES ('Hind', 'Oujda')
RETURNING id;
```

| id |
|---|
| 6 |

**Exemple 7 : texte contenant une apostrophe**

```sql
INSERT INTO Produits (nom, prix) VALUES ('Sac d''ordinateur', 300);   -- on double l'apostrophe
```

### 5.2 `UPDATE` — modifier des lignes

**Syntaxe**

```sql
UPDATE table
SET col1 = val1, col2 = val2, ...
WHERE condition;
```

**Exemple 1 : modifier une seule ligne**

```sql
UPDATE Clients SET ville = 'Tanger' WHERE id = 1;
```

| | id | nom | ville |
|---|---|---|---|
| Avant | 1 | Ali | Rabat |
| Après | 1 | Ali | **Tanger** |

**Exemple 2 : modifier plusieurs colonnes**

```sql
UPDATE Clients
SET ville = 'Agadir', age = 26
WHERE id = 1;
```

| | id | nom | ville | age |
|---|---|---|---|---|
| Avant | 1 | Ali | Rabat | 25 |
| Après | 1 | Ali | **Agadir** | **26** |

**Exemple 3 : modifier plusieurs lignes avec un calcul**

```sql
UPDATE Clients SET age = age + 1 WHERE ville = 'Casa';
```

| nom | ville | age avant | age après |
|---|---|---|---|
| Sara | Casa | 32 | **33** |
| Amine | Casa | 40 | **41** |

**Exemple 4 : augmenter les prix de 10 %**

```sql
UPDATE Produits SET prix = prix * 1.10 WHERE categorie = 'Informatique';
```

| nom | prix avant | prix après |
|---|---|---|
| PC | 5000.00 | **5500.00** |
| Souris | 150.00 | **165.00** |

**Exemple 5 : remplir une valeur manquante**

```sql
UPDATE Clients SET email = 'amine@gmail.com' WHERE email IS NULL;
```

➡️ Seul Amine est concerné.

**Exemple 6 : modifier selon une autre table (sous-requête)**

Marquer comme inactifs les clients qui n'ont jamais commandé :

```sql
ALTER TABLE Clients ADD COLUMN actif BOOLEAN DEFAULT TRUE;

UPDATE Clients
SET actif = FALSE
WHERE id NOT IN (SELECT client_id FROM Commandes);
```

| nom | actif |
|---|---|
| Ali | true |
| Sara | true |
| Youssef | **false** |
| Amine | **false** |
| Eli | true |

**Exemple 7 : modifier avec une jointure**

```sql
-- PostgreSQL
UPDATE Clients c
SET ville = 'Rabat'
FROM Commandes cmd
WHERE c.id = cmd.client_id AND cmd.id = 104;

-- MySQL
UPDATE Clients c
JOIN Commandes cmd ON c.id = cmd.client_id
SET c.ville = 'Rabat'
WHERE cmd.id = 104;
```

**Exemple 8 : ⚠️ l'oubli du `WHERE`**

```sql
UPDATE Clients SET ville = 'Inconnue';
```

| nom | ville |
|---|---|
| Ali | Inconnue |
| Sara | Inconnue |
| Youssef | Inconnue |
| Amine | Inconnue |
| Eli | Inconnue |

➡️ **Toutes** les lignes sont modifiées !

### 5.3 `DELETE` — supprimer des lignes

**Syntaxe**

```sql
DELETE FROM table WHERE condition;
```

**Exemple 1 : supprimer une ligne précise**

```sql
DELETE FROM Clients WHERE id = 3;    -- supprime Youssef
```

**Exemple 2 : supprimer selon une condition**

```sql
DELETE FROM Clients WHERE age < 18;
```

| id | nom | age |
|---|---|---|
| 1 | Ali | 25 |
| 2 | Sara | 32 |
| ~~3~~ | ~~Youssef~~ | ~~17~~ |
| 4 | Amine | 40 |
| 5 | Eli | 28 |

**Exemple 3 : supprimer selon plusieurs conditions**

```sql
DELETE FROM Commandes
WHERE date_commande < '2025-02-01' AND quantite = 1;   -- supprime la commande 101
```

**Exemple 4 : supprimer selon une autre table (sous-requête)**

Supprimer les clients qui n'ont jamais commandé :

```sql
DELETE FROM Clients
WHERE id NOT IN (SELECT client_id FROM Commandes);
```

➡️ Youssef et Amine sont supprimés.

**Exemple 5 : supprimer avec une jointure**

```sql
-- PostgreSQL
DELETE FROM Commandes cmd
USING Clients c
WHERE cmd.client_id = c.id AND c.ville = 'Casa';

-- MySQL
DELETE cmd
FROM Commandes cmd
JOIN Clients c ON cmd.client_id = c.id
WHERE c.ville = 'Casa';
```

➡️ Les commandes 102 et 103 (Sara, qui habite Casa) sont supprimées.

**Exemple 6 : supprimer toutes les lignes**

```sql
DELETE FROM Commandes;    -- la table est vide, mais existe toujours
```

> 🛡️ **Bon réflexe :** avant un `UPDATE` ou un `DELETE`, lancez d'abord un `SELECT` avec **le même `WHERE`** pour voir quelles lignes seront touchées.
>
> ```sql
> SELECT * FROM Clients WHERE age < 18;   -- 1. je vérifie : seulement Youssef
> DELETE FROM Clients WHERE age < 18;     -- 2. je supprime
> ```

---

## 6. SELECT — interroger les données

### 6.1 Ordre d'écriture et ordre d'exécution

```sql
SELECT   colonnes        -- 5. choisir les colonnes à afficher
FROM     table           -- 1. partir de la table
WHERE    condition       -- 2. garder certaines lignes
GROUP BY colonne         -- 3. regrouper
HAVING   condition       -- 4. garder certains groupes
ORDER BY colonne         -- 6. trier
LIMIT    n;              -- 7. garder les n premières lignes
```

➡️ On **écrit** `SELECT` en premier, mais la base l'**exécute** presque en dernier. C'est pourquoi un alias défini dans le `SELECT` ne peut pas être utilisé dans le `WHERE`.

### 6.2 `SELECT` de base

**Exemple 1 : toutes les colonnes**

```sql
SELECT * FROM Clients;
```

➡️ Affiche toute la table `Clients` (5 lignes, 5 colonnes).

**Exemple 2 : certaines colonnes**

```sql
SELECT nom, ville FROM Clients;
```

| nom | ville |
|---|---|
| Ali | Rabat |
| Sara | Casa |
| Youssef | Marrakech |
| Amine | Casa |
| Eli | Rabat |

**Exemple 3 : renommer une colonne avec un alias (`AS`)**

```sql
SELECT nom AS client, ville AS "Ville de résidence" FROM Clients;
```

| client | Ville de résidence |
|---|---|
| Ali | Rabat |
| … | … |

**Exemple 4 : colonne calculée**

```sql
SELECT nom, prix, prix * 1.20 AS prix_ttc FROM Produits;
```

| nom | prix | prix_ttc |
|---|---|---|
| PC | 5000.00 | 6000.00 |
| Souris | 150.00 | 180.00 |
| Téléphone | 3000.00 | 3600.00 |
| Casque | 400.00 | 480.00 |

**Exemple 5 : calcul sans table**

```sql
SELECT 2 + 3 AS somme, 10 / 3 AS division_entiere, 10.0 / 3 AS division;
```

| somme | division_entiere | division |
|---|---|---|
| 5 | 3 | 3.3333333333333333 |

> ⚠️ Entier ÷ entier = entier (la partie décimale est perdue). Écrire `10.0` pour obtenir un résultat décimal.

### 6.3 `DISTINCT` — supprimer les doublons

**Exemple 1 : une colonne**

```sql
SELECT ville FROM Clients;           -- 5 lignes : Rabat, Casa, Marrakech, Casa, Rabat
SELECT DISTINCT ville FROM Clients;  -- 3 lignes
```

| ville |
|---|
| Rabat |
| Casa |
| Marrakech |

**Exemple 2 : plusieurs colonnes** (c'est le **couple** qui doit être unique)

```sql
SELECT DISTINCT client_id, date_commande FROM Commandes;
```

| client_id | date_commande |
|---|---|
| 1 | 2025-01-10 |
| 2 | 2025-02-05 |
| 5 | 2025-03-12 |
| 1 | 2025-03-20 |

➡️ Les commandes 102 et 103 (client 2, le 2025-02-05) ne donnent qu'une ligne.

### 6.4 `WHERE` — filtrer les lignes

| Opérateur | Signification | Exemple |
|---|---|---|
| `=` | égal | `ville = 'Rabat'` |
| `<>` ou `!=` | différent | `ville <> 'Casa'` |
| `>` / `>=` | supérieur / supérieur ou égal | `age >= 28` |
| `<` / `<=` | inférieur / inférieur ou égal | `age < 18` |

**Exemple 1 : égalité**

```sql
SELECT nom FROM Clients WHERE ville = 'Rabat';
```

| nom |
|---|
| Ali |
| Eli |

**Exemple 2 : différent**

```sql
SELECT nom, ville FROM Clients WHERE ville <> 'Casa';
```

| nom | ville |
|---|---|
| Ali | Rabat |
| Youssef | Marrakech |
| Eli | Rabat |

**Exemple 3 : supérieur ou égal**

```sql
SELECT nom, age FROM Clients WHERE age >= 28;
```

| nom | age |
|---|---|
| Sara | 32 |
| Amine | 40 |
| Eli | 28 |

**Exemple 4 : inférieur**

```sql
SELECT nom, age FROM Clients WHERE age < 18;
```

| nom | age |
|---|---|
| Youssef | 17 |

**Exemple 5 : filtre sur une date**

```sql
SELECT id, date_commande FROM Commandes WHERE date_commande >= '2025-03-01';
```

| id | date_commande |
|---|---|
| 104 | 2025-03-12 |
| 105 | 2025-03-20 |

**Exemple 6 : filtre sur un calcul**

```sql
SELECT nom, prix FROM Produits WHERE prix * 1.20 > 1000;   -- prix TTC > 1000
```

| nom | prix |
|---|---|
| PC | 5000.00 |
| Téléphone | 3000.00 |

### 6.5 `AND`, `OR`, `NOT` — combiner des conditions

**Exemple 1 : `AND` — toutes les conditions doivent être vraies**

```sql
SELECT nom, ville, age FROM Clients WHERE ville = 'Casa' AND age > 35;
```

| nom | ville | age |
|---|---|---|
| Amine | Casa | 40 |

**Exemple 2 : `OR` — au moins une condition vraie**

```sql
SELECT nom, ville FROM Clients WHERE ville = 'Casa' OR ville = 'Marrakech';
```

| nom | ville |
|---|---|
| Sara | Casa |
| Youssef | Marrakech |
| Amine | Casa |

**Exemple 3 : `NOT` — inverser une condition**

```sql
SELECT nom, ville FROM Clients WHERE NOT ville = 'Casa';
```

| nom | ville |
|---|---|
| Ali | Rabat |
| Youssef | Marrakech |
| Eli | Rabat |

**Exemple 4 : ⚠️ la priorité — `AND` passe avant `OR`**

```sql
-- Sans parenthèses
SELECT nom, ville, age FROM Clients
WHERE ville = 'Rabat' OR ville = 'Casa' AND age > 35;
```

La base comprend : `ville = 'Rabat' OR (ville = 'Casa' AND age > 35)`

| nom | ville | age |
|---|---|---|
| Ali | Rabat | 25 |
| Amine | Casa | 40 |
| Eli | Rabat | 28 |

```sql
-- Avec parenthèses
SELECT nom, ville, age FROM Clients
WHERE (ville = 'Rabat' OR ville = 'Casa') AND age > 35;
```

| nom | ville | age |
|---|---|---|
| Amine | Casa | 40 |

> 💡 Dès qu'on mélange `AND` et `OR`, **mettre des parenthèses**.

### 6.6 `LIKE` — rechercher un motif dans un texte

| Joker | Signification |
|---|---|
| `%` | 0, 1 ou plusieurs caractères quelconques |
| `_` | exactement 1 caractère |

| Motif | Signification | Exemples qui correspondent |
|---|---|---|
| `'A%'` | commence par A | Ali, Amine |
| `'%e'` | se termine par e | Amine |
| `'%ou%'` | contient « ou » | Youssef |
| `'_li'` | 3 lettres, finit par « li » | Ali, Eli |
| `'____'` | exactement 4 caractères | Sara |

**Exemple 1 : commence par**

```sql
SELECT nom FROM Clients WHERE nom LIKE 'A%';
```

| nom |
|---|
| Ali |
| Amine |

**Exemple 2 : se termine par**

```sql
SELECT nom, email FROM Clients WHERE email LIKE '%@gmail.com';
```

| nom | email |
|---|---|
| Ali | ali@gmail.com |
| Sara | sara@gmail.com |
| Eli | eli@gmail.com |

**Exemple 3 : contient**

```sql
SELECT nom FROM Clients WHERE nom LIKE '%ou%';    -- Youssef
```

**Exemple 4 : nombre précis de caractères**

```sql
SELECT nom FROM Clients WHERE nom LIKE '_li';     -- Ali, Eli (pas "Sali" : 4 lettres)
```

**Exemple 5 : ne contient pas**

```sql
SELECT nom FROM Clients WHERE email NOT LIKE '%gmail%';   -- Youssef
```

**Exemple 6 : ignorer majuscules / minuscules**

```sql
SELECT nom FROM Clients WHERE nom LIKE 'a%';          -- aucun résultat (LIKE distingue a et A en PostgreSQL)
SELECT nom FROM Clients WHERE nom ILIKE 'a%';         -- Ali, Amine (PostgreSQL)
SELECT nom FROM Clients WHERE LOWER(nom) LIKE 'a%';   -- Ali, Amine (tous SGBD)
```

### 6.7 `IN` — appartenir à une liste

**Exemple 1 : dans la liste**

```sql
SELECT nom, ville FROM Clients WHERE ville IN ('Casa', 'Marrakech');
-- équivalent à : ville = 'Casa' OR ville = 'Marrakech'
```

| nom | ville |
|---|---|
| Sara | Casa |
| Youssef | Marrakech |
| Amine | Casa |

**Exemple 2 : hors de la liste**

```sql
SELECT nom, ville FROM Clients WHERE ville NOT IN ('Casa', 'Rabat');
```

| nom | ville |
|---|---|
| Youssef | Marrakech |

**Exemple 3 : avec des nombres**

```sql
SELECT id, client_id FROM Commandes WHERE client_id IN (1, 5);
```

| id | client_id |
|---|---|
| 101 | 1 |
| 104 | 5 |
| 105 | 1 |

**Exemple 4 : la liste vient d'une autre requête (sous-requête)**

```sql
SELECT nom FROM Clients
WHERE id IN (SELECT client_id FROM Commandes);    -- clients ayant au moins une commande
```

| nom |
|---|
| Ali |
| Sara |
| Eli |

### 6.8 `BETWEEN` — dans un intervalle (bornes incluses)

**Exemple 1 : nombres**

```sql
SELECT nom, age FROM Clients WHERE age BETWEEN 25 AND 32;
-- équivalent à : age >= 25 AND age <= 32
```

| nom | age |
|---|---|
| Ali | 25 |
| Sara | 32 |
| Eli | 28 |

➡️ 25 et 32 sont **inclus**.

**Exemple 2 : en dehors de l'intervalle**

```sql
SELECT nom, age FROM Clients WHERE age NOT BETWEEN 20 AND 30;
```

| nom | age |
|---|---|
| Sara | 32 |
| Youssef | 17 |
| Amine | 40 |

**Exemple 3 : dates**

```sql
SELECT id, date_commande FROM Commandes
WHERE date_commande BETWEEN '2025-02-01' AND '2025-02-28';
```

| id | date_commande |
|---|---|
| 102 | 2025-02-05 |
| 103 | 2025-02-05 |

**Exemple 4 : prix**

```sql
SELECT nom, prix FROM Produits WHERE prix BETWEEN 100 AND 500;
```

| nom | prix |
|---|---|
| Souris | 150.00 |
| Casque | 400.00 |

### 6.9 `IS NULL` / `IS NOT NULL` — valeurs manquantes

`NULL` signifie « **inconnu** » ou « **absent** ». Ce n'est ni `0`, ni `''`.

**Exemple 1 : valeurs manquantes**

```sql
SELECT nom FROM Clients WHERE email IS NULL;
```

| nom |
|---|
| Amine |

**Exemple 2 : valeurs renseignées**

```sql
SELECT nom FROM Clients WHERE email IS NOT NULL;
```

| nom |
|---|
| Ali |
| Sara |
| Youssef |
| Eli |

**Exemple 3 : ⚠️ le piège du `= NULL`**

```sql
SELECT nom FROM Clients WHERE email = NULL;    -- ❌ 0 ligne ! (toujours faux)
SELECT nom FROM Clients WHERE email IS NULL;   -- ✅ Amine
```

**Exemple 4 : `NULL` est ignoré par les comparaisons**

```sql
SELECT nom FROM Clients WHERE email <> 'ali@gmail.com';
```

| nom |
|---|
| Sara |
| Youssef |
| Eli |

➡️ Amine n'apparaît pas : comparer `NULL` à quoi que ce soit donne « inconnu », jamais « vrai ».

### 6.10 `ORDER BY` — trier

**Exemple 1 : ordre croissant (par défaut)**

```sql
SELECT nom, age FROM Clients ORDER BY age;       -- ou ORDER BY age ASC
```

| nom | age |
|---|---|
| Youssef | 17 |
| Ali | 25 |
| Eli | 28 |
| Sara | 32 |
| Amine | 40 |

**Exemple 2 : ordre décroissant**

```sql
SELECT nom, age FROM Clients ORDER BY age DESC;
```

| nom | age |
|---|---|
| Amine | 40 |
| Sara | 32 |
| Eli | 28 |
| Ali | 25 |
| Youssef | 17 |

**Exemple 3 : trier sur du texte (ordre alphabétique)**

```sql
SELECT nom FROM Clients ORDER BY nom;    -- Ali, Amine, Eli, Sara, Youssef
```

**Exemple 4 : plusieurs critères**

```sql
SELECT nom, ville, age FROM Clients ORDER BY ville ASC, age DESC;
```

| nom | ville | age |
|---|---|---|
| Amine | Casa | 40 |
| Sara | Casa | 32 |
| Youssef | Marrakech | 17 |
| Eli | Rabat | 28 |
| Ali | Rabat | 25 |

➡️ D'abord par ville (A→Z) ; **à ville égale**, par âge décroissant.

**Exemple 5 : trier sur un alias**

```sql
SELECT nom, prix * 1.20 AS prix_ttc FROM Produits ORDER BY prix_ttc DESC;
```

| nom | prix_ttc |
|---|---|
| PC | 6000.00 |
| Téléphone | 3600.00 |
| Casque | 480.00 |
| Souris | 180.00 |

### 6.11 `LIMIT` / `OFFSET` — limiter le nombre de lignes

**Exemple 1 : les N premiers**

```sql
SELECT nom, age FROM Clients ORDER BY age DESC LIMIT 2;    -- les 2 plus âgés
```

| nom | age |
|---|---|
| Amine | 40 |
| Sara | 32 |

**Exemple 2 : le produit le moins cher**

```sql
SELECT nom, prix FROM Produits ORDER BY prix LIMIT 1;
```

| nom | prix |
|---|---|
| Souris | 150.00 |

**Exemple 3 : pagination avec `OFFSET` (sauter des lignes)**

```sql
SELECT nom, age FROM Clients ORDER BY age DESC LIMIT 2 OFFSET 2;   -- « page 2 »
```

| nom | age |
|---|---|
| Eli | 28 |
| Ali | 25 |

**La syntaxe selon le SGBD :**

| SGBD | Les 5 premières lignes |
|---|---|
| PostgreSQL / MySQL / SQLite | `SELECT * FROM Clients LIMIT 5;` |
| SQL Server | `SELECT TOP 5 * FROM Clients;` |
| Oracle 12c et + / standard SQL | `SELECT * FROM Clients FETCH FIRST 5 ROWS ONLY;` |
| Oracle (ancien) | `SELECT * FROM Clients WHERE ROWNUM <= 5;` |

> ⚠️ Sans `ORDER BY`, l'ordre des lignes n'est **pas garanti** : `LIMIT` renvoie alors des lignes « au hasard ».

---

## 7. Les fonctions d'agrégation et GROUP BY

### 7.1 Les fonctions d'agrégation

Une fonction d'agrégation **résume plusieurs lignes en une seule valeur**.

| Fonction | Rôle |
|---|---|
| `COUNT()` | Compter |
| `SUM()` | Additionner |
| `AVG()` | Moyenne |
| `MIN()` | Plus petite valeur |
| `MAX()` | Plus grande valeur |

**Exemple 1 : `COUNT` — les trois formes**

```sql
SELECT COUNT(*)              AS nb_lignes,     -- compte toutes les lignes
       COUNT(email)          AS nb_emails,     -- compte les valeurs NON NULL
       COUNT(DISTINCT ville) AS nb_villes      -- compte les valeurs différentes
FROM Clients;
```

| nb_lignes | nb_emails | nb_villes |
|---|---|---|
| 5 | 4 | 3 |

**Exemple 2 : `SUM`**

```sql
SELECT SUM(quantite) AS articles_vendus FROM Commandes;
```

| articles_vendus |
|---|
| 8 |

➡️ 1 + 1 + 2 + 1 + 3 = 8

**Exemple 3 : `AVG`**

```sql
SELECT AVG(age) AS age_moyen FROM Clients;
```

| age_moyen |
|---|
| 28.4 |

➡️ (25 + 32 + 17 + 40 + 28) / 5 = 142 / 5 = 28.4

**Exemple 4 : `MIN` et `MAX`**

```sql
SELECT MIN(prix) AS moins_cher, MAX(prix) AS plus_cher FROM Produits;
```

| moins_cher | plus_cher |
|---|---|
| 150.00 | 5000.00 |

**Exemple 5 : `MIN` / `MAX` sur des dates et du texte**

```sql
SELECT MIN(date_commande) AS premiere, MAX(date_commande) AS derniere FROM Commandes;
SELECT MIN(nom) AS premier_alphabetique FROM Clients;
```

| premiere | derniere |
|---|---|
| 2025-01-10 | 2025-03-20 |

| premier_alphabetique |
|---|
| Ali |

**Exemple 6 : agrégation + `WHERE`**

```sql
SELECT COUNT(*) AS nb_clients_rabat, AVG(age) AS age_moyen_rabat
FROM Clients
WHERE ville = 'Rabat';
```

| nb_clients_rabat | age_moyen_rabat |
|---|---|
| 2 | 26.5 |

**Exemple 7 : arrondir un résultat**

```sql
SELECT ROUND(AVG(prix), 2) AS prix_moyen FROM Produits;
```

| prix_moyen |
|---|
| 2137.50 |

> ⚠️ Les fonctions d'agrégation **ignorent les `NULL`** (sauf `COUNT(*)`).

### 7.2 `GROUP BY` — calculer par groupe

`GROUP BY` forme des **groupes** de lignes ayant la même valeur, puis applique la fonction à **chaque groupe**.

**Exemple 1 : nombre de clients par ville**

```sql
SELECT ville, COUNT(*) AS nb_clients
FROM Clients
GROUP BY ville;
```

Ce que fait la base :

```text
Rabat     → Ali, Eli        → 2
Casa      → Sara, Amine     → 2
Marrakech → Youssef         → 1
```

| ville | nb_clients |
|---|---|
| Rabat | 2 |
| Casa | 2 |
| Marrakech | 1 |

**Exemple 2 : âge moyen par ville**

```sql
SELECT ville, AVG(age) AS age_moyen
FROM Clients
GROUP BY ville;
```

| ville | age_moyen |
|---|---|
| Rabat | 26.5 |
| Casa | 36 |
| Marrakech | 17 |

**Exemple 3 : plusieurs calculs par groupe**

```sql
SELECT categorie,
       COUNT(*)  AS nb_produits,
       MIN(prix) AS prix_min,
       MAX(prix) AS prix_max
FROM Produits
GROUP BY categorie;
```

| categorie | nb_produits | prix_min | prix_max |
|---|---|---|---|
| Informatique | 2 | 150.00 | 5000.00 |
| Téléphonie | 1 | 3000.00 | 3000.00 |
| Audio | 1 | 400.00 | 400.00 |

**Exemple 4 : quantité achetée par client**

```sql
SELECT client_id, SUM(quantite) AS total_articles
FROM Commandes
GROUP BY client_id
ORDER BY total_articles DESC;
```

| client_id | total_articles |
|---|---|
| 1 | 4 |
| 2 | 3 |
| 5 | 1 |

**Exemple 5 : regrouper sur plusieurs colonnes**

```sql
SELECT client_id, date_commande, COUNT(*) AS nb_commandes
FROM Commandes
GROUP BY client_id, date_commande;
```

| client_id | date_commande | nb_commandes |
|---|---|---|
| 1 | 2025-01-10 | 1 |
| 2 | 2025-02-05 | 2 |
| 5 | 2025-03-12 | 1 |
| 1 | 2025-03-20 | 1 |

**Exemple 6 : ⚠️ l'erreur classique**

```sql
SELECT ville, nom, COUNT(*) FROM Clients GROUP BY ville;
-- ❌ ERREUR : column "clients.nom" must appear in the GROUP BY clause
```

➡️ Pour la ville Rabat, il y a 2 noms (Ali, Eli) : la base ne sait pas lequel afficher.
**Règle :** chaque colonne du `SELECT` doit être soit dans le `GROUP BY`, soit dans une fonction d'agrégation.

### 7.3 `HAVING` — filtrer les groupes

**Exemple 1 : villes ayant au moins 2 clients**

```sql
SELECT ville, COUNT(*) AS nb_clients
FROM Clients
GROUP BY ville
HAVING COUNT(*) >= 2;
```

| ville | nb_clients |
|---|---|
| Rabat | 2 |
| Casa | 2 |

**Exemple 2 : villes dont l'âge moyen dépasse 30 ans**

```sql
SELECT ville, AVG(age) AS age_moyen
FROM Clients
GROUP BY ville
HAVING AVG(age) > 30;
```

| ville | age_moyen |
|---|---|
| Casa | 36 |

**Exemple 3 : clients ayant acheté plus de 3 articles**

```sql
SELECT client_id, SUM(quantite) AS total
FROM Commandes
GROUP BY client_id
HAVING SUM(quantite) > 3;
```

| client_id | total |
|---|---|
| 1 | 4 |

### 7.4 `WHERE` ou `HAVING` ?

| | `WHERE` | `HAVING` |
|---|---|---|
| Filtre… | les **lignes** | les **groupes** |
| Moment | **avant** `GROUP BY` | **après** `GROUP BY` |
| Fonctions d'agrégation autorisées ? | ❌ Non | ✅ Oui |

**Exemple : les deux ensemble**

```sql
SELECT ville, COUNT(*) AS nb_majeurs
FROM Clients
WHERE age >= 18            -- 1. on retire les mineurs (Youssef)
GROUP BY ville             -- 2. on regroupe
HAVING COUNT(*) >= 2;      -- 3. on garde les villes avec au moins 2 majeurs
```

| ville | nb_majeurs |
|---|---|
| Rabat | 2 |
| Casa | 2 |

```sql
-- ❌ Erreur : une fonction d'agrégation dans WHERE
SELECT ville FROM Clients WHERE COUNT(*) >= 2 GROUP BY ville;
```

### 7.5 Tout combiner, étape par étape

```sql
SELECT ville, AVG(age) AS age_moyen
FROM Clients
WHERE age > 20
GROUP BY ville
HAVING AVG(age) > 30
ORDER BY age_moyen DESC
LIMIT 3;
```

**Étape 1 — `FROM Clients` :** les 5 clients.

**Étape 2 — `WHERE age > 20` :** Youssef (17) est retiré.

| nom | ville | age |
|---|---|---|
| Ali | Rabat | 25 |
| Sara | Casa | 32 |
| Amine | Casa | 40 |
| Eli | Rabat | 28 |

**Étape 3 — `GROUP BY ville` + `AVG(age)` :**

| ville | age_moyen |
|---|---|
| Rabat | 26.5 |
| Casa | 36 |

**Étape 4 — `HAVING AVG(age) > 30` :** Rabat est retirée.

**Étapes 5 et 6 — `ORDER BY` + `LIMIT 3` :** résultat final.

| ville | age_moyen |
|---|---|
| Casa | 36 |

---

## 8. Les fonctions utiles

### 8.1 Fonctions sur le texte

| Fonction | Rôle | Exemple | Résultat |
|---|---|---|---|
| `UPPER(x)` | Majuscules | `UPPER('Ali')` | `ALI` |
| `LOWER(x)` | Minuscules | `LOWER('SARA')` | `sara` |
| `LENGTH(x)` | Nombre de caractères | `LENGTH('Youssef')` | `7` |
| `SUBSTRING(x, début, longueur)` | Extraire un morceau | `SUBSTRING('Youssef', 1, 3)` | `You` |
| `CONCAT(a, b, …)` | Coller des textes | `CONCAT('Ali', ' - ', 'Rabat')` | `Ali - Rabat` |
| `a \|\| b` | Coller (PostgreSQL, Oracle) | `'Ali' \|\| '!'` | `Ali!` |
| `TRIM(x)` | Retirer les espaces au début et à la fin | `TRIM('  Rabat  ')` | `Rabat` |
| `REPLACE(x, a, b)` | Remplacer | `REPLACE('ali@gmail.com', 'gmail', 'ensa')` | `ali@ensa.com` |
| `LEFT(x, n)` / `RIGHT(x, n)` | n premiers / derniers caractères | `LEFT('Marrakech', 4)` | `Marr` |
| `POSITION(a IN x)` | Position d'un texte | `POSITION('@' IN 'ali@gmail.com')` | `4` |

**Exemple 1 : majuscules, minuscules et longueur**

```sql
SELECT nom, UPPER(nom) AS majuscules, LOWER(nom) AS minuscules, LENGTH(nom) AS longueur
FROM Clients;
```

| nom | majuscules | minuscules | longueur |
|---|---|---|---|
| Ali | ALI | ali | 3 |
| Sara | SARA | sara | 4 |
| Youssef | YOUSSEF | youssef | 7 |
| Amine | AMINE | amine | 5 |
| Eli | ELI | eli | 3 |

**Exemple 2 : extraire un morceau**

```sql
SELECT nom, SUBSTRING(nom, 1, 3) AS initiales, LEFT(ville, 3) AS code_ville
FROM Clients;
```

| nom | initiales | code_ville |
|---|---|---|
| Ali | Ali | Rab |
| Sara | Sar | Cas |
| Youssef | You | Mar |
| Amine | Ami | Cas |
| Eli | Eli | Rab |

**Exemple 3 : coller des textes**

```sql
SELECT CONCAT(nom, ' (', ville, ')') AS client FROM Clients;
```

| client |
|---|
| Ali (Rabat) |
| Sara (Casa) |
| Youssef (Marrakech) |
| Amine (Casa) |
| Eli (Rabat) |

**Exemple 4 : extraire le domaine d'un email**

```sql
SELECT email, SUBSTRING(email, POSITION('@' IN email) + 1) AS domaine
FROM Clients
WHERE email IS NOT NULL;
```

| email | domaine |
|---|---|
| ali@gmail.com | gmail.com |
| sara@gmail.com | gmail.com |
| youssef@yahoo.fr | yahoo.fr |
| eli@gmail.com | gmail.com |

**Exemple 5 : nettoyer des données saisies**

```sql
SELECT TRIM('   Rabat   ')                    AS sans_espaces,   -- 'Rabat'
       UPPER(TRIM('  casa '))                 AS propre,         -- 'CASA'
       REPLACE('06-12-34-56-78', '-', '')     AS telephone;      -- '0612345678'
```

**Exemple 6 : chercher sans tenir compte de la casse**

```sql
SELECT nom FROM Clients WHERE UPPER(nom) = 'SARA';   -- trouve 'Sara'
```

### 8.2 Gérer les `NULL` : `COALESCE`

`COALESCE(a, b, …)` renvoie la **première valeur non `NULL`**.

**Exemple 1 : remplacer un `NULL` à l'affichage**

```sql
SELECT nom, COALESCE(email, 'non renseigné') AS email FROM Clients;
```

| nom | email |
|---|---|
| Ali | ali@gmail.com |
| Sara | sara@gmail.com |
| Youssef | youssef@yahoo.fr |
| Amine | non renseigné |
| Eli | eli@gmail.com |

**Exemple 2 : éviter un calcul faux**

```sql
SELECT 100 + NULL;               -- NULL (tout calcul avec NULL donne NULL)
SELECT 100 + COALESCE(NULL, 0);  -- 100
```

**Exemple 3 : `NULL` dans une concaténation**

```sql
SELECT 'Email : ' || email        FROM Clients WHERE id = 4;  -- NULL
SELECT CONCAT('Email : ', email)  FROM Clients WHERE id = 4;  -- 'Email : ' (CONCAT ignore NULL)
```

### 8.3 Fonctions sur les nombres

| Fonction | Rôle | Exemple | Résultat |
|---|---|---|---|
| `ROUND(x, n)` | Arrondir à n décimales | `ROUND(15.678, 1)` | `15.7` |
| `CEIL(x)` | Arrondir au-dessus | `CEIL(4.1)` | `5` |
| `FLOOR(x)` | Arrondir en dessous | `FLOOR(4.9)` | `4` |
| `ABS(x)` | Valeur absolue | `ABS(-8)` | `8` |
| `MOD(a, b)` ou `a % b` | Reste de la division | `MOD(10, 3)` | `1` |

**Exemple :**

```sql
SELECT nom, prix, ROUND(prix * 0.85, 0) AS prix_solde FROM Produits;   -- remise de 15 %
```

| nom | prix | prix_solde |
|---|---|---|
| PC | 5000.00 | 4250 |
| Souris | 150.00 | 128 |
| Téléphone | 3000.00 | 2550 |
| Casque | 400.00 | 340 |

### 8.4 Fonctions sur les dates

| Fonction | Rôle | SGBD |
|---|---|---|
| `CURRENT_DATE` | Date du jour | Tous |
| `NOW()` / `CURRENT_TIMESTAMP` | Date et heure actuelles | PostgreSQL, MySQL |
| `EXTRACT(YEAR FROM d)` | Extraire l'année (`MONTH`, `DAY`, `HOUR`…) | PostgreSQL, MySQL, Oracle |
| `d + INTERVAL '30 days'` | Ajouter une durée | PostgreSQL |
| `d2 - d1` | Nombre de jours entre deux dates | PostgreSQL |
| `AGE(d2, d1)` | Différence en années, mois, jours | PostgreSQL |
| `TO_CHAR(d, 'DD/MM/YYYY')` | Formater une date | PostgreSQL, Oracle |
| `DATEDIFF(d2, d1)` / `DATE_ADD(d, INTERVAL 30 DAY)` | Différence / ajout | MySQL |
| `DATEDIFF(day, d1, d2)` / `DATEADD(day, 30, d)` | Différence / ajout | SQL Server |

**Exemple 1 : date et heure actuelles**

```sql
SELECT CURRENT_DATE AS aujourd_hui, NOW() AS maintenant;
```

| aujourd_hui | maintenant |
|---|---|
| 2025-09-01 | 2025-09-01 10:30:15 |

**Exemple 2 : extraire l'année, le mois, le jour**

```sql
SELECT id, date_commande,
       EXTRACT(YEAR  FROM date_commande) AS annee,
       EXTRACT(MONTH FROM date_commande) AS mois,
       EXTRACT(DAY   FROM date_commande) AS jour
FROM Commandes;
```

| id | date_commande | annee | mois | jour |
|---|---|---|---|---|
| 101 | 2025-01-10 | 2025 | 1 | 10 |
| 102 | 2025-02-05 | 2025 | 2 | 5 |
| 103 | 2025-02-05 | 2025 | 2 | 5 |
| 104 | 2025-03-12 | 2025 | 3 | 12 |
| 105 | 2025-03-20 | 2025 | 3 | 20 |

**Exemple 3 : ajouter une durée (date de livraison prévue)**

```sql
SELECT id, date_commande, date_commande + INTERVAL '7 days' AS livraison_prevue
FROM Commandes;
```

| id | date_commande | livraison_prevue |
|---|---|---|
| 101 | 2025-01-10 | 2025-01-17 |
| 102 | 2025-02-05 | 2025-02-12 |
| … | … | … |

**Exemple 4 : écart entre deux dates**

```sql
SELECT DATE '2025-03-20' - DATE '2025-01-10'       AS nb_jours,  -- PostgreSQL
       AGE(DATE '2025-03-20', DATE '2025-01-10')   AS ecart;     -- PostgreSQL
```

| nb_jours | ecart |
|---|---|
| 69 | 2 mons 10 days |

```sql
SELECT DATEDIFF('2025-03-20', '2025-01-10');           -- MySQL      : 69
SELECT DATEDIFF(day, '2025-01-10', '2025-03-20');      -- SQL Server : 69
```

**Exemple 5 : formater une date**

```sql
SELECT id, TO_CHAR(date_commande, 'DD/MM/YYYY') AS date_fr FROM Commandes;
```

| id | date_fr |
|---|---|
| 101 | 10/01/2025 |
| 102 | 05/02/2025 |
| … | … |

**Exemple 6 : nombre de commandes par mois**

```sql
SELECT EXTRACT(MONTH FROM date_commande) AS mois, COUNT(*) AS nb_commandes
FROM Commandes
GROUP BY mois
ORDER BY mois;
```

| mois | nb_commandes |
|---|---|
| 1 | 1 |
| 2 | 2 |
| 3 | 2 |

**Exemple 7 : commandes des 30 derniers jours**

```sql
SELECT * FROM Commandes
WHERE date_commande >= CURRENT_DATE - INTERVAL '30 days';
```

### 8.5 `CASE` — des conditions dans une requête

`CASE` fonctionne comme un « si… alors… sinon ».

**Syntaxe**

```sql
CASE
    WHEN condition1 THEN valeur1
    WHEN condition2 THEN valeur2
    ELSE valeur_par_defaut
END
```

> Les conditions sont testées **dans l'ordre** : la première vraie l'emporte.

**Exemple 1 : catégorie d'âge**

```sql
SELECT nom, age,
       CASE
           WHEN age < 18 THEN 'Mineur'
           WHEN age <= 30 THEN 'Jeune adulte'
           ELSE 'Adulte'
       END AS categorie
FROM Clients;
```

| nom | age | categorie |
|---|---|---|
| Ali | 25 | Jeune adulte |
| Sara | 32 | Adulte |
| Youssef | 17 | Mineur |
| Amine | 40 | Adulte |
| Eli | 28 | Jeune adulte |

**Exemple 2 : gamme de prix**

```sql
SELECT nom, prix,
       CASE
           WHEN prix < 500   THEN 'Économique'
           WHEN prix <= 3000 THEN 'Moyenne gamme'
           ELSE 'Haut de gamme'
       END AS gamme
FROM Produits;
```

| nom | prix | gamme |
|---|---|---|
| PC | 5000.00 | Haut de gamme |
| Souris | 150.00 | Économique |
| Téléphone | 3000.00 | Moyenne gamme |
| Casque | 400.00 | Économique |

**Exemple 3 : forme simple (comparer une seule colonne)**

```sql
SELECT nom,
       CASE ville
           WHEN 'Casa' THEN 'Casablanca'
           WHEN 'Rabat' THEN 'Rabat (capitale)'
           ELSE ville
       END AS ville_complete
FROM Clients;
```

| nom | ville_complete |
|---|---|
| Ali | Rabat (capitale) |
| Sara | Casablanca |
| Youssef | Marrakech |
| Amine | Casablanca |
| Eli | Rabat (capitale) |

**Exemple 4 : compter selon une condition**

```sql
SELECT SUM(CASE WHEN age >= 18 THEN 1 ELSE 0 END) AS majeurs,
       SUM(CASE WHEN age <  18 THEN 1 ELSE 0 END) AS mineurs
FROM Clients;
```

| majeurs | mineurs |
|---|---|
| 4 | 1 |

**Exemple 5 : `CASE` dans un `UPDATE`**

```sql
UPDATE Produits
SET prix = CASE
               WHEN categorie = 'Informatique' THEN prix * 0.90   -- -10 %
               WHEN categorie = 'Audio'        THEN prix * 0.80   -- -20 %
               ELSE prix
           END;
```

| nom | prix avant | prix après |
|---|---|---|
| PC | 5000.00 | 4500.00 |
| Souris | 150.00 | 135.00 |
| Téléphone | 3000.00 | 3000.00 |
| Casque | 400.00 | 320.00 |

**Exemple 6 : `CASE` dans un `ORDER BY` (ordre personnalisé)**

```sql
SELECT nom, ville FROM Clients
ORDER BY CASE ville WHEN 'Rabat' THEN 1 WHEN 'Casa' THEN 2 ELSE 3 END;
```

➡️ D'abord les clients de Rabat, puis ceux de Casa, puis les autres.

---

## 9. Les jointures

Les informations sont réparties dans plusieurs tables : `Commandes` contient `client_id = 2`, mais pas le **nom** du client.
Une **jointure** relie les tables grâce à la colonne commune (clé primaire ↔ clé étrangère).

```text
Clients.id  ◄────────►  Commandes.client_id
```

| Jointure | Résultat |
|---|---|
| `INNER JOIN` | Seulement les lignes qui ont une correspondance des deux côtés |
| `LEFT JOIN` | Toutes les lignes de gauche + correspondances (sinon `NULL`) |
| `RIGHT JOIN` | Toutes les lignes de droite + correspondances (sinon `NULL`) |
| `FULL OUTER JOIN` | Toutes les lignes des deux tables |
| `CROSS JOIN` | Toutes les combinaisons possibles |
| `SELF JOIN` | Une table reliée à elle-même |

```text
 INNER JOIN        LEFT JOIN        RIGHT JOIN      FULL OUTER JOIN
  ( A (█) B )     (██A█(█) B )     ( A (█)█B██)     (██A█(█)█B██)
```

### 9.1 `INNER JOIN` — les correspondances

**Exemple 1 : nom du client pour chaque commande**

```sql
SELECT Commandes.id AS commande, Clients.nom, Commandes.quantite
FROM Clients
INNER JOIN Commandes ON Clients.id = Commandes.client_id;
```

| commande | nom | quantite |
|---|---|---|
| 101 | Ali | 1 |
| 102 | Sara | 1 |
| 103 | Sara | 2 |
| 104 | Eli | 1 |
| 105 | Ali | 3 |

➡️ Youssef et Amine n'apparaissent pas : ils n'ont **aucune** commande.

**Exemple 2 : avec des alias (écriture plus courte)**

```sql
SELECT cmd.id AS commande, c.nom, c.ville
FROM Clients c
JOIN Commandes cmd ON c.id = cmd.client_id      -- JOIN seul = INNER JOIN
WHERE c.ville = 'Rabat';
```

| commande | nom | ville |
|---|---|---|
| 101 | Ali | Rabat |
| 104 | Eli | Rabat |
| 105 | Ali | Rabat |

**Exemple 3 : nom du produit pour chaque commande**

```sql
SELECT cmd.id AS commande, p.nom AS produit, p.prix
FROM Commandes cmd
JOIN Produits p ON cmd.produit_id = p.id;
```

| commande | produit | prix |
|---|---|---|
| 101 | PC | 5000.00 |
| 102 | Téléphone | 3000.00 |
| 103 | Souris | 150.00 |
| 104 | Casque | 400.00 |
| 105 | Souris | 150.00 |

**Exemple 4 : ⚠️ colonne ambiguë**

```sql
SELECT id, nom FROM Clients JOIN Commandes ON Clients.id = Commandes.client_id;
-- ❌ ERREUR : column reference "id" is ambiguous (id existe dans les deux tables)

SELECT Clients.id, nom FROM Clients JOIN Commandes ON Clients.id = Commandes.client_id;  -- ✅
```

### 9.2 `LEFT JOIN` — tout le côté gauche

**Exemple 1 : tous les clients, avec ou sans commande**

```sql
SELECT c.nom, cmd.id AS commande
FROM Clients c
LEFT JOIN Commandes cmd ON c.id = cmd.client_id;
```

| nom | commande |
|---|---|
| Ali | 101 |
| Ali | 105 |
| Sara | 102 |
| Sara | 103 |
| Youssef | NULL |
| Amine | NULL |
| Eli | 104 |

**Exemple 2 : trouver les clients sans commande**

```sql
SELECT c.nom
FROM Clients c
LEFT JOIN Commandes cmd ON c.id = cmd.client_id
WHERE cmd.id IS NULL;
```

| nom |
|---|
| Youssef |
| Amine |

**Exemple 3 : nombre de commandes par client (y compris 0)**

```sql
SELECT c.nom, COUNT(cmd.id) AS nb_commandes
FROM Clients c
LEFT JOIN Commandes cmd ON c.id = cmd.client_id
GROUP BY c.nom
ORDER BY nb_commandes DESC;
```

| nom | nb_commandes |
|---|---|
| Ali | 2 |
| Sara | 2 |
| Eli | 1 |
| Youssef | 0 |
| Amine | 0 |

➡️ `COUNT(cmd.id)` (et non `COUNT(*)`) pour obtenir 0 : les `NULL` ne sont pas comptés.

### 9.3 `RIGHT JOIN` — tout le côté droit

Pour cet exemple, on ajoute une commande passée **sans compte client** :

```sql
INSERT INTO Commandes (id, client_id, produit_id, quantite, date_commande)
VALUES (106, NULL, 3, 1, '2025-04-01');
```

**Exemple 1 : toutes les commandes, même sans client**

```sql
SELECT c.nom, cmd.id AS commande
FROM Clients c
RIGHT JOIN Commandes cmd ON c.id = cmd.client_id;
```

| nom | commande |
|---|---|
| Ali | 101 |
| Sara | 102 |
| Sara | 103 |
| Eli | 104 |
| Ali | 105 |
| NULL | 106 |

**Exemple 2 : `RIGHT JOIN` = `LEFT JOIN` en inversant les tables**

```sql
SELECT c.nom, cmd.id AS commande
FROM Commandes cmd
LEFT JOIN Clients c ON c.id = cmd.client_id;   -- même résultat que l'exemple 1
```

> 💡 En pratique, on utilise surtout `LEFT JOIN` en plaçant la table « principale » à gauche.

### 9.4 `FULL OUTER JOIN` — tout des deux côtés

*(Avec la commande 106 ajoutée ci-dessus.)*

```sql
SELECT c.nom, cmd.id AS commande
FROM Clients c
FULL OUTER JOIN Commandes cmd ON c.id = cmd.client_id;
```

| nom | commande |
|---|---|
| Ali | 101 |
| Sara | 102 |
| Sara | 103 |
| Eli | 104 |
| Ali | 105 |
| NULL | 106 |
| Youssef | NULL |
| Amine | NULL |

➡️ On voit à la fois les **clients sans commande** et les **commandes sans client**.

> ⚠️ MySQL ne connaît pas `FULL OUTER JOIN`. On le remplace par :
> ```sql
> SELECT c.nom, cmd.id FROM Clients c LEFT JOIN  Commandes cmd ON c.id = cmd.client_id
> UNION
> SELECT c.nom, cmd.id FROM Clients c RIGHT JOIN Commandes cmd ON c.id = cmd.client_id;
> ```

### 9.5 `CROSS JOIN` — toutes les combinaisons

Chaque ligne de A est associée à **chaque** ligne de B (2 × 3 = 6 lignes).

**Exemple : générer toutes les variantes d'un t-shirt**

```sql
CREATE TABLE Tailles  (taille VARCHAR(5));
CREATE TABLE Couleurs (couleur VARCHAR(10));
INSERT INTO Tailles  VALUES ('S'), ('M');
INSERT INTO Couleurs VALUES ('Rouge'), ('Bleu'), ('Noir');

SELECT t.taille, c.couleur
FROM Tailles t
CROSS JOIN Couleurs c;
```

| taille | couleur |
|---|---|
| S | Rouge |
| S | Bleu |
| S | Noir |
| M | Rouge |
| M | Bleu |
| M | Noir |

> ⚠️ Sur de grandes tables, le résultat explose : 1 000 × 1 000 = 1 000 000 de lignes.

### 9.6 `SELF JOIN` — une table avec elle-même

**Exemple : chaque employé et son chef**

```sql
CREATE TABLE Employes (
    id      INT PRIMARY KEY,
    nom     VARCHAR(50),
    id_chef INT REFERENCES Employes(id)
);

INSERT INTO Employes VALUES
    (1, 'Karim', NULL),   -- directeur, pas de chef
    (2, 'Sara',  1),
    (3, 'Ali',   1),
    (4, 'Omar',  2);

SELECT e.nom AS employe, chef.nom AS chef
FROM Employes e
LEFT JOIN Employes chef ON e.id_chef = chef.id;
```

| employe | chef |
|---|---|
| Karim | NULL |
| Sara | Karim |
| Ali | Karim |
| Omar | Sara |

➡️ La même table est utilisée deux fois avec deux alias : `e` (l'employé) et `chef`.

### 9.7 Jointures multiples

**Exemple 1 : client + produit + montant de chaque commande**

```sql
SELECT cmd.id      AS commande,
       c.nom       AS client,
       p.nom       AS produit,
       cmd.quantite,
       cmd.quantite * p.prix AS montant
FROM Commandes cmd
JOIN Clients  c ON cmd.client_id  = c.id
JOIN Produits p ON cmd.produit_id = p.id
ORDER BY cmd.id;
```

| commande | client | produit | quantite | montant |
|---|---|---|---|---|
| 101 | Ali | PC | 1 | 5000.00 |
| 102 | Sara | Téléphone | 1 | 3000.00 |
| 103 | Sara | Souris | 2 | 300.00 |
| 104 | Eli | Casque | 1 | 400.00 |
| 105 | Ali | Souris | 3 | 450.00 |

**Exemple 2 : total dépensé par client (jointure + `GROUP BY`)**

```sql
SELECT c.nom, SUM(cmd.quantite * p.prix) AS total_depense
FROM Commandes cmd
JOIN Clients  c ON cmd.client_id  = c.id
JOIN Produits p ON cmd.produit_id = p.id
GROUP BY c.nom
ORDER BY total_depense DESC;
```

| nom | total_depense |
|---|---|
| Ali | 5450.00 |
| Sara | 3300.00 |
| Eli | 400.00 |

**Exemple 3 : chiffre d'affaires par catégorie**

```sql
SELECT p.categorie, SUM(cmd.quantite * p.prix) AS chiffre_affaires
FROM Commandes cmd
JOIN Produits p ON cmd.produit_id = p.id
GROUP BY p.categorie
ORDER BY chiffre_affaires DESC;
```

| categorie | chiffre_affaires |
|---|---|
| Informatique | 5750.00 |
| Téléphonie | 3000.00 |
| Audio | 400.00 |

➡️ Informatique = PC 5000 + Souris 300 + Souris 450 = 5750.

---

## 10. Les transactions

Une **transaction** regroupe plusieurs instructions en **un seul bloc** :
👉 **soit tout réussit, soit rien n'est appliqué.**

```text
BEGIN  →  INSERT / UPDATE / DELETE ...  →  COMMIT   (tout valider)
                                        ↘  ROLLBACK (tout annuler)
```

Table utilisée pour les exemples :

```sql
CREATE TABLE Comptes (
    id    CHAR(1) PRIMARY KEY,
    titulaire VARCHAR(50),
    solde DECIMAL(10,2) CHECK (solde >= 0)
);
INSERT INTO Comptes VALUES ('A', 'Ali', 1000), ('B', 'Sara', 500);
```

| id | titulaire | solde |
|---|---|---|
| A | Ali | 1000.00 |
| B | Sara | 500.00 |

### 10.1 `COMMIT` — valider

**Exemple : virement de 100 DH d'Ali vers Sara**

```sql
BEGIN;
UPDATE Comptes SET solde = solde - 100 WHERE id = 'A';
UPDATE Comptes SET solde = solde + 100 WHERE id = 'B';
COMMIT;
```

| id | titulaire | solde |
|---|---|---|
| A | Ali | **900.00** |
| B | Sara | **600.00** |

➡️ Les deux modifications sont enregistrées **définitivement**.

### 10.2 `ROLLBACK` — annuler

**Exemple 1 : annulation volontaire**

```sql
BEGIN;
UPDATE Comptes SET solde = solde - 100 WHERE id = 'A';
-- On se rend compte d'une erreur de montant…
ROLLBACK;
```

| id | titulaire | solde |
|---|---|---|
| A | Ali | 1000.00 |
| B | Sara | 500.00 |

➡️ Rien n'a changé.

**Exemple 2 : annulation après une erreur**

Ali veut virer 1 500 DH mais n'en a que 1 000 (la contrainte `CHECK (solde >= 0)` refuse) :

```sql
BEGIN;
UPDATE Comptes SET solde = solde + 1500 WHERE id = 'B';   -- ✅ Sara : 2000
UPDATE Comptes SET solde = solde - 1500 WHERE id = 'A';   -- ❌ ERREUR : solde négatif
ROLLBACK;                                                 -- on annule tout
```

➡️ Sara ne reçoit pas l'argent, Ali n'est pas débité. La base reste **cohérente**.

**Exemple 3 : le même virement sans transaction**

```sql
UPDATE Comptes SET solde = solde + 1500 WHERE id = 'B';   -- ✅ exécuté et validé
UPDATE Comptes SET solde = solde - 1500 WHERE id = 'A';   -- ❌ erreur
```

| id | titulaire | solde |
|---|---|---|
| A | Ali | 1000.00 |
| B | Sara | **2000.00** ❌ |

➡️ 1 500 DH ont été **créés de nulle part** !

### 10.3 `SAVEPOINT` — point de retour intermédiaire

**Exemple :**

```sql
BEGIN;
UPDATE Clients SET ville = 'Tanger' WHERE id = 1;     -- modification 1
SAVEPOINT sp1;
UPDATE Clients SET ville = 'Oujda'  WHERE id = 2;     -- modification 2
ROLLBACK TO sp1;                                      -- annule seulement la modification 2
COMMIT;
```

| id | nom | ville |
|---|---|---|
| 1 | Ali | **Tanger** ✅ conservé |
| 2 | Sara | Casa (inchangé) |

### 10.4 Les propriétés ACID

| Lettre | Propriété | Signification | Exemple |
|---|---|---|---|
| **A** | Atomicité | Tout ou rien | Le virement est complet ou annulé |
| **C** | Cohérence | Les règles restent respectées | Aucun solde négatif |
| **I** | Isolation | Les transactions ne se gênent pas | Deux virements simultanés ne se mélangent pas |
| **D** | Durabilité | Ce qui est validé n'est jamais perdu | Après `COMMIT`, même une panne ne l'efface pas |

> ⚠️ Dans la plupart des outils, chaque instruction hors `BEGIN` est validée **automatiquement** (mode *autocommit*). Les commandes DDL (`CREATE`, `DROP`…) valident aussi automatiquement dans MySQL et Oracle.

---

## 11. Index et vues

### 11.1 Les index

Un **index** accélère les recherches, comme l'index à la fin d'un livre 📖 : au lieu de lire toutes les pages, on va directement à la bonne.

**Exemple 1 : créer un index**

```sql
CREATE INDEX idx_clients_nom ON Clients(nom);

-- Cette requête utilise l'index : beaucoup plus rapide sur une grande table
SELECT * FROM Clients WHERE nom = 'Sara';
```

**Exemple 2 : index unique (empêche aussi les doublons)**

```sql
CREATE UNIQUE INDEX idx_clients_email ON Clients(email);
```

**Exemple 3 : index sur plusieurs colonnes**

```sql
CREATE INDEX idx_cmd_client_date ON Commandes(client_id, date_commande);

-- Utile pour :
SELECT * FROM Commandes WHERE client_id = 1 AND date_commande >= '2025-03-01';
```

**Exemple 4 : vérifier que l'index est utilisé (PostgreSQL)**

```sql
EXPLAIN SELECT * FROM Clients WHERE nom = 'Sara';
-- Index Scan using idx_clients_nom on clients   ← l'index est utilisé
-- Seq Scan on clients                           ← toute la table est parcourue
```

**Exemple 5 : supprimer un index**

```sql
DROP INDEX idx_clients_nom;
```

**Sur quelles colonnes créer un index ?**

| ✅ Oui | ❌ Non |
|---|---|
| Colonnes souvent dans `WHERE` | Petites tables (quelques centaines de lignes) |
| Colonnes de jointure (clés étrangères) | Colonnes rarement recherchées |
| Colonnes souvent dans `ORDER BY` | Colonnes avec très peu de valeurs différentes (ex. `actif` vrai/faux) |

| ✅ Avantages | ⚠️ Inconvénients |
|---|---|
| Recherches et tris beaucoup plus rapides | Occupe de l'espace disque |
| | Ralentit un peu `INSERT`, `UPDATE`, `DELETE` (l'index doit être mis à jour) |

> 💡 Les colonnes `PRIMARY KEY` et `UNIQUE` sont indexées **automatiquement**.

### 11.2 Les vues

Une **vue** est une **requête enregistrée sous un nom**. On l'utilise ensuite comme une table.
Elle ne stocke pas de données : la requête est ré-exécutée à chaque utilisation.

**Syntaxe**

```sql
CREATE VIEW nom_vue AS
SELECT ...;
```

**Exemple 1 : vue simple (filtre)**

```sql
CREATE VIEW vue_clients_rabat AS
SELECT nom, email FROM Clients
WHERE ville = 'Rabat';

SELECT * FROM vue_clients_rabat;
```

| nom | email |
|---|---|
| Ali | ali@gmail.com |
| Eli | eli@gmail.com |

**Exemple 2 : la vue est toujours à jour**

```sql
INSERT INTO Clients (nom, email, ville, age) VALUES ('Nadia', 'nadia@gmail.com', 'Rabat', 35);
SELECT * FROM vue_clients_rabat;
```

| nom | email |
|---|---|
| Ali | ali@gmail.com |
| Eli | eli@gmail.com |
| Nadia | nadia@gmail.com |

➡️ Nadia apparaît sans rien modifier dans la vue.

**Exemple 3 : vue qui cache une jointure complexe**

```sql
CREATE VIEW vue_detail_commandes AS
SELECT cmd.id AS commande, c.nom AS client, p.nom AS produit,
       cmd.quantite, cmd.quantite * p.prix AS montant, cmd.date_commande
FROM Commandes cmd
JOIN Clients  c ON cmd.client_id  = c.id
JOIN Produits p ON cmd.produit_id = p.id;

-- Utilisation très simple :
SELECT client, produit, montant FROM vue_detail_commandes WHERE montant > 400;
```

| client | produit | montant |
|---|---|---|
| Ali | PC | 5000.00 |
| Sara | Téléphone | 3000.00 |
| Ali | Souris | 450.00 |

**Exemple 4 : vue de statistiques**

```sql
CREATE VIEW vue_ca_par_client AS
SELECT client, SUM(montant) AS total
FROM vue_detail_commandes
GROUP BY client;

SELECT * FROM vue_ca_par_client ORDER BY total DESC;
```

| client | total |
|---|---|
| Ali | 5450.00 |
| Sara | 3300.00 |
| Eli | 400.00 |

**Exemple 5 : vue pour la sécurité (cacher des colonnes)**

```sql
CREATE VIEW vue_clients_public AS
SELECT nom, ville FROM Clients;      -- pas d'email ni d'âge

GRANT SELECT ON vue_clients_public TO stagiaire;   -- le stagiaire ne voit que nom et ville
```

**Exemple 6 : modifier ou supprimer une vue**

```sql
CREATE OR REPLACE VIEW vue_clients_rabat AS
SELECT nom, email, age FROM Clients WHERE ville = 'Rabat';   -- on ajoute la colonne age

DROP VIEW vue_clients_rabat;
```

| ✅ Avantages | ⚠️ Inconvénients |
|---|---|
| Simplifie les requêtes complexes | Peut être lente si la requête derrière est lourde |
| Sécurité : on cache des colonnes | Pas toujours modifiable (`INSERT`/`UPDATE`) selon la vue |
| On écrit la requête une fois, on la réutilise | |

---

## 12. Récapitulatif

### Les commandes essentielles

| Besoin | Commande |
|---|---|
| Créer une table | `CREATE TABLE t (id SERIAL PRIMARY KEY, nom VARCHAR(50));` |
| Ajouter une colonne | `ALTER TABLE t ADD COLUMN col TYPE;` |
| Supprimer une table | `DROP TABLE t;` |
| Vider une table | `TRUNCATE TABLE t;` |
| Ajouter une ligne | `INSERT INTO t (a, b) VALUES (1, 2);` |
| Modifier des lignes | `UPDATE t SET a = 1 WHERE id = 5;` |
| Supprimer des lignes | `DELETE FROM t WHERE id = 5;` |
| Lire | `SELECT a, b FROM t WHERE ... ORDER BY a;` |
| Sans doublons | `SELECT DISTINCT a FROM t;` |
| Rechercher un motif | `WHERE nom LIKE 'A%'` |
| Liste / intervalle | `WHERE x IN (1, 2)` / `WHERE x BETWEEN 1 AND 9` |
| Valeur manquante | `WHERE x IS NULL` |
| Compter par groupe | `SELECT a, COUNT(*) FROM t GROUP BY a HAVING COUNT(*) > 1;` |
| Relier deux tables | `SELECT ... FROM a JOIN b ON a.id = b.a_id;` |
| Transaction | `BEGIN; ... COMMIT;` ou `ROLLBACK;` |
| Accélérer une recherche | `CREATE INDEX idx ON t(col);` |
| Requête enregistrée | `CREATE VIEW v AS SELECT ...;` |

### Les pièges classiques ⚠️

| Piège | ❌ | ✅ |
|---|---|---|
| Oublier le `WHERE` | `DELETE FROM Clients;` | `DELETE FROM Clients WHERE id = 3;` |
| Comparer à `NULL` | `WHERE email = NULL` | `WHERE email IS NULL` |
| Agrégat dans `WHERE` | `WHERE COUNT(*) > 2` | `HAVING COUNT(*) > 2` |
| Mélanger `AND`/`OR` sans parenthèses | `a OR b AND c` | `(a OR b) AND c` |
| Guillemets pour du texte | `ville = "Rabat"` | `ville = 'Rabat'` |
| Colonne hors `GROUP BY` | `SELECT ville, nom ... GROUP BY ville` | `SELECT ville, COUNT(nom) ... GROUP BY ville` |
| `LIMIT` sans `ORDER BY` | `SELECT * FROM t LIMIT 3` | `SELECT * FROM t ORDER BY x LIMIT 3` |
| Colonne ambiguë dans une jointure | `SELECT id FROM a JOIN b ...` | `SELECT a.id FROM a JOIN b ...` |

---

## 13. Exercices

Avec la [base d'exemple](#0-la-base-dexemple) :

1. Afficher le nom et l'âge des clients de Casa, du plus jeune au plus âgé.
2. Afficher les produits dont le prix est compris entre 200 et 3 000.
3. Afficher les clients dont l'email est un email Gmail.
4. Afficher le nombre de clients et l'âge moyen par ville, en gardant seulement les villes ayant au moins 2 clients.
5. Afficher pour chaque commande : le nom du client, le nom du produit et le montant (`quantite × prix`).
6. Afficher les clients qui n'ont passé **aucune** commande.
7. Afficher le produit le plus cher.
8. Afficher chaque produit avec une colonne `gamme` (`Économique` < 500, `Moyenne gamme` ≤ 3 000, `Haut de gamme` au-delà).
9. Dans une transaction, augmenter de 10 % le prix des produits de moins de 500 DH, puis valider.
10. Créer une vue `vue_ventes_par_categorie` qui donne le nombre d'articles vendus par catégorie.

<details>
<summary>💡 Corrigé</summary>

```sql
-- 1
SELECT nom, age FROM Clients WHERE ville = 'Casa' ORDER BY age;
--  Sara 32 | Amine 40

-- 2
SELECT nom, prix FROM Produits WHERE prix BETWEEN 200 AND 3000;
--  Téléphone 3000.00 | Casque 400.00

-- 3
SELECT nom, email FROM Clients WHERE email LIKE '%@gmail.com';
--  Ali | Sara | Eli

-- 4
SELECT ville, COUNT(*) AS nb_clients, AVG(age) AS age_moyen
FROM Clients
GROUP BY ville
HAVING COUNT(*) >= 2;
--  Rabat 2 26.5 | Casa 2 36

-- 5
SELECT c.nom AS client, p.nom AS produit, cmd.quantite * p.prix AS montant
FROM Commandes cmd
JOIN Clients  c ON cmd.client_id  = c.id
JOIN Produits p ON cmd.produit_id = p.id;

-- 6
SELECT c.nom
FROM Clients c
LEFT JOIN Commandes cmd ON c.id = cmd.client_id
WHERE cmd.id IS NULL;
--  Youssef | Amine

-- 7
SELECT nom, prix FROM Produits ORDER BY prix DESC LIMIT 1;
--  PC 5000.00

-- 8
SELECT nom, prix,
       CASE
           WHEN prix < 500   THEN 'Économique'
           WHEN prix <= 3000 THEN 'Moyenne gamme'
           ELSE 'Haut de gamme'
       END AS gamme
FROM Produits;

-- 9
BEGIN;
UPDATE Produits SET prix = prix * 1.10 WHERE prix < 500;   -- Souris 165.00, Casque 440.00
COMMIT;

-- 10
CREATE VIEW vue_ventes_par_categorie AS
SELECT p.categorie, SUM(cmd.quantite) AS articles_vendus
FROM Commandes cmd
JOIN Produits p ON cmd.produit_id = p.id
GROUP BY p.categorie;
--  Informatique 6 | Téléphonie 1 | Audio 1
```
</details>

---

<p align="center">
⬅️ <a href="02-diagramme-er.md">Partie 2 : Diagramme ER</a> · 🏠 <a href="README.md">Sommaire</a>
</p>
