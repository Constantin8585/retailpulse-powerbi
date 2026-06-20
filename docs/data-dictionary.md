# Dictionnaire de données — NordRetail Analytics Platform

> Document de gouvernance décrivant le modèle sémantique Power BI du projet NordRetail.
> Dernière mise à jour : 19 juin 2026 — version 1.0

---

## 1. Vue d'ensemble

**Contexte.** NordRetail est une enseigne de distribution paneuropéenne (magasins physiques et canal e-commerce). Ce modèle sémantique fournit une vision gouvernée des indicateurs clés de vente, de stock et de clientèle.

**Type de modèle.** Schéma en étoile (star schema) : deux tables de faits entourées de leurs dimensions partagées.

**Source des données.** Jeu de données Contoso (généré via Contoso Data Generator V2), complété par deux tables générées sur mesure (`FactInventory`, `DimPromotion`) et une table de sécurité (`DimUser`).

### Liste des tables

| Table | Type | Grain | Volume approx. |
|---|---|---|---|
| FactSales | Fait | Ligne de commande (produit × commande) | ~200 000 |
| FactInventory | Fait | Produit × magasin × mois | ~700 000 |
| DimDate | Dimension | Un jour | ~3 650 |
| DimProduct | Dimension | Un produit | ~2 500 |
| DimCustomer | Dimension | Un client | ~105 000 |
| DimStore | Dimension | Un magasin | ~75 |
| DimPromotion | Dimension | Une campagne promotionnelle | 61 |
| DimUser | Sécurité | Un utilisateur | démo |

---

## 2. Tables de faits

### FactSales

| Attribut | Valeur |
|---|---|
| **Type** | Table de faits |
| **Description** | Transactions de vente de l'enseigne NordRetail |
| **Grain** | Une ligne par produit dans une commande |
| **Colonnes clés** | OrderKey, LineNumber, ProductKey, CustomerKey, StoreKey, OrderDate, PromotionKey |
| **Colonnes de mesure** | Quantity, UnitPrice, NetPrice, UnitCost, ExchangeRate |
| **Mesures associées** | Revenue, Total Cost, Gross Margin, Gross Margin %, Quantity Sold, et toutes les mesures de time intelligence |
| **Propriétaire métier** | Équipe Ventes (fictif) |
| **Sensibilité** | Faible — aucune donnée personnelle |

### FactInventory

| Attribut | Valeur |
|---|---|
| **Type** | Table de faits |
| **Description** | Relevés de stock (snapshot mensuel de fin de mois) par produit et par magasin |
| **Grain** | Une ligne par produit × magasin × mois |
| **Colonnes clés** | ProductKey, StoreKey, DateKey |
| **Colonnes de mesure** | StockLevel, ReorderThreshold, DaysOfStock |
| **Mesures associées** | Stock Level, Days of Stock, Out of Stock Products, Stock Coverage Alert |
| **Propriétaire métier** | Équipe Supply Chain (fictif) |
| **Sensibilité** | Faible |
| **Note de conception** | Grain mensuel (et non journalier) pour éviter une explosion volumétrique. Assortiment par magasin : chaque magasin ne référence qu'un sous-ensemble du catalogue. Magasins fermés exclus. |

---

## 3. Tables de dimensions

### DimDate

| Attribut | Valeur |
|---|---|
| **Description** | Calendrier analytique. Marquée comme table de dates officielle du modèle. |
| **Grain** | Un jour (2015-01-01 → 2024-12-31) |
| **Colonnes principales** | Date, DateKey, Year, Quarter, YearMonth, Month, MonthNumber, DayofWeek, WorkingDay |
| **Sensibilité** | Aucune |

### DimProduct

| Attribut | Valeur |
|---|---|
| **Description** | Référentiel produit : catégorie, sous-catégorie, marque, prix |
| **Grain** | Un produit (SKU) |
| **Colonnes principales** | ProductKey, ProductCode, ProductName, Brand, Manufacturer, CategoryName, SubCategoryName, Cost, Price |
| **Sensibilité** | Faible |

### DimCustomer

| Attribut | Valeur |
|---|---|
| **Description** | Référentiel client |
| **Grain** | Un client |
| **Colonnes principales** | CustomerKey, Continent, Country, City, Age, Gender, Occupation |
| **Colonnes sensibles (PII)** | GivenName, Surname, StreetAddress, ZipCode, Birthday, Latitude, Longitude |
| **Sensibilité** | **ÉLEVÉE** — contient des données personnelles identifiantes |
| **Mesure de protection** | Fiche client en drill-through anonymisée (CustomerKey uniquement, pas de nom/adresse). PII à exclure des rapports diffusés. |

### DimStore

| Attribut | Valeur |
|---|---|
| **Description** | Référentiel magasin : pays, région, surface, statut |
| **Grain** | Un magasin |
| **Colonnes principales** | StoreKey, StoreCode, CountryName, State, Description, SquareMeters, Status, OpenDate, CloseDate |
| **Sensibilité** | Faible |
| **Usage sécurité** | Colonne CountryName utilisée comme axe de filtrage du RLS régional |

