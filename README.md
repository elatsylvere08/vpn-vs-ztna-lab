# vpn-vs-ztna-lab
# VPN vs ZTNA — Étude comparative des modèles d'accès distant

> Banc d'essai expérimental démontrant les limites du modèle périmétrique (VPN) face à une approche **Zero Trust Network Access**, évalué selon le **NIST SP 800-207** et le **CISA Zero Trust Maturity Model**.

**Projet ADS — CESI École d'Ingénieurs (La Rochelle) · Cycle ingénieur M1 · 2026**

---

## Contexte

Le modèle périmétrique traditionnel repose sur un principe simple : *ce qui est à l'intérieur du réseau est de confiance*. Une fois le tunnel VPN établi, l'utilisateur obtient un accès large au réseau interne. Ce projet met cette hypothèse à l'épreuve sur un banc d'essai dédié, puis la confronte au modèle **Zero Trust** (« ne jamais faire confiance, toujours vérifier », accès au moindre privilège, ressource par ressource).

## Objectifs

- Mesurer l'impact d'un tunnel VPN sur les performances réseau (référence *baseline* vs. VPN).
- Caractériser la **surface d'attaque** exposée et démontrer le **déplacement latéral** possible une fois un poste client compromis.
- Positionner les résultats face aux référentiels **NIST SP 800-207** (Zero Trust Architecture) et **CISA Zero Trust Maturity Model**.
- Restituer les enjeux à un public non spécialiste (volet pédagogique).

## Cadre de référence

| Référentiel | Apport dans l'étude |
|---|---|
| **NIST SP 800-207** | Principes d'architecture Zero Trust, composants (PEP/PDP), vérification continue |
| **CISA Zero Trust Maturity Model** | Grille de maturité pour situer le modèle périmétrique vs. ZTNA |

## Architecture du banc d'essai

Environnement entièrement virtualisé sous **VirtualBox**, **3 VM Debian 13** interconnectées sur un **réseau interne isolé** :

```
        Réseau interne VirtualBox (isolé)
   ┌──────────┐    ┌───────────────────┐    ┌────────────────┐
   │  CLIENT  │───▶│ PASSERELLE DE SÉC. │───▶│ SERVEUR APPLIC.│
   │ (Debian) │    │  (VPN / contrôle)  │    │   (Debian)     │
   └──────────┘    └───────────────────┘    └────────────────┘
```

| VM | Rôle |
|---|---|
| **Client** | Poste utilisateur / point de départ des scénarios offensifs |
| **Passerelle de sécurité** | Terminaison du tunnel, point de contrôle d'accès |
| **Serveur applicatif** | Ressource cible (service exposé) |

## Méthodologie — 6 batteries de mesures

### Volet performance (4 batteries, *baseline* puis via tunnel VPN)

| # | Mesure | Ce que ça évalue |
|---|---|---|
| 1 | **Latence** | Surcoût d'encapsulation du tunnel |
| 2 | **Débit TCP** | Impact sur les transferts fiables |
| 3 | **Gigue UDP** | Stabilité pour les flux temps réel (VoIP, etc.) |
| 4 | **Temps de réponse HTTP** | Ressenti applicatif de bout en bout |

<!-- Outils utilisés — à confirmer/ajuster selon ce que tu as réellement employé :
     ping (latence), iperf3 (débit TCP / gigue UDP), curl -w (temps HTTP) -->

> ⚠️ **À COMPLÉTER** — résultats chiffrés (exemple de tableau à remplir avec tes vraies valeurs) :
>
> | Mesure | Baseline | Via VPN | Écart |
> |---|---|---|---|
> | Latence (ms) | … | … | … |
> | Débit TCP (Mbit/s) | … | … | … |
> | Gigue UDP (ms) | … | … | … |
> | Temps HTTP (ms) | … | … | … |

### Volet sécurité (2 batteries — `nmap`)

| # | Scénario | Objectif |
|---|---|---|
| 5 | **Cartographie de la surface d'attaque** | Énumération des hôtes/services visibles une fois connecté |
| 6 | **Déplacement latéral** | Démonstration : depuis le client connecté, atteinte de ressources non nécessaires à sa mission |

> ⚠️ **À COMPLÉTER** — synthèse de ce que `nmap` a révélé (hôtes/ports atteignables depuis le client, chemin de latéralisation observé).

## Résultats & enseignements

- **Le modèle périmétrique élargit la surface d'attaque** : une fois le tunnel monté, le client accède au-delà de ce dont il a strictement besoin, ce qui rend le **déplacement latéral** possible en cas de compromission.
- **Le VPN introduit un surcoût de performance** mesurable (voir tableau à compléter), à mettre en balance avec le niveau de sécurité apporté.
- **L'approche ZTNA** répond à ces limites par un accès **ressource par ressource**, au **moindre privilège**, avec **vérification continue** de l'identité et du contexte — conformément au NIST SP 800-207.

<!-- Ajoute ici, en 2-3 phrases, ta conclusion personnelle : dans quel cas VPN reste acceptable, à partir de quand ZTNA s'impose. -->

## Stack technique

`Debian 13` · `VirtualBox` · `VPN` · `nmap` · `NIST SP 800-207` · `CISA ZTMM`

---

## Note

Projet mené dans un cadre pédagogique, incluant une **restitution vulgarisée** des enjeux du Zero Trust auprès d'un public non spécialiste.

*Auteur : Sylvère Elat — Étudiant ingénieur Systèmes, Réseaux & Cybersécurité (CESI La Rochelle)*
