# Fiche de révision — TP2 : Infrastructure as Code avec Terraform (ShopEasy)

> **Thème :** automatiser le déploiement de l'architecture TP1 avec **Terraform** : reproductible,
> versionné, multi-environnements, maîtrisé par le coût.

---

## 0. Les bases du TP (à savoir expliquer avec ses mots)

**L'Infrastructure as Code (IaC)**, c'est **décrire son infrastructure dans du code** (fichiers
texte versionnés) au lieu de la créer à la main dans un portail. Le code devient la **source de
vérité** de ce qui doit exister.

**Terraform**, c'est l'outil **IaC** le plus répandu (édité par HashiCorp). On écrit *ce qu'on veut*
(l'**état souhaité**) dans des fichiers `.tf`, et Terraform calcule et exécute les opérations
nécessaires (créer / modifier / détruire) pour y arriver.

- **Pourquoi ?**
  - **Reproductible** : recréer la même infra à l'identique, autant de fois qu'on veut.
  - **Versionné** : l'infra suit le même cycle que le code (Git, revue par PR, historique).
  - **Multi-environnements** : un même code paramétré pour dev / test / prod.
  - **Garde-fou** : `plan` montre ce qui va changer **avant** d'appliquer → moins d'erreurs.
  - **Multi-cloud** : un même outil pour Azure, AWS, GCP… (via des **providers**).
- **Comment ?**
  1. On écrit des ressources en **HCL** (langage déclaratif de Terraform).
  2. `init` télécharge le **provider** (`azurerm` pour Azure).
  3. `plan` prévisualise, `apply` déploie, `destroy` nettoie.
  4. L'état réel est mémorisé dans un fichier **state** (`terraform.tfstate`).

> **Déclaratif vs impératif :** on décrit la **cible** (« je veux 2 VM derrière un LB »), pas la
> suite d'étapes. Terraform se charge du *comment*.

**Lien avec les autres TP :** le TP1 **conçoit** l'architecture, le TP2 la **code** et la **déploie**,
les TP3/TP4 l'**exploitent** et la **pilotent**.

---

## 1. Concepts Terraform incontournables

| Notion                          | À retenir                                                                                                             |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **IaC**                         | Décrire l'infra dans du **code versionné** au lieu de cliquer. Le code = source de vérité                             |
| **Déclaratif**                  | Je décris **l'état souhaité** ; Terraform calcule les opérations (create/update/delete). Je dis *quoi*, pas *comment* |
| **Provider**                    | Plugin traduisant les ressources en appels d'API (`azurerm` pour Azure, `random` pour suffixe)                        |
| **State** (`terraform.tfstate`) | Lien entre code et ressources réelles. **Peut contenir des secrets → jamais dans Git** (`.gitignore`)                 |
| **`plan` avant `apply`**        | Prévisualise (créé / modifié `~` / détruit) → garde-fou anti-destruction                                              |

**Cycle de commandes :**
`init` (providers) → `fmt` (formate) → `validate` (syntaxe) → `plan -out` (prévisualise) →
`apply` (déploie) → `output` (récupère infos) → `destroy` (nettoie).

Commandes utiles : `terraform state list` (ressources gérées), `terraform output -raw <nom>`.

---

## 2. L'architecture en code

```text
Internet → Public IP + Load Balancer → 2 VM Linux (Nginx via cloud-init), subnet snet-web
         NSG (HTTP ouvert, SSH limité à l'IP admin /32)
         Storage Account privé (versioning Blob)
         Tags + outputs sur l'ensemble
```

Fichiers `.tf` typiques : `versions`, `providers`, `variables`, `locals`, `network`, `security`,
`compute`, `loadbalancer`, `storage`, `outputs`. Valeurs dans `terraform.tfvars` (auto-chargé).

---

## 3. Réseau & sécurité

- **Plage privée `10.20.0.0/16`** (RFC 1918) : non routable depuis Internet, réseau isolé.
- **Subnet dédié `snet-web`** : segmentation ; une future BDD irait dans un subnet privé séparé.
- **NSG** : pare-feu par règles (priorité, port, source, action).
  - `Allow-HTTP` : 80 depuis Internet.
  - `Allow-SSH-Admin` : 22 depuis **mon IP en /32** (réduire la surface d'attaque ; en prod → **Bastion**/VPN).

---

## 4. Compute, LB, stockage

- **`count = 2`** : déployer N instances identiques sans dupliquer le code (paramétrable via `vm_count`).
- **`custom_data` / cloud-init** : script au 1er démarrage (installe Nginx, page « serveur web N »).
- **VM `Standard_B2pts_v2`** : *burstable* low-cost (Arm64 ici, voir §7).
- **Load Balancer** : 1 IP publique en entrée, répartit sur les 2 VM, **sonde de santé** port 80 →
  si une VM tombe, elle est exclue du pool, pas de coupure. **Couche 4 (TCP)** vs Application Gateway
  **couche 7 (HTTP, routage URL, TLS, WAF)**. Preuve : la page **alterne** entre serveur 1 et 2.
- **Storage** : container **privé** (documents métier), `allow_nested_items_to_be_public = false`,
  TLS 1.2. **Versioning** : garde les versions → coût qui s'accumule → ajouter une **règle de cycle de vie**.

---

## 5. Variables, tags, outputs

- **Variables** : éviter le code en dur, réutiliser sur dev/test/prod.
- **Tags** (gouvernance/FinOps) : `project`, `environment`, `owner`, `managed_by=terraform`, `cost_center`.
- **Outputs** : exposer les infos utiles après déploiement (IP du LB, IP des VM, nom du Storage).

---

## 6. Dérive (drift) & state distant

**Drift** = l'infra réelle ne correspond plus au code (modif manuelle dans le portail). Démo :
ajouter un tag via CLI → `terraform plan` détecte l'écart et propose de **réaligner sur le code**.
**Règle d'équipe :** interdire les changements via le portail ; tout passe par une **PR** de code revue.

**State distant** (backend Storage Azure) en équipe : apporte le **verrouillage** (pas deux applies
simultanés) + le **partage** d'un état unique. Le séparer **par environnement** (clé distincte) pour
qu'une opération dev n'impacte pas la prod. Le protéger (chiffré, accès restreint).

---

## 7. Choix région / Arm64 (à défendre)

- **`germanywestcentral`** car France Central en **restriction de capacité** sur Azure for Students.
- **VM Arm64** car les tailles x86 abordables étaient indisponibles (`SkuNotAvailable`) → image Ubuntu
  ARM64 correspondante. **Ça ne change pas la conception, juste le CPU.**
- Terraform a aidé : `location`, `vm_size`, `vm_image` étaient **variabilisés** → changer une valeur suffit.

---

## 8. Multi-environnements (extension C)

- Variabiliser ce qui change (`environment`, `vm_count`, `vm_size`) + 1 fichier de valeurs par env.
- `dev` = `terraform apply` (charge `terraform.tfvars`). `test`/`prod` = `apply -var-file=environments/test.tfvars`.
- Préfixe `projet-environnement` → noms distincts (`rg-shopeasy-dev` / `-prod`) → pas de conflit.
- Seul **dev** appliqué/détruit (coûts + obligation de `destroy`) ; test/prod livrés en code, **state séparé**.

---

## 9. FinOps & nettoyage

VM *burstable* B-series · un seul env déployé · **`terraform destroy` en fin de séance** (vérifier que
le RG n'existe plus → plus de facturation). Pistes : tiering storage, lifecycle sur le versioning,
arrêt planifié des VM.

---

## ✅ Réflexe « démo live »

- Reformater/valider : `terraform fmt` puis `terraform validate`.
- Ajouter un tag dans `locals.tf` → `plan` = mise à jour *in-place* (`~`), pas de recréation.
- Changer le nombre de VM : `vm_count` dans `terraform.tfvars`.
- Sécurité SSH : `security.tf`, règle `Allow-SSH-Admin`, source = `var.allowed_ssh_cidr`.

## ✅ Questions types — avec réponses

1. **C'est quoi l'IaC, et pourquoi ?**
   Décrire l'infra dans du **code versionné** plutôt qu'à la main. Avantages : reproductible,
   traçable (Git/PR), multi-environnements, et `plan` sert de garde-fou avant tout changement.

2. **Déclaratif vs impératif ?**
   En **déclaratif**, on décrit **l'état souhaité** ; Terraform calcule les opérations à faire. En
   impératif, on écrirait la suite d'étapes. On dit *quoi*, pas *comment*.

3. **À quoi sert un provider ?**
   C'est le **plugin** qui traduit les ressources Terraform en appels d'API du fournisseur
   (`azurerm` pour Azure, `random` pour générer un suffixe).

4. **Qu'est-ce que le state, et pourquoi ne pas le versionner dans Git ?**
   Le **state** (`terraform.tfstate`) est le lien entre le code et les ressources réelles. Il peut
   contenir des **secrets** → on l'exclut de Git (`.gitignore`) et, en équipe, on utilise un
   **state distant** chiffré et verrouillé.

5. **Pourquoi `plan` avant `apply` ?**
   Pour **prévisualiser** ce qui sera créé / modifié (`~`) / détruit → éviter une destruction ou une
   recréation accidentelle.

6. **Qu'est-ce que le drift, et comment le gérer ?**
   Le **drift** est l'écart entre l'infra réelle et le code (ex. modif manuelle dans le portail).
   `terraform plan` le détecte et propose de **réaligner sur le code**. Règle d'équipe : tout passe
   par une **PR**, jamais par le portail.

7. **Load Balancer vs Application Gateway ?**
   Le **LB** travaille en **couche 4 (TCP)** : il répartit le trafic et exclut une VM en panne grâce
   à sa **sonde de santé**. L'**Application Gateway** travaille en **couche 7 (HTTP)** : routage par
   URL, terminaison TLS, **WAF**.

8. **À quoi servent les variables et les outputs ?**
   Les **variables** évitent le code en dur et permettent de réutiliser le même code sur dev/test/prod.
   Les **outputs** exposent après déploiement les infos utiles (IP du LB, IP des VM, nom du Storage).

9. **Comment gère-t-on plusieurs environnements ?**
   On variabilise ce qui change (`environment`, `vm_count`, `vm_size`) + un fichier de valeurs par
   env (`-var-file=environments/test.tfvars`), un **préfixe** par env pour éviter les conflits, et un
   **state séparé** par environnement.

10. **Pourquoi `terraform destroy` en fin de séance ?**
    Parce que les ressources cloud **continuent de facturer** tant qu'elles existent. `destroy`
    supprime tout proprement (vérifier que le resource group a disparu).

> **3 phrases qui résument le projet :** (1) j'ai transformé l'archi TP1 en **code Terraform**
> (réseau, sécu, 2 VM derrière un LB, stockage privé, tagué/paramétrable) ; (2) **réellement déployé**
> et vérifié (alternance des serveurs, drift détecté) puis **détruit** ; (3) rendu **réutilisable**
> multi-env, avec évolutions prod documentées (state distant, Bastion, CI/CD).
