# Fiche de révision — TP3 : Administration & Automatisation Azure (ShopEasy)

> **Thème :** passer du **déploiement** à l'**exploitation**. Administrer, superviser, documenter,
> automatiser et optimiser via **Azure CLI · Bash · Python · Azure Monitor**. Compétences C21, C23, C24, C25.

---

## 0. Les bases du TP (à savoir expliquer avec ses mots)

**L'exploitation (ou administration)**, c'est **faire vivre** une infrastructure une fois qu'elle
existe : vérifier qu'elle est disponible, la piloter (démarrer/arrêter), la surveiller, la sécuriser,
en maîtriser les coûts et la documenter. Déployer (TP2) ne suffit pas : il faut **exploiter**.

Les **3 outils** du TP, du plus simple au plus puissant :

- **Azure CLI (`az`)** : un programme en ligne de commande pour piloter Azure depuis un terminal.
  - **Pourquoi plutôt que le portail web ?** La CLI se **documente**, se **rejoue**, s'**automatise**
    et laisse des traces ; le portail (clics) est manuel, non reproductible et peu traçable.
  - **Comment ?** `az login` puis des commandes comme `az vm list`, filtrées avec `--query`.
- **Bash** : le langage de script du terminal Linux/macOS. On **enchaîne** des commandes `az` dans un
  fichier `.sh` pour automatiser une tâche répétitive (inventaire, contrôle, arrêt de VM).
  - **Pourquoi ?** Automatiser = moins d'erreurs, des actions **relançables** et planifiables.
- **Python (+ SDK Azure)** : un vrai langage de programmation, avec les bibliothèques `azure-*`.
  - **Pourquoi Python plutôt que Bash ?** Dès qu'il faut du **traitement structuré** (données, JSON),
    des appels d'API, de la génération de rapports ou de l'orchestration complexe.

**Azure Monitor** est le service de **supervision** d'Azure : il collecte **métriques**, **logs** et
déclenche des **alertes**. (Approfondi au TP4.)

> **Fil rouge :** TP1 conçoit, TP2 déploie, **TP3 exploite au quotidien**, TP4 pilote (gouvernance).

---

## 1. Provisionner n'est pas exploiter

**Provisionnement** = créer les ressources (Terraform). **Exploitation** = la vie d'après : les
ressources sont-elles disponibles, saturées, bien taguées, conformes en coût ? Peut-on inventorier
et automatiser ?

**Les 7 familles d'actions d'exploitation :**

| Famille            | Objectif               | Exemples Azure                                 |
| ------------------ | ---------------------- | ---------------------------------------------- |
| **Inventaire**     | Savoir ce qui existe   | `az resource list`, export CSV, tags           |
| **Administration** | Piloter les ressources | start/stop VM, resize, update tags             |
| **Supervision**    | Observer l'état        | Azure Monitor, metrics, logs, alertes          |
| **Automatisation** | Réduire le manuel      | Bash, Python, scripts planifiés                |
| **Sécurité**       | Réduire l'exposition   | RBAC, NSG, diagnostic settings, audit          |
| **FinOps**         | Maîtriser les coûts    | budgets, tags, arrêt VM, ressources orphelines |
| **Documentation**  | Transmettre            | rapport, procédures, runbooks                  |

