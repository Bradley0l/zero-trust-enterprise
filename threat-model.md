# Threat Model — STRIDE

Périmètre : architecture `VNET-ZERO-TRUST` (DMZ / INTERNAL / Azure Firewall) et couche identité Entra ID de **ZAMI Secure Technologies**.

Légende : ✅ implémenté dans les labs · 🧩 conçu (non déployé — licence ou coût)

| Catégorie | Cible | Menace | Contrôle | Statut | Risque résiduel |
| :--- | :--- | :--- | :--- | :---: | :---: |
| **S**poofing | Comptes utilisateurs | Vol d'identifiants, phishing | MFA via Security Defaults (Microsoft Authenticator) | ✅ | Moyen → Faible |
| **S**poofing | Comptes privilégiés | Usurpation d'un administrateur | MFA résistant au phishing + PIM (JIT) | 🧩 | Moyen |
| **T**ampering | Flux DMZ → données | Modification de requêtes, MitM | Seul TCP 1433 DMZ → INTERNAL autorisé par le pare-feu ; TLS sur les flux applicatifs | ✅ / 🧩 | Faible |
| **R**epudiation | Actions d'administration | Un admin nie une modification | Journal d'activité Azure + audit logs Entra ID ; centralisation dans Log Analytics | ✅ / 🧩 | Moyen |
| **I**nformation disclosure | Subnet INTERNAL | Exfiltration vers Internet | UDR `0.0.0.0/0` → pare-feu ; sortie limitée à `*.microsoft.com`, `github.com` en HTTPS | ✅ | Faible |
| **I**nformation disclosure | Ressources PROD | Lecture par un développeur | RBAC : `SEC-Developers` sans accès à `RG-PROD` (testé avec Alice) | ✅ | Faible |
| **D**enial of Service | Tier Web / API | Saturation volumétrique | Azure Firewall en frontal ; DDoS Protection à ajouter | 🧩 | Moyen |
| **E**levation of privilege | Abonnement Azure | Un utilisateur devient Owner | Rôles attribués par groupe et par Resource Group, aucun Owner hors admin du tenant ; PIM prévu | ✅ / 🧩 | Moyen → Faible |
| **E**levation of privilege | Mouvement latéral | Un serveur DMZ compromis attaque le réseau interne | Segmentation par subnets + NSG + inspection Est-Ouest par le pare-feu | ✅ | Faible |

## Raisonnement Threat → Risk → Control → Residual

Exemple pour le mouvement latéral :

```
Threat   : un serveur Web de la DMZ est compromis
   ↓
Risk     : l'attaquant pivote vers les données du subnet INTERNAL
   ↓
Control  : UDR forçant le passage par Azure Firewall
           + règle unique Allow-Web-to-DB (TCP 1433) + deny par défaut
           + NSG-INTERNAL
   ↓
Residual : l'attaquant ne peut atteindre que le port SQL
           → à couvrir par l'authentification forte de la base
             et la surveillance (Sentinel)
```
