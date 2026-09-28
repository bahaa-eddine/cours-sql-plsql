# Partie 3 — SQL fondamental

> Pr. BE. ELBAGHAZAOUI — ENSA BM
> DDL, DML, DQL, agrégations, fonctions, jointures, transactions, index et vues.

## Sommaire

1. [Introduction à SQL](#1-introduction-à-sql)
2. [Types de données](#2-types-de-données)
3. [DDL : CREATE, ALTER, DROP, TRUNCATE](#3-ddl--create-alter-drop-truncate)
4. [Contraintes SQL](#4-les-contraintes-sql)
5. [DML : INSERT, UPDATE, DELETE](#5-dml--insert-update-delete)
6. [DQL : SELECT, filtres, tris](#6-dql--select)
7. [Agrégations : GROUP BY, HAVING](#7-fonctions-dagrégation-group-by-having)
8. [Fonctions utiles (chaînes, dates, CASE)](#8-fonctions-utiles)
9. [Jointures](#9-les-jointures)
10. [Transactions](#10-transactions-commit-rollback-savepoint)
11. [Index et vues](#11-index-et-vues)
12. [Récapitulatif](#12-récapitulatif)

---

## 1. Introduction à SQL

**SQL** = *Structured Query Language* : langage standard pour dialoguer avec les bases relationnelles.

Les commandes SQL se répartissent en familles :

| Famille | Rôle | Commandes |
|---|---|---|
| **DDL** (Data Definition Language) | Définir la **structure** | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** (Data Manipulation Language) | Manipuler les **données** | `INSERT`, `UPDATE`, `DELETE` |
| **DQL** (Data Query Language) | **Interroger** les données | `SELECT` |
| **TCL** (Transaction Control) | Gérer les **transactions** | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

### Architecture d'une base

```text
Base de données
 └── Schéma (dossier logique, ex. : public, shop)
      └── Tables
           ├── Colonnes (attributs : nom, email, âge…)
           └── Lignes (enregistrements)
```

### Le schéma (PostgreSQL)
Un **schéma** est un « dossier logique » qui regroupe tables, vues, fonctions…

- **Organisation** : `shop`, `analytics`, `audit`…
- **Isolation** : on peut avoir `shop.clients` et `crm.clients`.
- **Sécurité** : droits (`GRANT`/`REVOKE`) par schéma.
- Par défaut, tout va dans le schéma `public`.

```sql
CREATE DATABASE ecommerce;                               -- crée la base
\c ecommerce                                             -- (psql) se connecte à la base
CREATE SCHEMA IF NOT EXISTS shop AUTHORIZATION postgres; -- crée le schéma shop
SET search_path TO shop, public;                         -- cherche d'abord dans shop, puis public
```

➡️ Après le `SET search_path`, `CREATE TABLE clients (...)` crée automatiquement `shop.clients`.

---

## 2. Types de données

Bien choisir le type permet d'**économiser la mémoire**, d'**accélérer les requêtes** et de **garantir l'intégrité**.

### Numériques

| Type | Description | Exemple |
|---|---|---|
| `INT` / `INTEGER` | Entier (≈ ±2 milliards) | `age INT` → 25 |
| `BIGINT` | Très grand entier | `vues BIGINT` → 8 500 000 000 |
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | Nombre exact : `p` chiffres au total dont `s` après la virgule | `prix DECIMAL(10,2)` → 12345678.90 |
| `FLOAT` / `DOUBLE` | Nombre réel approché | 3.14159 |

> 💡 Pour l'**argent**, utilisez toujours `DECIMAL` (pas d'erreur d'arrondi).

### Chaînes de caractères

| Type | Description | Exemple |
|---|---|---|
| `CHAR(n)` | Longueur **fixe** (complété par des espaces) | `CHAR(3)` : `'AB'` → `'AB '` |
| `VARCHAR(n)` | Longueur **variable**, max `n` | `nom VARCHAR(100)` |
| `TEXT` | Texte long | `description TEXT` |

### Dates et temps

| Type | Contenu | Exemple |
|---|---|---|
| `DATE` | Date seule | `2025-08-21` |
| `TIME` | Heure seule | `14:30:00` |
| `TIMESTAMP` / `DATETIME` | Date + heure | `2025-08-21 14:30:00` |

> 💡 Très utilisé : `created_at TIMESTAMP` pour savoir quand une ligne a été créée.

### Autres

| Type | Contenu | Exemple |
|---|---|---|
| `BOOLEAN` | `TRUE` / `FALSE` (MySQL : `TINYINT(1)` → 1/0) | `is_active BOOLEAN` |
| `BLOB` | Données binaires (images, fichiers) | `photo BLOB` |

---

## 3. DDL : CREATE, ALTER, DROP, TRUNCATE

Le DDL agit sur la **structure** de la base. ⚠️ Changements souvent **irréversibles** (COMMIT automatique dans la plupart des SGBD).

### 3.1 CREATE TABLE

**Syntaxe :**
```sql
CREATE TABLE nom_table (
    nom_colonne1 TYPE [CONTRAINTES],
    nom_colonne2 TYPE [CONTRAINTES],
    ...
    [CONTRAINTES DE TABLE]
);
```

**Exemple :**
```sql
CREATE TABLE Clients (
    id    SERIAL PRIMARY KEY,        -- identifiant auto-incrémenté (1, 2, 3…)
    nom   VARCHAR(100) NOT NULL,     -- obligatoire
    email VARCHAR(100) UNIQUE,       -- pas de doublon
    ville VARCHAR(50),               -- optionnel
    age   INT                        -- optionnel
);
```

Résultat (après quelques insertions) :

| id | nom | email | ville | age |
|---|---|---|---|---|
| 1 | Sara | sara@mail.com | Rabat | 30 |
| 2 | Ali | ali@mail.com | Casablanca | 28 |

### 3.2 Auto-incrément selon le SGBD

| SGBD | Syntaxe |
|---|---|
| PostgreSQL | `id SERIAL PRIMARY KEY` ou `id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` |
| MySQL / MariaDB | `id INT AUTO_INCREMENT PRIMARY KEY` |
| SQL Server | `id INT IDENTITY(1,1) PRIMARY KEY` |
| Oracle 12c+ | `id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY` |
| Oracle < 12c | `CREATE SEQUENCE seq_clients START WITH 1 INCREMENT BY 1;` |

### 3.3 ALTER TABLE (modifier une table)

```sql
-- Ajouter une colonne
ALTER TABLE Clients ADD telephone VARCHAR(20);

-- Ajouter une colonne avec valeur par défaut
ALTER TABLE Clients ADD pays VARCHAR(50) DEFAULT 'Maroc';

-- Ajouter plusieurs colonnes d'un coup
ALTER TABLE Clients
    ADD date_naissance DATE,
    ADD actif BOOLEAN DEFAULT TRUE;

-- Ajouter une contrainte après coup
ALTER TABLE Clients
    ADD CONSTRAINT pk_clients PRIMARY KEY (id);

ALTER TABLE Commandes
    ADD CONSTRAINT fk_client FOREIGN KEY (id_client) REFERENCES Clients(id);
```

**Voir la structure :** `\d Clients` (PostgreSQL) ou `DESCRIBE Clients;` (MySQL / Oracle).

### 3.4 DROP (supprimer un objet)

```sql
DROP TABLE Clients;             -- supprime la table ET ses données
DROP TABLE IF EXISTS Clients;   -- pas d'erreur si la table n'existe pas
```

⚠️ **Irréversible** : la structure, les données et les contraintes sont perdues.
⚠️ Si une autre table a une clé étrangère vers `Clients`, la suppression peut être **bloquée**.

### 3.5 TRUNCATE (vider une table)

```sql
TRUNCATE TABLE Clients;                     -- supprime toutes les lignes, garde la structure
TRUNCATE TABLE Clients RESTART IDENTITY;    -- PostgreSQL : remet aussi le compteur id à 1
```

- ⚡ Très rapide (ne parcourt pas ligne par ligne).
- 🚫 Pas de `WHERE` : tout est supprimé.
- ⚠️ Bloqué si la table est référencée par une clé étrangère (utiliser `DELETE` à la place).

### DELETE vs TRUNCATE vs DROP

| Commande | Effet | Données supprimées | Structure supprimée | Vitesse |
|---|---|---|---|---|
| `DELETE` | Ligne par ligne (avec `WHERE` possible) | ✅ | ❌ | Lent si beaucoup de lignes |
| `TRUNCATE` | Vide la table d'un coup | ✅ | ❌ | Très rapide |
| `DROP` | Supprime la table | ✅ | ✅ | Radical |

---

## 4. Les contraintes SQL

Les contraintes sont des **règles** qui empêchent les données invalides d'entrer dans la base.

| Contrainte | Rôle | Exemple |
|---|---|---|
| `NOT NULL` | Valeur obligatoire | `nom VARCHAR(100) NOT NULL` |
| `UNIQUE` | Pas de doublon | `email VARCHAR(100) UNIQUE` |
| `CHECK` | Condition à respecter | `age INT CHECK (age > 0)` |
| `DEFAULT` | Valeur par défaut | `ville VARCHAR(50) DEFAULT 'Inconnue'` |
| `PRIMARY KEY` | Identifiant unique et non nul | `id INT PRIMARY KEY` |
| `FOREIGN KEY` | Lien vers une autre table | `id_client INT REFERENCES Clients(id)` |

### Exemples de violation

```sql
INSERT INTO Clients (nom, age) VALUES (NULL, 20);   -- ❌ NOT NULL : nom obligatoire
INSERT INTO Clients (nom, age) VALUES ('Ali', -5);  -- ❌ CHECK : âge négatif
INSERT INTO Clients (nom, email) VALUES ('Sara', 'ali@mail.com'); -- ❌ UNIQUE si l'email existe déjà
INSERT INTO Clients (nom) VALUES ('Omar');          -- ✅ ville = 'Inconnue' grâce à DEFAULT
```

### Clé primaire simple et composite

```sql
-- Clé primaire simple
CREATE TABLE Clients (
    id  INT PRIMARY KEY,
    nom VARCHAR(100)
);

-- Clé primaire composite : c'est le COUPLE (id_client, id_produit) qui est unique
CREATE TABLE Commandes (
    id_client  INT,
    id_produit INT,
    PRIMARY KEY (id_client, id_produit)
);
```

### Clé étrangère : 3 écritures

```sql
-- 1. Simple (en ligne)
CREATE TABLE Commandes (
    id        INT PRIMARY KEY,
    id_client INT REFERENCES Clients(id)
);

-- 2. Avec un nom de contrainte
CREATE TABLE Commandes (
    id        INT PRIMARY KEY,
    id_client INT,
    CONSTRAINT fk_client FOREIGN KEY (id_client) REFERENCES Clients(id)
);

-- 3. Composite (plusieurs colonnes)
CREATE TABLE LigneCommandes (
    id_commande INT,
    id_produit  INT,
    FOREIGN KEY (id_commande, id_produit) REFERENCES Commandes(id_commande, id_produit)
);
```

### Options de l'intégrité référentielle

Que se passe-t-il pour les commandes quand on supprime/modifie un client ?

| Option | Effet | Exemple |
|---|---|---|
| `ON DELETE CASCADE` | Supprimer le parent supprime les enfants | Client supprimé → ses commandes aussi |
| `ON UPDATE CASCADE` | Modifier la PK du parent met à jour les enfants | `id` 5 → 10 : les commandes passent à `id_client = 10` |
| `ON DELETE RESTRICT` | Interdit la suppression si des enfants existent | Impossible de supprimer un client qui a des commandes |
| `ON DELETE SET NULL` | L'enfant est gardé, mais la FK devient `NULL` | Les commandes restent, `id_client = NULL` |

```sql
CREATE TABLE Commandes (
    id        INT PRIMARY KEY,
    id_client INT,
    FOREIGN KEY (id_client) REFERENCES Clients(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

> 🧠 **À retenir :** CASCADE = tout suit · SET NULL = on coupe le lien · RESTRICT = on interdit.

### Exemple complet : base « magasin »

```sql
CREATE TABLE Clients (
    id    SERIAL PRIMARY KEY,
    nom   VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    ville VARCHAR(50),
    age   INT CHECK (age > 0)
);

CREATE TABLE Produits (
    id   SERIAL PRIMARY KEY,
    nom  VARCHAR(100) NOT NULL,
    prix DECIMAL(10,2) CHECK (prix > 0)
);

CREATE TABLE Commandes (
    id            SERIAL PRIMARY KEY,
    date_commande DATE DEFAULT CURRENT_DATE,
    client_id     INT,
    produit_id    INT,
    FOREIGN KEY (client_id)  REFERENCES Clients(id),
    FOREIGN KEY (produit_id) REFERENCES Produits(id)
);
```

---

## 5. DML : INSERT, UPDATE, DELETE

Le DML agit sur les **données** (pas la structure). Les opérations peuvent être annulées avec `ROLLBACK` dans une transaction.

### 5.1 INSERT (ajouter)

```sql
-- Syntaxe
INSERT INTO table (col1, col2, ...) VALUES (val1, val2, ...);

-- Une ligne, en précisant les colonnes (recommandé)
INSERT INTO Clients (nom, email, ville, age)
VALUES ('Ali', 'ali@gmail.com', 'Rabat', 25);

-- Sans préciser les colonnes (⚠️ l'ordre doit être exactement celui de la table)
INSERT INTO Clients VALUES (1, 'Ali', 'ali@gmail.com', 'Rabat', 25);

-- Plusieurs lignes d'un coup
INSERT INTO Clients (nom, email, ville, age) VALUES
    ('Sara',    'sara@gmail.com',    'Casa',      30),
    ('Youssef', 'youssef@gmail.com', 'Marrakech', 22);
```

### 5.2 UPDATE (modifier)

```sql
-- Syntaxe
UPDATE table SET col1 = val1, col2 = val2 WHERE condition;

-- Changer la ville d'un client
UPDATE Clients SET ville = 'Tanger' WHERE id = 1;

-- Utiliser une expression : +1 an pour les clients de Casa
UPDATE Clients SET age = age + 1 WHERE ville = 'Casa';

-- ⚠️ DANGER : sans WHERE, TOUTES les lignes sont modifiées
UPDATE Clients SET ville = 'Inconnue';

-- Avec sous-requête
UPDATE Commandes
SET montant = (SELECT AVG(montant) FROM Commandes)
WHERE id_client = 5;
```

### 5.3 DELETE (supprimer)

```sql
-- Syntaxe
DELETE FROM table WHERE condition;

-- Supprimer les clients mineurs
DELETE FROM Clients WHERE age < 18;

-- Avec sous-requête : clients ayant une commande à 0
DELETE FROM Clients
WHERE id IN (SELECT id_client FROM Commandes WHERE montant = 0);

-- ⚠️ Sans WHERE : toutes les lignes sont supprimées
DELETE FROM Clients;
```

> 🛡️ **Bon réflexe :** avant un `UPDATE` ou `DELETE`, exécutez d'abord un `SELECT` avec le même `WHERE` pour voir les lignes concernées.

---

## 6. DQL : SELECT

Pour tous les exemples suivants, on utilise cette table :

**Table `Clients`**

| id | nom | email | ville | age |
|---|---|---|---|---|
| 1 | Ali | ali@gmail.com | Rabat | 25 |
| 2 | Sara | sara@gmail.com | Casa | 32 |
| 3 | Youssef | youssef@yahoo.fr | Marrakech | 17 |
| 4 | Amine | NULL | Casa | 40 |
| 5 | Eli | eli@gmail.com | Rabat | 28 |

### Ordre d'écriture d'une requête

```sql
SELECT   colonnes        -- 5. ce qu'on affiche
FROM     table           -- 1. d'où
WHERE    condition       -- 2. filtre des lignes
GROUP BY colonne         -- 3. regroupement
HAVING   condition       -- 4. filtre des groupes
ORDER BY colonne         -- 6. tri
LIMIT    n;              -- 7. nombre de lignes
```

### 6.1 SELECT de base

```sql
SELECT * FROM Clients;             -- toutes les colonnes
SELECT nom, email FROM Clients;    -- certaines colonnes
SELECT nom AS client FROM Clients; -- renommer une colonne (alias)
```

### 6.2 DISTINCT (supprimer les doublons)

```sql
SELECT DISTINCT ville FROM Clients;
```
| ville |
|---|
| Rabat |
| Casa |
| Marrakech |

### 6.3 WHERE (filtrer)

```sql
SELECT nom, age FROM Clients WHERE age > 25;
```
| nom | age |
|---|---|
| Sara | 32 |
| Amine | 40 |
| Eli | 28 |

Opérateurs de comparaison : `=`, `<>` (ou `!=`), `<`, `>`, `<=`, `>=`.

### 6.4 AND, OR, NOT

```sql
-- AND : toutes les conditions vraies
SELECT nom FROM Clients WHERE ville = 'Casa' AND age > 30;       -- Sara, Amine

-- OR : au moins une condition vraie
SELECT nom FROM Clients WHERE ville = 'Casa' OR ville = 'Rabat'; -- Ali, Sara, Amine, Eli

-- NOT : exclure
SELECT nom FROM Clients WHERE NOT ville = 'Casa';                -- Ali, Youssef, Eli
```

### 6.5 LIKE (recherche par motif)

- `%` = 0, 1 ou plusieurs caractères
- `_` = exactement 1 caractère

```sql
SELECT nom FROM Clients WHERE nom LIKE 'A%';             -- commence par A : Ali, Amine
SELECT nom FROM Clients WHERE email LIKE '%@gmail.com';  -- Ali, Sara, Eli
SELECT nom FROM Clients WHERE nom LIKE '_li';            -- 3 lettres finissant par "li" : Ali, Eli
```

### 6.6 IN (appartenance à une liste)

```sql
SELECT nom FROM Clients WHERE ville IN ('Casa', 'Rabat');      -- Ali, Sara, Amine, Eli
SELECT nom FROM Clients WHERE ville NOT IN ('Casa', 'Rabat');  -- Youssef
```

### 6.7 BETWEEN (intervalle, bornes incluses)

```sql
SELECT nom, age FROM Clients WHERE age BETWEEN 20 AND 30;      -- Ali (25), Eli (28)
SELECT nom, age FROM Clients WHERE age NOT BETWEEN 20 AND 30;  -- Sara, Youssef, Amine
```

### 6.8 IS NULL / IS NOT NULL

`NULL` = valeur **inconnue ou absente**. ⚠️ On n'écrit **jamais** `= NULL`.

```sql
SELECT nom FROM Clients WHERE email IS NULL;      -- Amine
SELECT nom FROM Clients WHERE email IS NOT NULL;  -- Ali, Sara, Youssef, Eli
```

### 6.9 ORDER BY (trier)

```sql
SELECT nom, age FROM Clients ORDER BY age ASC;   -- croissant (par défaut)
SELECT nom, age FROM Clients ORDER BY age DESC;  -- décroissant
SELECT nom, ville FROM Clients ORDER BY ville, nom; -- plusieurs critères
```

Résultat de `ORDER BY age DESC` :

| nom | age |
|---|---|
| Amine | 40 |
| Sara | 32 |
| Eli | 28 |
| Ali | 25 |
| Youssef | 17 |

### 6.10 Limiter le nombre de résultats

| SGBD | Syntaxe |
|---|---|
| PostgreSQL / MySQL | `SELECT * FROM Clients LIMIT 5;` |
| SQL Server | `SELECT TOP 5 * FROM Clients;` |
| Oracle | `SELECT * FROM Clients WHERE ROWNUM <= 5;` (ou `FETCH FIRST 5 ROWS ONLY` en 12c+) |

```sql
-- Les 2 clients les plus âgés
SELECT nom, age FROM Clients ORDER BY age DESC LIMIT 2;   -- Amine, Sara
```

---

## 7. Fonctions d'agrégation, GROUP BY, HAVING

### 7.1 Fonctions d'agrégation

| Fonction | Rôle |
|---|---|
| `COUNT()` | Nombre de lignes |
| `SUM()` | Somme |
| `AVG()` | Moyenne |
| `MIN()` | Minimum |
| `MAX()` | Maximum |

```sql
SELECT COUNT(*)  AS nb_clients FROM Clients;  -- 5
SELECT AVG(age)  AS age_moyen  FROM Clients;  -- 28.4
SELECT MAX(age)  AS age_max    FROM Clients;  -- 40
SELECT MIN(age)  AS age_min    FROM Clients;  -- 17
SELECT COUNT(email) FROM Clients;             -- 4 (COUNT(colonne) ignore les NULL)
```

### 7.2 GROUP BY (regrouper)

Regroupe les lignes ayant la même valeur, puis applique la fonction d'agrégation **à chaque groupe**.

```sql
SELECT ville, COUNT(*) AS nb_clients
FROM Clients
GROUP BY ville;
```
| ville | nb_clients |
|---|---|
| Rabat | 2 |
| Casa | 2 |
| Marrakech | 1 |

> ⚠️ Règle : toute colonne du `SELECT` qui n'est pas dans une fonction d'agrégation doit être dans le `GROUP BY`.

### 7.3 HAVING (filtrer les groupes)

- `WHERE` filtre les **lignes** **avant** le regroupement.
- `HAVING` filtre les **groupes** **après** le regroupement.

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

### 7.4 Tout combiner

```sql
SELECT ville, AVG(age) AS age_moyen
FROM Clients
WHERE age > 20              -- 1. on ignore les moins de 20 ans (Youssef)
GROUP BY ville              -- 2. on regroupe par ville
HAVING AVG(age) > 30        -- 3. on garde les villes dont l'âge moyen > 30
ORDER BY age_moyen DESC     -- 4. on trie
LIMIT 3;                    -- 5. on garde 3 lignes max
```
| ville | age_moyen |
|---|---|
| Casa | 36 |

---

## 8. Fonctions utiles

### 8.1 Fonctions sur les chaînes

| Fonction | Rôle | Exemple | Résultat |
|---|---|---|---|
| `UPPER(x)` | Majuscules | `UPPER('ali')` | `ALI` |
| `LOWER(x)` | Minuscules | `LOWER('SARA')` | `sara` |
| `LENGTH(x)` | Longueur | `LENGTH('Youssef')` | `7` |
| `SUBSTRING(x, début, long)` | Extraire | `SUBSTRING('ali@gmail.com', 1, 3)` | `ali` |
| `CONCAT(a, b, …)` | Concaténer | `CONCAT('Ali', ' - ', 'Rabat')` | `Ali - Rabat` |
| `TRIM(x)` | Enlever les espaces aux extrémités | `TRIM('  Rabat ')` | `Rabat` |

```sql
SELECT UPPER(nom)                 AS nom_maj,
       LENGTH(nom)                AS longueur,
       CONCAT(nom, ' - ', ville)  AS nom_ville
FROM Clients;
```
| nom_maj | longueur | nom_ville |
|---|---|---|
| ALI | 3 | Ali - Rabat |
| SARA | 4 | Sara - Casa |
| … | … | … |

### 8.2 Fonctions sur les dates

| Fonction | Rôle | SGBD |
|---|---|---|
| `NOW()` | Date + heure actuelles | PostgreSQL, MySQL |
| `CURRENT_DATE` | Date du jour | Standard |
| `EXTRACT(YEAR FROM d)` | Extraire année / mois / jour | Standard |
| `AGE(d1, d2)` | Différence entre 2 dates | PostgreSQL |
| `d + INTERVAL '1 year'` | Ajouter une durée | PostgreSQL |
| `DATEADD(...)` / `DATEDIFF(...)` | Ajouter / différence | SQL Server, MySQL |

```sql
-- PostgreSQL
SELECT nom,
       EXTRACT(YEAR FROM date_naissance)      AS annee_naiss,
       AGE(CURRENT_DATE, date_naissance)      AS age_exact,
       date_inscription + INTERVAL '1 year'   AS plus_un_an
FROM Clients;
```
| nom | annee_naiss | age_exact | plus_un_an |
|---|---|---|---|
| Ali | 2000 | 25 ans 3 mois 13 jours | 2024-01-01 |
| Sara | 1995 | 29 ans 8 mois 1 jour | 2025-06-15 |

### 8.3 CASE (le « IF » du SQL)

```sql
CASE
    WHEN condition1 THEN valeur1
    WHEN condition2 THEN valeur2
    ELSE valeur_par_defaut
END
```

```sql
SELECT nom, age,
       CASE
           WHEN age < 18               THEN 'Mineur'
           WHEN age BETWEEN 18 AND 60  THEN 'Adulte'
           ELSE 'Senior'
       END AS categorie_age
FROM Clients;
```
| nom | age | categorie_age |
|---|---|---|
| Ali | 25 | Adulte |
| Youssef | 17 | Mineur |
| Amine | 40 | Adulte |

---

## 9. Les jointures

Une **jointure** combine des données de plusieurs tables grâce à une colonne commune (PK / FK).

### Tables d'exemple

**Clients**

| id_client | nom | ville |
|---|---|---|
| 1 | Ali | Rabat |
| 2 | Sara | Casa |
| 3 | Youssef | Marrakech |

**Commandes**

| id_commande | id_client | produit |
|---|---|---|
| 101 | 1 | PC |
| 102 | 2 | Téléphone |
| 103 | 2 | Tablette |

### Vue d'ensemble

| Jointure | Résultat |
|---|---|
| `INNER JOIN` | Seulement les lignes qui correspondent dans les 2 tables |
| `LEFT JOIN` | Toutes les lignes de gauche + correspondances (sinon `NULL`) |
| `RIGHT JOIN` | Toutes les lignes de droite + correspondances (sinon `NULL`) |
| `FULL OUTER JOIN` | Toutes les lignes des 2 tables |
| `CROSS JOIN` | Toutes les combinaisons (produit cartésien) |
| `SELF JOIN` | Une table jointe avec elle-même |

### 9.1 INNER JOIN

```sql
SELECT Clients.nom, Commandes.produit
FROM Clients
INNER JOIN Commandes ON Clients.id_client = Commandes.id_client;
```
| nom | produit |
|---|---|
| Ali | PC |
| Sara | Téléphone |
| Sara | Tablette |

➡️ Youssef n'apparaît pas : il n'a aucune commande.

### 9.2 LEFT JOIN

```sql
SELECT Clients.nom, Commandes.produit
FROM Clients
LEFT JOIN Commandes ON Clients.id_client = Commandes.id_client;
```
| nom | produit |
|---|---|
| Ali | PC |
| Sara | Téléphone |
| Sara | Tablette |
| Youssef | NULL |

➡️ Astuce : trouver les clients **sans commande** :
```sql
SELECT c.nom
FROM Clients c
LEFT JOIN Commandes cmd ON c.id_client = cmd.id_client
WHERE cmd.id_commande IS NULL;   -- Youssef
```

### 9.3 RIGHT JOIN

```sql
SELECT Clients.nom, Commandes.produit
FROM Clients
RIGHT JOIN Commandes ON Clients.id_client = Commandes.id_client;
```
Toutes les commandes sont gardées. Ici chaque commande a un client, donc même résultat que l'INNER JOIN. Si une commande avait `id_client = 9` (inexistant), on verrait `NULL | produit`.

### 9.4 FULL OUTER JOIN

```sql
SELECT Clients.nom, Commandes.produit
FROM Clients
FULL OUTER JOIN Commandes ON Clients.id_client = Commandes.id_client;
```
| nom | produit |
|---|---|
| Ali | PC |
| Sara | Téléphone |
| Sara | Tablette |
| Youssef | NULL |

> ⚠️ Non supporté par MySQL (on combine `LEFT JOIN` + `UNION` + `RIGHT JOIN`).

### 9.5 CROSS JOIN

```sql
SELECT Clients.nom, Commandes.produit
FROM Clients
CROSS JOIN Commandes;
```
➡️ 3 clients × 3 commandes = **9 lignes** (Ali-PC, Ali-Téléphone, Ali-Tablette, Sara-PC, …).

### 9.6 SELF JOIN

Exemple : une table `Employes(id, nom, id_chef)` où `id_chef` référence un autre employé.

```sql
SELECT e.nom AS employe, c.nom AS chef
FROM Employes e
LEFT JOIN Employes c ON e.id_chef = c.id;
```

### 9.7 Jointures multiples (avec alias)

```sql
SELECT c.nom, p.nom_produit, p.prix
FROM Clients c
JOIN Commandes cmd ON c.id_client   = cmd.id_client
JOIN Produits  p   ON cmd.id_produit = p.id_produit;
```
| nom | nom_produit | prix |
|---|---|---|
| Ali | PC | 5000 |
| Sara | Téléphone | 3000 |
| Sara | Tablette | 2500 |

> 💡 `JOIN` tout seul = `INNER JOIN`. Les alias (`c`, `cmd`, `p`) raccourcissent l'écriture.

---

## 10. Transactions (COMMIT, ROLLBACK, SAVEPOINT)

Une **transaction** est un groupe d'instructions exécutées comme **une seule unité** :
👉 **soit tout réussit, soit rien n'est appliqué.**

### Cycle d'une transaction

```text
BEGIN  →  INSERT / UPDATE / DELETE ...  →  COMMIT   (tout valider)
                                        ↘  ROLLBACK (tout annuler)
```

### COMMIT : valider

```sql
BEGIN;
UPDATE Clients SET ville = 'Rabat' WHERE id = 1;
INSERT INTO Commandes (id_client, produit) VALUES (1, 'PC Portable');
COMMIT;   -- les deux modifications sont enregistrées définitivement
```

### ROLLBACK : annuler

Exemple du **virement bancaire** de 100 DH du compte A vers le compte B :

```sql
BEGIN;
UPDATE Comptes SET solde = solde - 100 WHERE id = 'A';  -- débiter A
UPDATE Comptes SET solde = solde + 100 WHERE id = 'B';  -- créditer B
-- ❌ erreur détectée (ex. le compte B n'existe pas)
ROLLBACK; -- A retrouve son argent, rien n'a changé
```

**Sans transaction :** si le 2ᵉ `UPDATE` échoue, les 100 DH sont débités de A mais jamais crédités à B → **argent perdu** !

### SAVEPOINT : point de sauvegarde intermédiaire

```sql
BEGIN;
UPDATE Clients SET ville = 'Casa'  WHERE id = 3;
SAVEPOINT sp1;
UPDATE Clients SET ville = 'Rabat' WHERE id = 4;
ROLLBACK TO sp1;   -- annule seulement ce qui suit sp1
COMMIT;            -- seule la modification de id = 3 est conservée
```

---

## 11. Index et vues

### 11.1 Index

Un **index** est une structure qui **accélère les recherches**, comme l'index à la fin d'un livre 📖.

```sql
CREATE INDEX idx_nom ON Clients(nom);

-- Cette requête devient beaucoup plus rapide sur une grande table
SELECT * FROM Clients WHERE nom = 'Sara';
```

| ✅ Avantages | ⚠️ Inconvénients |
|---|---|
| Recherches (`WHERE`) et tris (`ORDER BY`) plus rapides | Prend de l'espace disque |
| Optimise les requêtes complexes | Ralentit un peu `INSERT`, `UPDATE`, `DELETE` (l'index doit être mis à jour) |

> 💡 Les clés primaires et colonnes `UNIQUE` sont indexées automatiquement.

### 11.2 Vues

Une **vue** (`VIEW`) est une **table virtuelle** définie par une requête. Elle ne stocke pas de données : elle ré-exécute la requête à chaque utilisation.

```sql
-- Syntaxe
CREATE VIEW nom_vue AS
SELECT colonnes FROM table WHERE condition;

-- Exemple
CREATE VIEW vue_clients_rabat AS
SELECT nom, email FROM Clients
WHERE ville = 'Rabat';

-- Utilisation : comme une table normale
SELECT * FROM vue_clients_rabat;
```
| nom | email |
|---|---|
| Ali | ali@gmail.com |
| Eli | eli@gmail.com |

| ✅ Avantages | ⚠️ Inconvénients |
|---|---|
| Simplifie les requêtes complexes | Peut être lente si la requête est lourde |
| Sécurité : on cache certaines colonnes | Pas toujours modifiable directement |
| Réutilisation du code SQL | |

---

## 12. Récapitulatif

### Les commandes essentielles

| Besoin | Commande |
|---|---|
| Créer une table | `CREATE TABLE t (...);` |
| Ajouter une colonne | `ALTER TABLE t ADD col TYPE;` |
| Supprimer une table | `DROP TABLE t;` |
| Vider une table | `TRUNCATE TABLE t;` |
| Ajouter une ligne | `INSERT INTO t (a, b) VALUES (1, 2);` |
| Modifier des lignes | `UPDATE t SET a = 1 WHERE ...;` |
| Supprimer des lignes | `DELETE FROM t WHERE ...;` |
| Lire des données | `SELECT ... FROM t WHERE ... ORDER BY ...;` |
| Compter / moyenne par groupe | `SELECT x, COUNT(*) FROM t GROUP BY x HAVING ...;` |
| Relier deux tables | `SELECT ... FROM a JOIN b ON a.id = b.a_id;` |
| Transaction | `BEGIN; ... COMMIT;` / `ROLLBACK;` |
| Accélérer une recherche | `CREATE INDEX idx ON t(col);` |
| Table virtuelle | `CREATE VIEW v AS SELECT ...;` |

### Pièges classiques ⚠️
1. `UPDATE` / `DELETE` **sans `WHERE`** → toutes les lignes sont touchées.
2. `= NULL` ne fonctionne pas → utiliser **`IS NULL`**.
3. `WHERE` filtre **avant** le regroupement, `HAVING` **après**.
4. `DROP` supprime la structure, `TRUNCATE` / `DELETE` gardent la table.
5. `INSERT` sans liste de colonnes → l'ordre des valeurs doit être exactement celui de la table.

---

## Exercices d'application

Avec la base « magasin » (Clients, Produits, Commandes) :

1. Créer les trois tables avec toutes les contraintes (`NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`).
2. Insérer 5 clients, 4 produits et 6 commandes.
3. Afficher les clients de Casa âgés de plus de 25 ans, triés par nom.
4. Afficher les clients dont l'email se termine par `@gmail.com`.
5. Compter le nombre de clients par ville et garder les villes ayant au moins 2 clients.
6. Afficher pour chaque commande : nom du client, nom du produit, prix.
7. Afficher les clients qui n'ont passé **aucune** commande.
8. Augmenter de 10 % le prix de tous les produits dont le prix est inférieur à 100, dans une transaction.
9. Créer une vue `vue_commandes_detail` qui reprend la requête de la question 6.

<details>
<summary>💡 Corrigé (questions 3 à 7)</summary>

```sql
-- 3
SELECT * FROM Clients WHERE ville = 'Casa' AND age > 25 ORDER BY nom;

-- 4
SELECT * FROM Clients WHERE email LIKE '%@gmail.com';

-- 5
SELECT ville, COUNT(*) AS nb FROM Clients GROUP BY ville HAVING COUNT(*) >= 2;

-- 6
SELECT c.nom, p.nom AS produit, p.prix
FROM Commandes cmd
JOIN Clients  c ON cmd.client_id  = c.id
JOIN Produits p ON cmd.produit_id = p.id;

-- 7
SELECT c.nom
FROM Clients c
LEFT JOIN Commandes cmd ON c.id = cmd.client_id
WHERE cmd.id IS NULL;

-- 8
BEGIN;
UPDATE Produits SET prix = prix * 1.10 WHERE prix < 100;
COMMIT;

-- 9
CREATE VIEW vue_commandes_detail AS
SELECT c.nom, p.nom AS produit, p.prix
FROM Commandes cmd
JOIN Clients  c ON cmd.client_id  = c.id
JOIN Produits p ON cmd.produit_id = p.id;
```
</details>

---

<p align="center">
⬅️ <a href="02-diagramme-er.md">Partie 2 : Diagramme ER</a> · 🏠 <a href="README.md">Sommaire</a>
</p>
