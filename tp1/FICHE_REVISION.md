# Fiche de révision — TP1 : Architecture Cloud Azure (ShopEasy)

> **Thème :** concevoir et justifier une architecture cloud cible pour migrer une application
> web d'entreprise. Compétences C21, C24, C25, C26.

---

## 0. Les bases du TP (à savoir expliquer avec ses mots)

**Le cloud computing**, c'est **louer de l'informatique** (serveurs, stockage, bases de données,
réseau) chez un fournisseur, **à la demande** et **facturée à l'usage**, au lieu d'acheter et
d'héberger ses propres machines.

- **Pourquoi ?** Éviter l'achat initial (CAPEX), ne payer que ce qu'on consomme (OPEX), démarrer en
  quelques minutes, monter/descendre en charge selon le besoin, et déléguer une partie de
  l'exploitation au fournisseur.
- **Comment ?** On crée des **ressources** (VM, base, réseau…) dans des **régions** (datacenters),
  via un portail web, une **CLI** ou du **code (IaC)** — voir TP2.

**Azure**, c'est la **plateforme cloud de Microsoft** (concurrente d'AWS et Google Cloud). Elle
fournit des centaines de services organisés sous une hiérarchie *Tenant → Subscription → Resource
Group → Resource*.

- **Pourquoi Azure ici ?** C'est l'environnement du cours (**Azure for Students**), avec une forte
  intégration identité (**Entra ID**) et gouvernance.
- **Comment on l'utilise au TP1 ?** On **conçoit** une architecture cible et on **justifie** chaque
  choix (réseau, compute, base, sécurité, coûts) — on dessine la cible avant de coder (TP2) ou
  d'exploiter (TP3/TP4).

**Et le monitoring / l'observabilité ?** Surveiller que « ça marche » (métriques, alertes) et
comprendre « pourquoi ça casse » (logs). Abordé en §9 ici, approfondi aux TP3 et TP4.

---

## 1. Du SI traditionnel au cloud

**Limites de l'infra traditionnelle :** délai de mise à disposition, surdimensionnement (payer
les pics), coût initial (CAPEX), maintenance à la charge de l'entreprise, **point de défaillance
unique** (mono-serveur).

**Cloud computing :** fourniture de ressources informatiques **à la demande**, accessibles via le
réseau, configurables rapidement, mesurables et **facturées à l'usage**.

**4 propriétés fondamentales :**

| Propriété                 | Idée                                                            |
| ------------------------- | --------------------------------------------------------------- |
| **Élasticité**            | Augmenter/diminuer les ressources selon le besoin               |
| **Facturation à l'usage** | CAPEX → OPEX. Risque : une ressource oubliée continue de coûter |
| **Automatisation**        | Décrire l'infra (IaC) plutôt que cliquer                        |
| **Services managés**      | Le fournisseur prend en charge une partie de l'exploitation     |

---

## 2. IaaS / PaaS / SaaS

| Modèle   | Le client gère…                             | Exemple Azure                   |
| -------- | ------------------------------------------- | ------------------------------- |
| **IaaS** | OS, runtime, application, données, sécurité | Virtual Machines, VNet          |
| **PaaS** | Code, config, données, droits               | App Service, Azure SQL Database |
| **SaaS** | Utilisateurs, données, config fonctionnelle | Microsoft 365, Dynamics 365     |

> Le choix est un **arbitrage** : contrôle ↔ responsabilité ↔ rapidité ↔ compétences ↔ coût ↔ conformité.
> Plus c'est managé, moins on contrôle, mais moins on administre.

---

## 3. Concepts Azure de base

| Concept               | Rôle                                                       |
| --------------------- | ---------------------------------------------------------- |
| **Tenant (Entra ID)** | Annuaire d'identité de l'organisation                      |
| **Subscription**      | Conteneur de **facturation et gouvernance**                |
| **Resource Group**    | Conteneur logique = unité de cycle de vie (app/env/projet) |
| **Resource**          | Service concret : VM, VNet, Storage, SQL…                  |

