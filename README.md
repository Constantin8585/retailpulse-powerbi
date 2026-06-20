# NordRetail Analytics Platform

Plateforme analytique Power BI enterprise-grade pour une enseigne de distribution paneuropéenne fictive. Le projet couvre l'ensemble de la chaîne BI côté Power BI : modélisation sémantique, DAX avancé, sécurité, gouvernance et DevOps.

---

## Description du projet

NordRetail est une enseigne de distribution disposant de magasins physiques, d'un canal e-commerce et de clients répartis dans plusieurs pays. Ce projet construit un modèle sémantique gouverné et un ensemble de rapports orientés métier, permettant à différentes audiences (direction, acheteurs, responsables régionaux, supply chain, marketing) d'exploiter des indicateurs fiables et sécurisés.

Le périmètre est volontairement centré sur Power BI : modélisation, DAX, rapports, sécurité, gouvernance et versioning. La couche data engineering (ingestion, transformation lourde) est hors périmètre ; les données sont fournies sous forme de fichiers prêts à l'emploi.

## Objectifs métier

- Fournir une vision consolidée et fiable du chiffre d'affaires, de la marge et des volumes de vente.
- Suivre les niveaux de stock et alerter sur les ruptures et les couvertures faibles.
- Analyser la base client (segmentation, rétention) dans le respect des données personnelles.
- Garantir que chaque audience n'accède qu'aux données qui la concernent (sécurité par rôle).
- Offrir un modèle certifiable, réutilisable en self-service par les analystes.

## Architecture Power BI

Le modèle suit un **schéma en étoile** : deux tables de faits entourées de dimensions partagées.

**Tables de faits**
- `FactSales` — transactions de vente (grain : ligne de commande)
- `FactInventory` — relevés de stock (grain : produit × magasin × mois)

**Dimensions**
- `DimDate` (table de dates officielle), `DimProduct`, `DimCustomer`, `DimStore`, `DimPromotion`
- `DimUser` — table de sécurité pour le RLS dynamique (non reliée)

Mode de stockage : Import.

## Structure du modèle

```
DimDate ─┐
DimProduct ─┤
DimCustomer ─┼─< FactSales
DimStore ─┤        │
DimPromotion ─┘    │
                   │
DimDate ─┐         │
DimProduct ─┼─< FactInventory
DimStore ─┘
```

Relations one-to-many des dimensions vers les faits, direction de filtre simple, `DimDate` marquée comme table de dates.

## Principales mesures DAX

**Ventes**
- `Revenue = SUMX(FactSales, FactSales[Quantity] * FactSales[NetPrice])`
- `Gross Margin = [Revenue] - [Total Cost]`
- `Gross Margin % = DIVIDE([Gross Margin], [Revenue])`

**Time Intelligence**
- `Revenue LY = CALCULATE([Revenue], SAMEPERIODLASTYEAR(DimDate[Date]))`
- `Revenue YTD = TOTALYTD([Revenue], DimDate[Date])`
- Variantes temporelles centralisées dans un **calculation group** (Current, MTD, QTD, YTD, Previous Year, YoY Difference, YoY Percentage).

**Client**
- `Active Customers = DISTINCTCOUNT(FactSales[CustomerKey])`
- `Customer Retention Rate = DIVIDE([Repeat Customers], [Active Customers])`

**Opérations**
- `Stock Level`, `Days of Stock`, `Out of Stock Products`, `Stock Coverage Alert`

## Rapports

Cinq pages, chacune adaptée à une audience métier :

| Page | Audience | Contenu clé |
|---|---|---|
| Executive | Direction générale | KPI globaux, évolution mensuelle, carte géographique, tops |
| Catégorie | Acheteurs / category managers | Matrice hiérarchique, field parameter, mise en forme conditionnelle, drill-through produit |
| Magasins | Responsables régionaux | Carte, classement, sparklines, drill-through magasin |
| Stocks | Supply chain | Niveau de stock, ruptures, jours de couverture, alertes, table exportable |
| Client | Marketing / CRM | Segmentation, rétention, drill-through fiche client anonymisée |

Fonctionnalités d'expérience : field parameters, drill-through, mise en forme conditionnelle, navigation, thème cohérent.

## Stratégie de sécurité (RLS)

Row-Level Security **dynamique** : un rôle unique dont la règle résout le périmètre de l'utilisateur connecté via la table `DimUser`.

```
VAR UserRegion =
    LOOKUPVALUE(DimUser[Region], DimUser[Email], USERPRINCIPALNAME())
RETURN
    UserRegion = "ALL" || DimStore[CountryName] = UserRegion
```

Les responsables régionaux ne voient que leur région ; les profils « ALL » (direction) voient l'ensemble.

**Données personnelles.** `DimCustomer` contient des PII (nom, adresse, date de naissance, coordonnées). Elles sont exclues des rapports diffusés ; la fiche client en drill-through est anonymisée (CustomerKey et attributs non identifiants uniquement).

## Stratégie DevOps

- Projet versionné au format **PBIP** (fichiers texte `.tmdl`, diffables et compatibles code review).
- Dépôt Git structuré (`src/`, `docs/`, `tests/`, `screenshots/`).
- Stratégie de branches : `main` (production), `develop` (intégration), `feature/*`, `hotfix/*`.
- Contrôles qualité automatisés via Tabular Editor Best Practice Analyzer.
- Déploiement DEV → TEST → PROD via deployment pipelines (à la mise en service).

## Limites du projet

- Données fictives (Contoso) ; les tables de stock et de promotion sont générées et alignées sur les clés réelles.
- Pas de couche data engineering (ingestion / transformation hors périmètre).
- Workspaces, publication, deployment pipelines et certification nécessitent une licence Power BI Pro ou PPU (non déployés dans la version locale).
- Gestion multi-devises non implémentée (table de change disponible mais non exploitée).
- Segmentation client basée sur les attributs disponibles ; pas de scoring RFM/LTV avancé.

## Prochaines améliorations

- Intégration d'un scoring RFM et d'une LTV calculés en DAX.
- Branchement de la dimension promotion sur les ventes (PromotionKey).
- Gestion multi-devises via la table de taux de change.
- Publication complète DEV/TEST/PROD avec deployment pipelines.
- Certification du semantic model dans le service Power BI.

---

## Structure du dépôt

```
.
├── README.md
├── docs/                 # architecture, dictionnaire de données, gouvernance, déploiement
├── src/
│   ├── semantic-model/   # modèle PBIP (.tmdl)
│   └── reports/          # rapports PBIP
├── screenshots/          # captures des pages
└── tests/                # règles Best Practice Analyzer
```

*Projet réalisé dans le cadre d'un parcours d'apprentissage Power BI enterprise.*