### DimPromotion

| Attribut | Valeur |
|---|---|
| **Description** | Référentiel des campagnes promotionnelles |
| **Grain** | Une campagne |
| **Colonnes principales** | PromotionKey, PromotionType, PromotionName, StartDate, EndDate, DiscountRate |
| **Sensibilité** | Faible |
| **Note** | Inclut une ligne « Aucune promotion » (clé 0) pour les ventes hors campagne |

### DimUser (table de sécurité)

| Attribut | Valeur |
|---|---|
| **Description** | Table d'association utilisateur → périmètre, support du RLS dynamique |
| **Grain** | Un utilisateur |
| **Colonnes** | Email, Region, Category, Role |
| **Sensibilité** | Modérée (emails) |
| **Note** | Non reliée au modèle ; interrogée via LOOKUPVALUE dans les règles RLS. Convention « ALL » = accès total. |

---

## 4. Mesures DAX

### 01 — Sales

| Mesure | Formule | Description |
|---|---|---|
| Revenue | `SUMX(FactSales, FactSales[Quantity] * FactSales[NetPrice])` | Chiffre d'affaires net |
| Quantity Sold | `SUM(FactSales[Quantity])` | Quantité totale vendue |
| Total Cost | `SUMX(FactSales, FactSales[Quantity] * FactSales[UnitCost])` | Coût total des ventes |
| Gross Margin | `[Revenue] - [Total Cost]` | Marge brute en valeur |
| Gross Margin % | `DIVIDE([Gross Margin], [Revenue])` | Taux de marge brute |

### 02 — Time Intelligence

| Mesure | Formule | Description |
|---|---|---|
| Revenue LY | `CALCULATE([Revenue], SAMEPERIODLASTYEAR(DimDate[Date]))` | CA de l'année précédente |
| Revenue YoY | `[Revenue] - [Revenue LY]` | Écart annuel en valeur |
| Revenue YoY % | `DIVIDE([Revenue YoY], [Revenue LY])` | Écart annuel en pourcentage |
| Revenue YTD | `TOTALYTD([Revenue], DimDate[Date])` | Cumul annuel à date |

> Variantes temporelles également disponibles via le **calculation group** « Time Intelligence » (Current, MTD, QTD, YTD, Previous Year, YoY Difference, YoY Percentage), applicables dynamiquement à toute mesure.

### 03 — Customer

| Mesure | Formule | Description |
|---|---|---|
| Active Customers | `DISTINCTCOUNT(FactSales[CustomerKey])` | Clients ayant acheté |
| Repeat Customers | `COUNTROWS(FILTER(VALUES(FactSales[CustomerKey]), CALCULATE(DISTINCTCOUNT(FactSales[OrderKey])) > 1))` | Clients avec plus d'une commande |
| Customer Retention Rate | `DIVIDE([Repeat Customers], [Active Customers])` | Taux de clients fidèles |

### 04 — Operations

| Mesure | Formule | Description |
|---|---|---|
| Stock Level | `SUM(FactInventory[StockLevel])` | Niveau de stock |
| Days of Stock | `AVERAGE(FactInventory[DaysOfStock])` | Jours de couverture moyens |
| Out of Stock Products | `CALCULATE(DISTINCTCOUNT(FactInventory[ProductKey]), FactInventory[StockLevel] = 0)` | Nombre de produits en rupture |
| Stock Coverage Alert | `IF([Days of Stock] < 7, "Alert", "OK")` | Alerte de couverture sous seuil |

---

## 5. Sécurité et sensibilité des données

### Données personnelles (PII)

Les colonnes suivantes de **DimCustomer** sont identifiantes et relèvent du RGPD :
GivenName, Surname, StreetAddress, ZipCode, Birthday, Latitude, Longitude.

**Mesures de protection appliquées :**
- Aucune de ces colonnes n'est exposée dans les rapports diffusés.
- La fiche client (drill-through) est anonymisée : elle s'appuie sur CustomerKey et des attributs non identifiants (Occupation, Continent, Age), jamais sur le nom ou l'adresse.

### Row-Level Security (RLS)

| Rôle | Table filtrée | Logique |
|---|---|---|
| Regional Manager | DimStore | L'utilisateur ne voit que les magasins de sa région (CountryName), résolue via DimUser. « ALL » = accès total. |

**Règle RLS (Regional Manager) :**
```
VAR UserRegion =
    LOOKUPVALUE(DimUser[Region], DimUser[Email], USERPRINCIPALNAME())
RETURN
    UserRegion = "ALL" || DimStore[CountryName] = UserRegion
```

### Personas et accès

| Persona | Accès attendu |
|---|---|
| Executives | Toutes régions, toutes catégories |
| Regional Managers | Leur région uniquement (RLS) |
| Category Buyers | Leur périmètre produit |
| Data Analysts | Modèle certifié pour self-service |
| DPO / Governance | Lecture seule, vigilance PII |

---

*Document conçu dans le cadre du projet d'apprentissage NordRetail Analytics Platform.*