- **Région** = zone géographique (enjeux : latence, conformité, résidence des données, services dispo).
- **Availability Zone** = datacenters physiquement séparés dans une région → réduit le risque d'indispo.
- ⚠️ Erreur : tout mettre dans un seul RG. Un RG doit traduire une **logique de cycle de vie**.

---

## 4. Well-Architected Framework (5 piliers)

**Reliability · Security · Cost Optimization · Operational Excellence · Performance Efficiency.**
Cadre de réflexion pour **justifier** une architecture (quoi faire si une ressource tombe ? accès
limités ? coûts suivis ? exploitation standardisée ? capable d'absorber la croissance ?).

---

## 5. Briques de l'architecture cible ShopEasy

```text
Internet → Load Balancer → VM Web 1 / VM Web 2 (subnet applicatif)
                                 ↓
                        Azure SQL Database (données) + Storage Account (documents)
        Entra ID/RBAC (identité)   ·   Azure Monitor (logs/métriques)
                        le tout dans 1 VNet / 1 Resource Group
```

- **VNet** : réseau privé isolé (plage RFC 1918, ex. `10.x.0.0/16`).
- **Subnets** : segmentation par exposition/fonction (public / applicatif / données / admin).
- **NSG** : pare-feu par règles (priorité, source, port, Allow/Deny).
- **VM** : compute IaaS. **Load Balancer** : répartit + sonde de santé.
- **Azure SQL Database** : base managée. **Storage Account** : fichiers/documents (Blob).

**Règles NSG attendues :**

| Flux  | Port | Source                  | Justification                       |
| ----- | ---- | ----------------------- | ----------------------------------- |
| HTTP  | 80   | Internet / LB           | Accès web de test                   |
| HTTPS | 443  | Internet / AG           | Cible pro chiffrée                  |
| SSH   | 22   | **IP admin uniquement** | Admin contrôlée (sinon brute-force) |
| SQL   | 1433 | Subnet applicatif       | Pas d'exposition directe de la base |

---

## 6. Compute : VM vs App Service

| Critère            | VM (IaaS)                     | App Service (PaaS)           |
| ------------------ | ----------------------------- | ---------------------------- |
| Contrôle système   | Élevé (OS, services)          | Limité (plateforme managée)  |
| Administration     | Lourde (patchs, durcissement) | Simplifiée (focus code)      |
| Migration existant | Adaptée                       | Peut demander une adaptation |

> **Dimensionnement** : CPU, mémoire, disque, réseau, OS, budget. Mauvais dimensionnement →
> mauvaise perf **ou** gaspillage. Question type : 1 grosse VM (simple, risque concentré) vs
> plusieurs petites derrière un LB (dispo + scalabilité horizontale).

---

## 7. Base de données : SQL sur VM vs Azure SQL Database

| Critère                 | SQL sur VM                       | Azure SQL Database              |
| ----------------------- | -------------------------------- | ------------------------------- |
| Contrôle                | Très fort (OS + moteur)          | Limité, managé                  |
| Maintenance/sauvegardes | À la charge du client            | Largement simplifiées           |
| Cas TP1                 | Comprendre l'héritage on-premise | Cible cloud moderne recommandée |

**Vigilance BDD :** ne pas exposer à Internet · stratégie de sauvegarde · niveau de service adapté ·
surveiller CPU/DTU/connexions · **moindre privilège** sur les comptes applicatifs.

---

## 8. Identité, droits, gouvernance (RBAC)

- **Entra ID** = identités. **RBAC** = droits sur un **scope** (management group / subscription / RG / resource).
- **Principe du moindre privilège** : minimum de droits, pour la durée nécessaire.
- Rôles : **Owner** (tout, à limiter) · **Contributor** (gère sans droits) · **Reader** (lecture) · **Cost Management Reader**.
- ⚠️ Donner Owner à toute l'équipe = erreur de gouvernance.

---

## 9. Supervision (Azure Monitor) & FinOps

**Superviser** répond à : ça marche ? pourquoi pas ? comment évoluent perf et coûts ?
Métriques utiles VM : **CPU, mémoire, disque, réseau**. LB : requêtes/erreurs/dispo backends.

**Bonne alerte = actionnable** : symptôme + seuil + cible + criticité + action. Trop d'alertes =
bruit ; trop peu = incidents masqués.

**FinOps** = pilotage financier du cloud (visibilité, responsabilisation, optimisation sans dégrader
la valeur). Outils : **Pricing Calculator** (estimer avant), **Cost Management** (suivre), **tags**,
**budgets+alertes**. Optimisations : arrêt planifié VM, rétention des logs, rightsizing SQL, tagging.

---

## 10. Sécurité & nommage

- **Responsabilité partagée** : le fournisseur sécurise l'infra physique ; le client reste responsable
  config, identités, données, accès, secrets. **Utiliser Azure ≠ être sécurisé.**
- Risques ShopEasy : SSH ouvert (→ IP/Bastion/MFA), base publique (→ accès privé), droits trop larges
  (→ RBAC), stockage public (→ privé + versioning), absence de logs (→ Azure Monitor).
- **Nommage** : `rg-shopeasy-dev-frc`, `vnet-…`, `snet-web-…`, `vm-web-01-dev`, storage en minuscules.
- **Tags** : `Application`, `Environment`, `Owner`, `CostCenter`, `Criticality`.

---

## ✅ Questions types — avec réponses

1. **Différence région / Availability Zone ?**
   La **région** est une zone géographique (un ou plusieurs datacenters), choisie pour la latence,
   la conformité, la résidence des données et les services disponibles. L'**Availability Zone** est
   un datacenter physiquement isolé **à l'intérieur** d'une région ; répartir sur plusieurs zones
   réduit le risque d'indisponibilité.

2. **Pourquoi plusieurs subnets ?**
   Pour **segmenter** le réseau par exposition/fonction (public, applicatif, données, admin),
   appliquer des règles NSG différentes par segment, et limiter la propagation en cas de
   compromission.

3. **Rôle d'un NSG ?**
   C'est un **pare-feu** : un jeu de règles (priorité, source, port, Allow/Deny) qui autorise ou
   bloque le trafic entrant/sortant d'un subnet ou d'une carte réseau.

4. **Pourquoi ne pas exposer la base à Internet ?**
   Pour réduire la surface d'attaque (brute-force, fuite de données). La base ne doit être joignable
   que depuis le **subnet applicatif**, en accès privé et avec des comptes à **moindre privilège**.

5. **Quand App Service (PaaS) > VM (IaaS) ?**
   Quand on veut se concentrer sur le **code** sans gérer l'OS ni les patchs : app web standard,
   déploiement et scalabilité simplifiés. La VM reste préférable pour un fort contrôle système ou
   une migration « telle quelle » de l'existant.

6. **Pourquoi le versioning Storage ?**
   Pour conserver les versions précédentes d'un fichier → récupération après suppression/écrasement
   accidentel ou rançongiciel. ⚠️ Le coût s'accumule → ajouter une **règle de cycle de vie**.

7. **Quelles métriques sur une VM ?**
   **CPU**, **mémoire**, **disque** (I/O et espace), **réseau**, et la **disponibilité**.

8. **Rôle de Cost Management ?**
   **Suivre** les coûts réels, les **ventiler** (par tag / resource group), poser des **budgets** et
   des **alertes** de dépassement.

9. **2 mauvaises pratiques de sécurité ?** (en citer 2)
   SSH ouvert à `0.0.0.0/0` · base exposée à Internet · rôle **Owner** donné à toute l'équipe ·
   stockage public · absence de logs/supervision.

10. **Comment justifier une archi à une DSI ?**
    Partir du **besoin métier** (pas du service), s'appuyer sur le **Well-Architected Framework**
    (fiabilité, sécurité, coût, exploitation, performance), montrer les **arbitrages** et **chiffrer**
    les coûts estimés.

> **À retenir :** partir du **besoin métier**, pas du service. Une archi cible doit intégrer au
> minimum : segmentation réseau, base managée, supervision, sécurité cohérente et coûts estimés.