> Une VM peut être parfaitement déployée et **mal exploitée** (pas de tag owner, SSH ouvert, aucune
> alerte, aucune procédure d'arrêt, aucun suivi de coût). Le TP3 traite ces sujets.

---

## 2. Azure CLI : outil central

**Pourquoi CLI plutôt que portail ?** Le portail est difficile à **rejouer**, produit peu de traces,
favorise le manuel non standardisé, se prête mal à l'**automatisation**. La CLI se documente, se rejoue,
s'intègre à des scripts/pipelines.

**Commandes de base :** `az login`, `az account show`, `az account set --subscription <id>`,
`az group list -o table`, `az resource list -g rg-shopeasy-dev -o table`.

**Formats de sortie :** `-o json` (programmatique/jq) · `-o table` (lecture humaine) · `-o tsv`
(extraction dans une variable Bash) · `-o yaml` · `-o none`.

**`--query` (JMESPath)** filtre/transforme le JSON :

```bash
az vm list -g "$RG" --query "[].{name:name, size:hardwareProfile.vmSize, location:location}" -o table
az resource list -g "$RG" --query "[?tags.Environment=='dev'].{name:name,type:type}" -o table
```

---

## 3. Inventaire & tags

L'inventaire est **la base** : combien de ressources, quels types, où, taguées ?, IP publiques ?,
VM démarrées/arrêtées ?

**Tags d'exploitation :** `Environment` (dev/rec/prod), `Owner`, `CostCenter`, `Application`,
`Criticality`, `AutoShutdown`. Simples à poser, puissants pour FinOps/audit/inventaire/automatisation.

```bash
az resource tag --ids "<resource-id>" --tags Environment=dev Application=ShopEasy Owner=ops-team
```

---

## 4. Bash : automatiser l'exploitation

**Structure minimale d'un script fiable :**

```bash
#!/usr/bin/env bash
set -euo pipefail          # stoppe sur erreur / variable non définie / échec de pipe
RG="rg-shopeasy-dev"
OUT_DIR="./out"; mkdir -p "$OUT_DIR"
az resource list -g "$RG" --query "[].{name:name,type:type,location:location}" \
  -o table | tee "$OUT_DIR/inventory.txt"
```

**Bonnes pratiques :** `set -euo pipefail` · variables centralisées · répertoires de sortie ·
messages explicites · commandes **idempotentes** (relançables) · journalisation.

Un script est satisfaisant s'il est **lisible, relançable, paramétrable et produit une sortie réutilisable**.

---

## 5. Administration des VM

**Cycle de vie :** créée → démarrée → arrêtée → désallouée → redémarrée → supprimée.
⚠️ **`stop` ≠ `deallocate`** : une VM **arrêtée** (depuis l'OS) peut continuer à facturer le compute ;
une VM **désallouée** libère la ressource de calcul (les **disques restent facturés**).

```bash
az vm get-instance-view -g "$RG" -n "$VM" --query "instanceView.statuses[].displayStatus" -o table
az vm stop|deallocate|start|restart -g "$RG" -n "$VM"
az vm run-command invoke -g "$RG" -n "$VM" --command-id RunShellScript \
  --scripts "uptime && df -h && systemctl status nginx --no-pager"
```

> Ne **jamais** automatiser aveuglément redémarrages/suppressions : périmètre limité, testé, documenté, validé.

---

## 6. Stockage des rapports

Conserver les inventaires/exports/rapports dans un Storage Account (suivi dans le temps) :
`az storage account create` → `az storage container create --auth-mode login` →
`az storage blob upload --name "inventory-$(date +%Y%m%d).txt" --auth-mode login --overwrite true`.
Une équipe mature **historise** les résultats et sait expliquer ce qui a été contrôlé.

---

## 7. Azure Monitor & observabilité

**3 familles d'info :** **métriques** (valeurs numériques dans le temps) · **logs** (événements
détaillés) · **alertes** (seuil/condition atteint).

**Métriques pertinentes :** VM → CPU/réseau/disque/dispo · LB → état des probes, flux ·
Storage → transactions/latence/capacité · SQL → DTU/vCore, connexions, stockage · NSG → flux refusés.

**Créer une alerte métrique :**

```bash
az monitor metrics alert create -g "$RG" -n "alert-high-cpu-vm-web-01" --scopes "$VM_ID" \
  --condition "avg Percentage CPU > 80" --window-size 5m --evaluation-frequency 1m
```

**Alerting :** éviter le trop-plein · relier chaque alerte à une **action attendue** · documenter la
procédure · adapter les seuils à l'env. Une alerte **sans procédure** n'est que du bruit.

---

## 8. FinOps opérationnel

| Action               | Exemple                                 | Bénéfice                     |
| -------------------- | --------------------------------------- | ---------------------------- |
| Taguer               | CostCenter, Owner, Environment          | Ventiler les coûts           |
| Arrêter les VM dev   | Désallocation hors usage                | Réduire la facture compute   |
| Supprimer l'orphelin | Disques, IP publiques, NIC non attachés | Éviter les coûts invisibles  |
| Rightsizing          | Taille adaptée                          | Éviter le surdimensionnement |
| Budgets + alertes    | Par subscription/RG                     | Détecter les dépassements    |

Détecter l'orphelin : `az disk list --query "[?managedBy==null]…"`, `az network public-ip list`.
Le **FinOps est une discipline d'exploitation** : nommer, taguer, surveiller, arrêter, supprimer,
dimensionner, **expliquer**.

---

## 9. Sécurité d'exploitation

La sécurité **se dégrade dans le temps** (règles modifiées, droits excessifs, logs désactivés).
Points de contrôle : **RBAC** (qui a accès ?), **ports exposés** (SSH/RDP ouverts ?), **NSG** (règles
trop larges ?), **secrets** (dans les scripts ? → Key Vault), **logs** (activité tracée ?), **tags**
(responsable identifié ?).
⚠️ Une règle entrante `0.0.0.0/0` vers SSH/RDP = **anomalie** hors labo contrôlé.

---

## 10. Python & SDK Azure

**Quand Python > Bash ?** traitement structuré, appels API, génération de rapports, enrichissement,
intégration, orchestration complexe.
**Libs :** `azure-identity` (`DefaultAzureCredential`), `azure-mgmt-resource`, `azure-mgmt-compute`, `azure-mgmt-monitor`.

```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient
client = ResourceManagementClient(DefaultAzureCredential(), subscription_id)
for r in client.resources.list_by_resource_group("rg-shopeasy-dev"):
    print(r.name, r.type, r.location)
```

---

## 11. Rapport d'exploitation (livrable)

**Structure :** contexte/périmètre · inventaire · état VM/services · contrôle tags/gouvernance ·
contrôle réseau/sécurité · supervision/alertes · analyse FinOps · risques résiduels · recommandations priorisées.
Lisible par **2 publics** : l'équipe technique (rejouer) **et** la DSI (risques, coûts, priorités).
> ❌ « La VM marche. » ✅ « La VM `vm-shopeasy-web-01` est *running*, répond au test HTTP, mais n'a pas
> de tag `CostCenter` : recommandé de l'ajouter et de créer une alerte CPU à 80 %. »

---

## ✅ Questions types — avec réponses

1. **Provisionnement vs exploitation ?**
   Le **provisionnement** crée les ressources (Terraform). L'**exploitation** est la vie d'après :
   les ressources sont-elles disponibles, saturées, bien taguées, conformes en coût, inventoriables
   et automatisables ?

2. **Pourquoi CLI > portail ?**
   La CLI se **documente**, se **rejoue**, s'**intègre** à des scripts/pipelines et laisse des
   traces. Le portail est manuel, difficile à reproduire et peu propice à l'automatisation.

3. **Rôle de `--query` ?**
   C'est un filtre **JMESPath** qui sélectionne et transforme le JSON renvoyé par `az` (ne garder que
   certains champs, filtrer par condition) → sorties ciblées et exploitables.

