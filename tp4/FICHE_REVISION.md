# Fiche de révision — TP4 : Monitoring, FinOps & Sécurité Azure (ShopEasy)

> **Thème :** exploiter une architecture sous l'angle **observabilité · FinOps · sécurité · audit ·
> gouvernance**, et produire une **note de recommandations DSI**. Compétences C21, C24, C25, C26.
>
> **4 piliers opérationnels d'une plateforme mature :** observabilité, FinOps, sécurité, gouvernance.

---

## 0. Les bases du TP (à savoir expliquer avec ses mots)

Ce TP regarde une infra **déjà déployée et exploitée** (TP1→TP3) sous l'angle du **pilotage**. Quatre
notions à bien distinguer :

- **Le monitoring (surveillance) :** observer si « **ça marche** » à l'aide d'indicateurs et de seuils.
  - *Question :* « Est-ce que ça fonctionne ? » · *Ex. :* CPU VM > 85 %.
  - **Pourquoi ?** Détecter les pannes/saturations **avant** l'utilisateur. **Comment ?** Métriques +
    **alertes** dans **Azure Monitor**.
- **L'observabilité :** la capacité à **comprendre pourquoi** ça ne marche pas, à partir des données
  produites par le système.
  - *Question :* « Pourquoi ça ne fonctionne pas ? » · *Ex. :* les **logs** montrent une erreur de
    connexion à la base.
  - **3 signaux :** **métriques** (un problème existe) · **logs** (le contexte) · **traces** (le
    chemin d'une requête à travers les composants).
  - **Monitoring vs observabilité :** le monitoring **alerte** sur un symptôme connu ; l'observabilité
    **explique** des problèmes même imprévus.
- **Le FinOps :** la discipline qui croise **finance + technique + opérations** pour **comprendre,
  piloter et optimiser** les dépenses cloud — pas seulement réduire, mais **maximiser la valeur**.
  - **Pourquoi ?** Le cloud rend la dépense flexible donc facile à disperser. **Comment ?** Tags,
    budgets, alertes de coût, rightsizing, extinction des environnements hors prod.
- **La sécurité & la gouvernance :** protéger (identités, réseau, données) et **encadrer** l'usage
  (qui peut faire quoi, règles imposées) pour que la flexibilité ne devienne pas du désordre.
  - **Audit :** « Qui a fait quoi ? » via l'**Activity Log** et les logs de diagnostic.

**Le livrable :** une **note DSI** = des constats traduits en **risques** et en **recommandations
priorisées et argumentées**, pas une liste brute de problèmes.

---

## 1. Exploiter = passer du déploiement au pilotage

**Objectifs de l'exploitation :** disponibilité · performance · sécurité · maintenabilité · maîtrise
des coûts · **traçabilité**.
⚠️ Un environnement cloud **non surveillé** peut paraître fonctionner tout en accumulant des risques :
coûts cachés, absence de sauvegarde, ports ouverts, droits excessifs, logs non collectés, aucune alerte.

---

## 2. Monitoring vs observabilité vs audit vs sécurité

| Concept           | Question centrale               | Exemple                                   |
| ----------------- | ------------------------------- | ----------------------------------------- |
| **Monitoring**    | Est-ce que ça fonctionne ?      | CPU VM > 85 %                             |
| **Observabilité** | Pourquoi ça ne fonctionne pas ? | Logs : erreur de connexion à la base      |
| **Audit**         | Qui a fait quoi ?               | Un utilisateur a modifié une règle réseau |
| **Sécurité**      | Le système est-il protégé ?     | Un port d'admin est exposé à Internet     |

**3 signaux classiques :** **métriques** (numériques dans le temps) · **logs** (événements textuels) ·
**traces** (suivi d'une requête à travers les composants). *Métrique = un problème existe ; log = le
contexte ; trace = le chemin.*

---

## 3. Azure Monitor : architecture

`Sources (VM, SQL, Storage, VNet)` → `Collecte (metrics, logs, events)` → `Analyse (dashboards,
requêtes, workbooks)` → `Action (alertes, tickets, corrections)`.

| Composant                          | Rôle                                                     |
| ---------------------------------- | -------------------------------------------------------- |
| **Metrics**                        | Stockage/analyse de métriques numériques                 |
| **Logs / Log Analytics Workspace** | Centraliser, interroger et **corréler** les logs         |
| **Alert Rules**                    | Déclenchement sur condition                              |
| **Action Groups**                  | Destinataires/actions d'une alerte                       |
| **Workbooks / Dashboards**         | Rapports interactifs / tableaux de bord                  |
| **Activity Log**                   | Journal des **opérations de gestion** sur les ressources |

**Métriques vs logs :** métriques = données agrégées, alerte rapide sur seuil, coût prévisible.
Logs = événements détaillés, diagnostic/investigation, coût lié au volume ingéré/conservé.

---

## 4. Construire une stratégie de monitoring

Ne pas tout surveiller pareil : **partir des objectifs métier/opérationnels**.
> **Avant de créer une alerte :** « si elle se déclenche, quelqu'un doit-il vraiment **agir** ? »
> Si non → c'est une info de **dashboard**, pas une alerte.

**Axes d'indicateurs :** Disponibilité · Performance · Capacité · Sécurité · Coût · Qualité d'exploitation.

**SLI / SLO / SLA :**

- **SLI** = indicateur mesurable (ex. taux de dispo).
- **SLO** = objectif **interne** sur un SLI (ex. 99,5 %/mois).
- **SLA** = engagement **contractuel** avec un client.

---

## 5. Alertes & gestion des incidents

**Anatomie d'une alerte :** Condition · Période (fenêtre) · Sévérité (critique/majeur/mineur/info) ·
Action group · **Runbook** (procédure de diagnostic/correction).

**Fatigue d'alerte :** trop d'alertes tue l'alerte. Distinguer : critiques (action immédiate) /
avertissement (analyse) / info (reporting seul).

**Cycle d'incident :** Détecter → Qualifier → Diagnostiquer → Corriger → **Capitaliser**.

---

## 6. FinOps sur Azure

**FinOps** = finance + technologie + opérations pour **comprendre, piloter et optimiser** les dépenses
cloud. Pas seulement réduire : **maximiser la valeur** par rapport au coût. Le cloud rend la dépense
flexible mais facile à disperser (sans tags/budgets/alertes/gouvernance).

**3 objectifs :** rendre les coûts **visibles** · **responsabiliser** les équipes · **optimiser en continu**.

**Outils :** Cost Management · Budgets · Cost alerts · Pricing Calculator · Tags · Reservations/savings
plans · **Advisor** (recommandations).

**Axes d'optimisation :** supprimer le non utilisé · éteindre les env hors prod · rightsizing VM ·
bon niveau de service storage/BDD · réservations si usage stable · budgets+alertes par env · tags obligatoires.
> Une bonne reco FinOps précise : **ressource concernée, problème, impact estimé, risque, action**.

---

## 7. Sécurité Azure

**Responsabilité partagée :** fournisseur = infra physique/hyperviseur/dispo des services ;
client = comptes, rôles, données, config réseau, chiffrement, journalisation, gouvernance.

**Identité & accès :** Utilisateur / Groupe / **Rôle RBAC** / **Scope** (mgmt group → subscription → RG → resource).
**Moindre privilège** : droits strictement nécessaires. ⚠️ Donner **Owner** par facilité = mauvaise
pratique ; Contributor (ou plus restreint) suffit souvent, Reader pour la lecture seule.

**Sécurité réseau (bonnes pratiques) :** fermer les ports inutiles · limiter SSH/RDP à des IP autorisées ·
isoler les bases (jamais exposées) · segmenter les subnets · **journaliser les changements**.

**Microsoft Defender for Cloud :** pilotage de la **posture** de sécurité.

- **Secure score** (indicateur synthétique) · **Recommandations** · **Regulatory compliance** · **Workload protection**.
- Prioriser les recommandations selon : exposition, sensibilité des données, coût de correction,
  impact opérationnel, conformité, criticité métier. *Une reco doit être traduite en action.*

---

## 8. Audit & traçabilité

**Pourquoi ?** Qui a modifié ? Quand ? Quelle action a provoqué un incident ? Ressources critiques
journalisées ? Peut-on **prouver** qu'une mesure existe ?

**Sources de traces :** **Activity Log** (opérations de gestion) · Resource Logs · Entra ID logs
(connexions/identités) · **Diagnostic settings** (export vers Log Analytics/Storage/Event Hub) ·
Defender for Cloud · Azure Policy.

**Notion de preuve :** capture de config, export de logs, dashboard, rapport d'audit, historique
d'activité, règle Azure Policy, revue d'accès.

---

## 9. Gouvernance Azure

Éviter que la flexibilité ne devienne du désordre. **Niveaux :** Management Group → Subscription →
Resource Group → **Tags** → **Azure Policy** (imposer/auditer des règles) → **RBAC**.

**Politique de tags recommandée :** `Environment`, `Owner`, `Application`, `CostCenter`,
`Criticality` (low/medium/high), `DataSensitivity` (public/internal/confidential).

---

## 10. Analyse de risques & note DSI

**Matrice risque/impact (exemples) :**

| Risque               | Cause                             | Mesure corrective                    |
| -------------------- | --------------------------------- | ------------------------------------ |
| VM exposée en SSH    | NSG trop permissif                | Restriction IP, Bastion, MFA         |
| Absence de logs      | Diagnostic settings non configuré | Centraliser dans Log Analytics       |
| Coût inattendu       | Ressources oubliées               | Budget, tags, alertes, suppression   |
| Droits excessifs     | Rôle Owner trop large             | RBAC minimal + revue d'accès         |
| Stockage non protégé | Config faible                     | Chiffrement, accès privé, sauvegarde |

**Priorisation :** Haute (compromission / indispo majeure / coût imminent) · Moyenne · Basse.

**Note DSI (structure) :** Contexte → Constats → Risques → Recommandations (priorité + justification)
→ Plan d'action (court/moyen/long terme) → **Indicateurs de suivi**.
> Une DSI attend une **décision argumentée, lisible et priorisée** — pas une liste brute de problèmes.

---

## ✅ Questions types — avec réponses

1. **Monitoring vs observabilité ?**
   Le **monitoring** répond à « est-ce que ça marche ? » (métriques + seuils). L'**observabilité**
   répond à « **pourquoi** ça ne marche pas ? » à partir des métriques, logs et traces. Le premier
   alerte sur un symptôme connu, le second permet de diagnostiquer même l'imprévu.

2. **Pourquoi une alerte doit être actionnable ?**
   Parce qu'une alerte n'a de sens que si quelqu'un doit **agir** quand elle se déclenche. Sinon,
   c'est une info de **dashboard**, pas une alerte. Trop d'alertes inutiles → **fatigue d'alerte**.

3. **Rôle d'un Log Analytics Workspace ?**
   **Centraliser**, **interroger** et **corréler** les logs de plusieurs ressources au même endroit
   (avec le langage de requête KQL) → diagnostic et investigation.

4. **Métriques vs logs ?**
   Les **métriques** sont des valeurs numériques agrégées dans le temps → alerte rapide sur seuil,
   coût prévisible. Les **logs** sont des événements détaillés → diagnostic ; leur coût dépend du
   volume ingéré/conservé.

5. **Pourquoi les tags en FinOps ?**
   Pour **ventiler** les coûts (par application, environnement, centre de coût, propriétaire),
   **responsabiliser** les équipes et **filtrer** les analyses dans Cost Management.

6. **Que pilote un budget ?**
   Un **budget** fixe un seuil de dépense (par subscription/RG) et déclenche des **alertes de coût**
   en cas de dépassement (prévu ou réel) → détection précoce des dérives.

7. **Pourquoi limiter Owner ?**
   **Owner** donne tous les droits, y compris la gestion des accès → risque élevé. Par **moindre
   privilège**, on préfère **Contributor** (ou plus restreint) et **Reader** pour la lecture seule.

8. **À quoi sert l'Activity Log ?**
   C'est le **journal des opérations de gestion** sur les ressources (qui a créé/modifié/supprimé
   quoi, et quand) → base de l'**audit** et de la traçabilité.

9. **Risques d'un port SSH/RDP ouvert (à `0.0.0.0/0`) ?**
   Exposition aux attaques par **brute-force**, prise de contrôle de la machine, rebond vers le reste
   du réseau. À restreindre par **IP autorisées**, **Bastion**/VPN et **MFA**.

10. **Qu'est-ce qu'une reco SI bien formulée ?**
    Elle précise la **ressource concernée**, le **problème**, l'**impact estimé**, le **risque** et
    l'**action** à mener — et elle est **priorisée** (haute / moyenne / basse). Une reco doit pouvoir
    être traduite en **décision**.

> **Synthèse :** une infra cloud n'est **pas prête pour la production** si elle n'est pas surveillée,
> auditée, gouvernée et optimisée.
