# 🛡️ Zero Trust Enterprise — Architecture réseau sécurisée sur Azure

![Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?logo=microsoftazure&logoColor=white)
![Azure Firewall](https://img.shields.io/badge/Azure_Firewall-Standard-red)
![Zero Trust](https://img.shields.io/badge/Zero_Trust-Architecture-success)
![Status](https://img.shields.io/badge/Lab-Termin%C3%A9-brightgreen)

> **Lab 1** de mon parcours *Cloud Security Architect*.
> Objectif : concevoir et déployer l'architecture sécurisée d'une entreprise fictive de **100 utilisateurs** — **ZAMI Secure Technologies** — en appliquant les trois principes Zero Trust :
> **Verify explicitly → Least privilege → Assume breach.**

La couche **identité** (Entra ID, groupes, RBAC, MFA) est documentée dans le lab compagnon 👉 [azure-entra-id-iam](https://github.com/Bradley0l/azure-entra-id-iam).
Ce dépôt couvre la couche **réseau** : segmentation, routage forcé et inspection par Azure Firewall.

---

## 📐 Architecture

![Architecture Zero Trust](architecture.png)

| Composant | Valeur | Rôle |
| :--- | :--- | :--- |
| VNet | `VNET-ZERO-TRUST` — `10.0.0.0/16` | Réseau de l'entreprise |
| Subnet DMZ | `Subnet-DMZ` — `10.0.1.0/24` + `NSG-DMZ` | Tier Web / API exposé |
| Subnet interne | `Subnet-INTERNAL` — `10.0.2.0/24` + `NSG-INTERNAL` | Postes, applications, données |
| Subnet pare-feu | `AzureFirewallSubnet` — `10.0.3.0/26` | Héberge Azure Firewall |
| Table de routage | `UDR-FIREWALL-REDIRECT` | `0.0.0.0/0` → `10.0.3.4` (Virtual Appliance) |
| Pare-feu | `FW-ZERO-TRUST` (Standard) — IP privée `10.0.3.4` | Inspection Nord-Sud et Est-Ouest |
| Stratégie | `POLICY-ZERO-TRUST` (Standard) | Règles réseau + règles d'application |
| IP publique | `PIP-FW-ZERO-TRUST` | Point d'entrée du pare-feu |

**Idée clé :** aucun subnet ne parle directement à un autre ni à Internet. Grâce à l'UDR, **tout le trafic sortant de la DMZ et du réseau interne passe obligatoirement par le pare-feu**, qui n'autorise que les flux explicitement déclarés (*deny by default*).

---

## 🧱 Contrôles Zero Trust couverts

| Contrôle | Mise en œuvre | Où |
| :--- | :--- | :--- |
| Identity verification | Entra ID, comptes nominatifs | [Lab 2](https://github.com/Bradley0l/azure-entra-id-iam) |
| MFA | Security Defaults (MFA imposé à l'inscription) | [Lab 2](https://github.com/Bradley0l/azure-entra-id-iam) |
| Least Privilege / RBAC | Matrice de rôles par groupe sur `RG-DEV` / `RG-PROD` / `RG-SECURITY` | [Lab 2](https://github.com/Bradley0l/azure-entra-id-iam) |
| Network segmentation | VNet découpé en DMZ / INTERNAL / Firewall + NSG par subnet | ce dépôt |
| Traffic inspection | UDR + Azure Firewall + Firewall Policy | ce dépôt |
| Conditional Access, Device trust, PIM | Conçus (nécessitent Entra ID P1/P2 + Intune) | [security-controls.md](security-controls.md) |
| Logging / Monitoring | Conçus (Log Analytics + Sentinel) | [security-controls.md](security-controls.md) |

📄 Documents d'architecte :
- [`threat-model.md`](threat-model.md) — analyse STRIDE
- [`security-controls.md`](security-controls.md) — détail des contrôles techniques
- [`decisions.md`](decisions.md) — Architecture Decision Records (ADR)

---

## 🛠️ Réalisation pas à pas

### 1. Réseau virtuel `VNET-ZERO-TRUST`

Création du VNet dans `RG-DEV` (France Central) avec l'espace d'adressage `10.0.0.0/16`.

| Informations de base | Espace d'adressage |
| :---: | :---: |
| ![VNet basics](images/01-vnet-basics.png) | ![Address space](images/02-vnet-address-space.png) |

![VNet déployé](images/03-vnet-deployed.png)

### 2. Segmentation en sous-réseaux

Trois sous-réseaux dédiés. `AzureFirewallSubnet` est un nom imposé par Azure et doit être créé avec l'objectif *Azure Firewall*. La plage `10.0.0.0/24` étant occupée par le subnet `default`, le subnet pare-feu a été placé en `10.0.3.0/26`.

![Sous-réseaux](images/04-subnets.png)

### 3. Network Security Groups

Un NSG par tier : `NSG-DMZ` et `NSG-INTERNAL`, puis association à leur subnet respectif.

| Création NSG-DMZ | Validation |
| :---: | :---: |
| ![NSG DMZ](images/05-nsg-dmz-create.png) | ![Validation](images/06-nsg-dmz-validation.png) |

| Création NSG-INTERNAL | NSG créés |
| :---: | :---: |
| ![NSG INTERNAL](images/07-nsg-internal-create.png) | ![Liste NSG](images/08-nsg-list.png) |

| NSG-DMZ → Subnet-DMZ | NSG-INTERNAL → Subnet-INTERNAL |
| :---: | :---: |
| ![Association DMZ](images/09-nsg-dmz-association.png) | ![Association INTERNAL](images/10-nsg-internal-association.png) |

### 4. Routage forcé vers le pare-feu (UDR)

Table de routage `UDR-FIREWALL-REDIRECT` (propagation des routes de passerelle **désactivée**) avec une route par défaut `To-Azure-Firewall` : `0.0.0.0/0` → **Appliance virtuelle** `10.0.3.4`.

| Création de la table | Route vers le pare-feu |
| :---: | :---: |
| ![UDR](images/11-route-table-create.png) | ![Route](images/12-route-to-firewall.png) |

![Routes](images/13-route-list.png)

Association de la table aux deux subnets applicatifs :

| Subnet-DMZ | Subnet-INTERNAL |
| :---: | :---: |
| ![UDR DMZ](images/14-udr-subnet-dmz.png) | ![UDR INTERNAL](images/15-udr-subnet-internal.png) |

![UDR associée](images/16-udr-subnets.png)

### 5. Stratégie de pare-feu `POLICY-ZERO-TRUST`

| Création de la stratégie (Standard) | Déploiement |
| :---: | :---: |
| ![Policy](images/17-firewall-policy-create.png) | ![Policy deploy](images/18-firewall-policy-deployment.png) |

### 6. Déploiement d'Azure Firewall `FW-ZERO-TRUST`

Pare-feu **Standard**, géré par `POLICY-ZERO-TRUST`, rattaché au VNet existant avec une nouvelle IP publique `PIP-FW-ZERO-TRUST`. La carte réseau de gestion (forced tunneling) a été laissée désactivée : elle exigerait un `AzureFirewallManagementSubnet` supplémentaire.

| Informations de base | Stratégie et réseau |
| :---: | :---: |
| ![FW basics](images/19-firewall-basics.png) | ![FW policy](images/20-firewall-policy-vnet.png) |

| Vérification | Déploiement |
| :---: | :---: |
| ![FW review](images/21-firewall-review.png) | ![FW deploy](images/22-firewall-deployment.png) |

### 7. Règles de filtrage

**Règle réseau** — `RC-Net-Allow-DMZ-to-Internal` (priorité 200, Allow)

| Nom | Source | Protocole | Port | Destination |
| :--- | :--- | :--- | :--- | :--- |
| `Allow-Web-to-DB` | `10.0.1.0/24` (DMZ) | TCP | 1433 (SQL) | `10.0.2.0/24` (INTERNAL) |

| Création | Résultat |
| :---: | :---: |
| ![Network rule](images/23-network-rule-create.png) | ![Network rules](images/24-network-rules.png) |

**Règle d'application** — `RC-App-Allow-Outbound` (priorité 300, Allow)

| Nom | Source | Protocole | Destination (FQDN) |
| :--- | :--- | :--- | :--- |
| `Allow-HTTPS-Outbound` | `10.0.2.0/24` (INTERNAL) | HTTPS:443 | `*.microsoft.com`, `github.com` |

| Création | Résultat |
| :---: | :---: |
| ![App rule](images/25-application-rule-create.png) | ![App rules](images/26-application-rules.png) |

Tout autre flux est **refusé par défaut** par le pare-feu.

---

## ✅ Validation

**Pare-feu opérationnel** — état d'approvisionnement `Succeeded`, IP privée `10.0.3.4`, stratégie `POLICY-ZERO-TRUST` associée :

![Firewall Succeeded](images/27-firewall-succeeded.png)

**Chaîne de routage cohérente** — la route `0.0.0.0/0` pointe vers l'IP privée réelle du pare-feu et la table est associée à `Subnet-DMZ` et `Subnet-INTERNAL` :

![UDR overview](images/28-udr-overview.png)

**Network Watcher** — activé dans la région. L'outil *Next hop* nécessite une VM / NIC existante dans le subnet source ; aucune VM n'ayant été déployée (choix de coût), la validation du saut suivant s'appuie sur la configuration UDR ci-dessus.

![Network Watcher](images/29-network-watcher.png)

---

## 💰 Maîtrise des coûts

Azure Firewall Standard est facturé environ **1,25 $/heure** même sans trafic. Stratégie appliquée :

1. Construire gratuitement toute la structure (VNet, subnets, NSG, UDR, Firewall Policy).
2. Déployer le pare-feu sur l'abonnement **Azure for Students** uniquement le temps des tests et des captures.
3. **Supprimer `FW-ZERO-TRUST` immédiatement après**, en conservant les ressources gratuites.

---

## 🧠 Ce que j'ai appris / difficultés rencontrées

- `AzureFirewallSubnet` exige un nom exact, l'objectif *Azure Firewall* et au minimum un `/26`.
- Le subnet `default` créé automatiquement occupe `10.0.0.0/24` : il faut le supprimer ou choisir une autre plage.
- L'option *Management NIC* du pare-feu impose un `AzureFirewallManagementSubnet` dédié.
- Annuler un abonnement Azure désactive toutes ses ressources, mais **pas** les identités Entra ID (rattachées au tenant). J'ai dû reconstruire la couche réseau sur l'abonnement Azure for Students.
- Next hop (Network Watcher) ne fonctionne qu'avec une NIC réelle comme source.

## 🚀 Prochaines étapes

- Déployer deux VM B1s (DMZ / INTERNAL) pour tester les flux autorisés et refusés de bout en bout.
- Envoyer les logs du pare-feu et des NSG (flow logs) vers Log Analytics, puis Microsoft Sentinel.
- Ajouter Conditional Access + Intune (Entra ID P1/P2) pour le *device trust*.
- Décrire toute l'infrastructure en Terraform (voir [terraform-security](https://github.com/Bradley0l/terraform-security)).

---

**Auteur :** André Zami — étudiant en cybersécurité, orienté Cloud Security Architecture.
