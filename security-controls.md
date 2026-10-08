# Security Controls

Légende : ✅ implémenté · 🧩 conçu (non déployé — licence Entra ID P1/P2 ou coût)

## 1. Segmentation réseau ✅

| Subnet | Plage | NSG | Table de routage |
| :--- | :--- | :--- | :--- |
| `Subnet-DMZ` | `10.0.1.0/24` | `NSG-DMZ` | `UDR-FIREWALL-REDIRECT` |
| `Subnet-INTERNAL` | `10.0.2.0/24` | `NSG-INTERNAL` | `UDR-FIREWALL-REDIRECT` |
| `AzureFirewallSubnet` | `10.0.3.0/26` | — (géré par Azure) | — |

## 2. Routage forcé (UDR) ✅

| Route | Préfixe | Next hop | Adresse |
| :--- | :--- | :--- | :--- |
| `To-Azure-Firewall` | `0.0.0.0/0` | Virtual appliance | `10.0.3.4` |

Propagation des routes de passerelle : **désactivée**.

## 3. Azure Firewall — `POLICY-ZERO-TRUST` ✅

**Règles réseau** — groupe `DefaultNetworkRuleCollectionGroup`

| Collection | Priorité | Action | Règle | Source | Port / Proto | Destination |
| :--- | :---: | :---: | :--- | :--- | :--- | :--- |
| `RC-Net-Allow-DMZ-to-Internal` | 200 | Allow | `Allow-Web-to-DB` | `10.0.1.0/24` | TCP 1433 | `10.0.2.0/24` |

**Règles d'application** — groupe `DefaultApplicationRuleCollectionGroup`

| Collection | Priorité | Action | Règle | Source | Protocole | FQDN |
| :--- | :---: | :---: | :--- | :--- | :--- | :--- |
| `RC-App-Allow-Outbound` | 300 | Allow | `Allow-HTTPS-Outbound` | `10.0.2.0/24` | HTTPS:443 | `*.microsoft.com`, `github.com` |

Tout flux non listé est refusé par défaut.

## 4. Identité et accès ✅ (détails : [azure-entra-id-iam](https://github.com/Bradley0l/azure-entra-id-iam))

| Groupe Entra ID | RG-DEV | RG-PROD | RG-SECURITY |
| :--- | :--- | :--- | :--- |
| `SEC-Developers` | Contributor | ❌ | ❌ |
| `SEC-IT-Admins` | Contributor | Contributor | Reader |
| `SEC-Security` | Reader | Reader | Contributor |
| `SEC-HR` | ❌ | ❌ | ❌ |
| `SEC-Employees` | ❌ | ❌ | ❌ |

- MFA : **Security Defaults activés** (inscription Microsoft Authenticator obligatoire, protocoles hérités bloqués).

## 5. Conditional Access & Device Trust 🧩

| Politique | Utilisateurs | Applications | Conditions | Contrôle |
| :--- | :--- | :--- | :--- | :--- |
| `CAP-01-MFA-Compliant-Device` | Tous | Toutes les apps cloud | — | MFA **et** appareil conforme (Intune) |
| `CAP-02-Block-Risky-SignIn` | Tous | Toutes | Risque de connexion moyen / élevé | Bloquer |
| `CAP-03-Admins-Phishing-Resistant` | Rôles privilégiés | Azure Management | — | MFA résistant au phishing (FIDO2) |

## 6. Comptes privilégiés — PIM 🧩

- Aucune attribution permanente Owner / Global Administrator.
- Rôles *Eligible* activés en Just-In-Time : MFA + justification + ticket, durée maximale 2 h, approbation par l'équipe sécurité.

## 7. Journalisation et supervision 🧩

| Source | Destination | Usage |
| :--- | :--- | :--- |
| Azure Firewall (règles réseau / application) | Log Analytics | Traçabilité des flux autorisés / refusés |
| NSG flow logs (Network Watcher) | Storage + Traffic Analytics | Analyse Est-Ouest |
| Entra ID sign-in & audit logs | Log Analytics / Sentinel | Détection brute force, impossible travel |
| Azure Activity Log | Log Analytics | Non-répudiation des actions d'administration |
