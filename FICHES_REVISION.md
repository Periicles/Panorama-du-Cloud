# Fiches de révision — Bloc 4 : Optimisation du SI par le Cloud Computing

Synthèses pour réviser les 4 TP (cas fil rouge **ShopEasy**, Azure). Chaque fiche condense le cours
magistral + le sujet de TP : une section **« Les bases du TP »** (c'est quoi / pourquoi / comment),
les concepts clés, des tableaux comparatifs, des commandes, et des **questions types avec réponses**.

| TP      | Thème                                                 | Fiche                                            |
| ------- | ----------------------------------------------------- | ------------------------------------------------ |
| **TP1** | Architecture Cloud Azure (conception d'une cible)     | [`tp1/FICHE_REVISION.md`](tp1/FICHE_REVISION.md) |
| **TP2** | Infrastructure as Code avec Terraform                 | [`tp2/FICHE_REVISION.md`](tp2/FICHE_REVISION.md) |
| **TP3** | Administration & automatisation (CLI · Bash · Python) | [`tp3/FICHE_REVISION.md`](tp3/FICHE_REVISION.md) |
| **TP4** | Monitoring, FinOps, Sécurité, Audit & Gouvernance     | [`tp4/FICHE_REVISION.md`](tp4/FICHE_REVISION.md) |

## Fil conducteur

1. **TP1 — Concevoir :** partir du besoin métier, choisir les services (IaaS/PaaS/SaaS), produire une
   architecture cible justifiée (réseau, VM, LB, base managée, stockage, sécurité, supervision, coûts).
2. **TP2 — Automatiser :** transformer l'architecture en **code Terraform** reproductible, versionné,
   multi-environnements (state, plan/apply, drift, tags, outputs).
3. **TP3 — Exploiter :** administrer et superviser au quotidien via Azure CLI / Bash / Python
   (inventaire, pilotage VM, alertes, FinOps, rapport d'exploitation).
4. **TP4 — Piloter :** observabilité, FinOps, sécurité, audit et gouvernance ; produire une **note DSI**
   priorisée. Les **4 piliers** d'une plateforme mature.

## Notions transverses (reviennent dans plusieurs TP)

- **Responsabilité partagée** · **moindre privilège / RBAC** · **NSG & exposition réseau**.
- **Tags & gouvernance** (Environment, Owner, CostCenter, Application, Criticality).
- **FinOps** : visibilité → responsabilisation → optimisation continue.
- **Supervision** : métriques / logs / alertes actionnables.
- **Région `germanywestcentral`** et **VM Arm64 `Standard_B2pts_v2`** (contraintes Azure for Students).
