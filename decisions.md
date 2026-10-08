# Architecture Decision Records (ADR)

## ADR-01 — Pare-feu central et routage forcé (hub filtrant)

- **Statut :** Accepté
- **Contexte :** les flux entre DMZ, réseau interne et Internet doivent être inspectés pour 100 utilisateurs.
- **Décision :** déployer Azure Firewall dans `AzureFirewallSubnet` et associer une UDR `0.0.0.0/0 → 10.0.3.4` aux subnets DMZ et INTERNAL.
- **Conséquences :** visibilité et journalisation centralisées ; léger surcoût de latence ; la table de routage doit être maintenue à chaque nouveau subnet.

## ADR-02 — Firewall Policy plutôt que règles classiques

- **Statut :** Accepté
- **Contexte :** les règles doivent être versionnables et réutilisables sur plusieurs pare-feux.
- **Décision :** gérer `FW-ZERO-TRUST` via la stratégie `POLICY-ZERO-TRUST` (Standard), avec des collections séparées *réseau* et *application*.
- **Conséquences :** la stratégie existe indépendamment du pare-feu (gratuite) et survit à sa suppression.

## ADR-03 — Deny by default, autorisations minimales

- **Statut :** Accepté
- **Décision :** n'autoriser que `DMZ → INTERNAL` en TCP 1433 et la sortie HTTPS de INTERNAL vers une liste de FQDN.
- **Conséquences :** toute nouvelle dépendance applicative exige une règle explicite et documentée.

## ADR-04 — Contrôle d'accès par groupes, au niveau Resource Group

- **Statut :** Accepté
- **Décision :** attribuer les rôles Azure RBAC à des groupes Entra ID (`SEC-*`), jamais à des utilisateurs, avec un scope limité à `RG-DEV`, `RG-PROD` ou `RG-SECURITY`.
- **Conséquences :** l'arrivée ou le départ d'un collaborateur se gère uniquement par l'appartenance aux groupes.

## ADR-05 — Security Defaults en attendant Conditional Access

- **Statut :** Accepté (transitoire)
- **Contexte :** Conditional Access, PIM et Intune exigent Entra ID P1/P2.
- **Décision :** activer les Security Defaults (MFA pour tous, blocage des protocoles hérités) sur le tenant Free.
- **Conséquences :** pas de politique contextuelle (appareil, localisation) ; migration vers Conditional Access dès qu'une licence P1 est disponible.

## ADR-06 — Déploiement éphémère du pare-feu

- **Statut :** Accepté
- **Contexte :** Azure Firewall Standard coûte environ 1,25 $/h ; budget limité au crédit Azure for Students.
- **Décision :** déployer le pare-feu uniquement le temps de la validation, puis le supprimer en conservant VNet, NSG, UDR et Firewall Policy.
- **Conséquences :** l'environnement doit pouvoir être redéployé rapidement ; à terme, via Terraform.