4. **Pourquoi les tags ?**
   Ce sont des étiquettes (`Environment`, `Owner`, `CostCenter`…) qui servent à la fois au **FinOps**
   (ventiler les coûts), à l'**audit**, à l'**inventaire** et à l'**automatisation**. Simples à poser,
   très puissants.

5. **`stop` vs `deallocate` ?**
   Une VM **arrêtée depuis l'OS** (`stop`) peut **continuer à facturer le compute**. Une VM
   **désallouée** (`deallocate`) libère la ressource de calcul → plus de coût compute, mais les
   **disques restent facturés**.

6. **3 métriques VM web ?**
   **CPU**, **réseau**, **disque** (et disponibilité). Sur le LB on regarde l'état des **probes**.

7. **Pourquoi une alerte = une procédure ?**
   Une alerte doit être **actionnable** : si elle se déclenche, quelqu'un doit savoir **quoi faire**.
   Sans procédure (runbook), elle n'est que du **bruit** et alimente la fatigue d'alerte.

8. **Quand Python > Bash ?**
   Pour le traitement **structuré** (données/JSON), les appels d'**API**, la **génération de
   rapports**, l'enrichissement et l'orchestration complexe. Bash reste bien pour enchaîner des
   commandes simples.

9. **2 risques NSG ?**
   Une règle entrante `0.0.0.0/0` vers **SSH/RDP** (exposition à toute Internet) et des règles
   **trop larges** (ports inutiles ouverts) → surface d'attaque accrue.

10. **3 actions FinOps ?** (en citer 3)
    **Taguer** pour ventiler · **arrêter/désallouer** les VM dev hors usage · **supprimer les
    ressources orphelines** (disques, IP, NIC) · **rightsizing** · **budgets + alertes**.
