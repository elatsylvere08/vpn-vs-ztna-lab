# VPN vs ZTNA — Étude comparative des modèles d'accès distant

![Debian](https://img.shields.io/badge/Debian-13-red?logo=debian)
![OpenVPN](https://img.shields.io/badge/OpenVPN-2.6-orange)
![NIST](https://img.shields.io/badge/NIST-SP%20800--207-blue)
![Status](https://img.shields.io/badge/status-Phase%201%20terminée-green)

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

![Schéma du banc d'essai](docs/schemas/schema-banc-essai.png) 

Environnement entièrement virtualisé sous **VirtualBox**, **3 VM Debian 13** interconnectées sur un **réseau interne isolé** (192.168.50.0/24) :

​```
   Réseau interne VirtualBox « lan-entreprise » (192.168.50.0/24) ┌──────────────┐ ┌───────────────────────┐ ┌───────────────────┐ │ vm-client │───▶│ vm-securite │───▶│ vm-serveur │ │ 192.168.50.10│ │ 192.168.50.30 │ │ 192.168.50.20 │ │ (Debian 13) │ │ Serveur OpenVPN │ │ nginx (intranet) │ └──────────────┘ │ Tunnel 10.8.0.0/24 │ └───────────────────┘ └───────────────────────┘ 
​```

| VM | IP | Rôle |
|---|---|---|
| **vm-client** | 192.168.50.10 | Poste utilisateur distant / point de départ des scénarios offensifs |
| **vm-securite** | 192.168.50.30 | Passerelle de sécurité — terminaison du tunnel OpenVPN (UDP/1194, AES-256-GCM) |
| **vm-serveur** | 192.168.50.20 | Ressource cible — application interne (nginx) |

### Stack de déploiement

- **PKI X.509** générée via Easy-RSA (CA auto-signée, certificats serveur/client, DH 2048)
- **OpenVPN 2.6** en mode TUN, chiffrement AES-256-GCM, hachage SHA-256, TLS-Auth
- **Routage avancé Linux** : `net.ipv4.ip_forward=1`, règle NAT MASQUERADE via iptables
- **Administration SSH** depuis poste hôte Windows (redirection de ports VirtualBox)

## Méthodologie — 6 batteries de mesures

### Volet performance (4 batteries, *baseline* puis via tunnel VPN)

| # | Mesure | Outil | Ce que ça évalue |
|---|---|---|---|
| 1 | **Latence** | `ping -c 100 -i 0.2` | Surcoût d'encapsulation du tunnel |
| 2 | **Débit TCP** | `iperf3 -t 30` | Impact sur les transferts fiables |
| 3 | **Gigue UDP** | `iperf3 -u -b 100M -t 30` | Stabilité pour les flux temps réel (VoIP, etc.) |
| 4 | **Temps de réponse HTTP** | `curl -w` (10 essais) | Ressenti applicatif de bout en bout |

**Résultats chiffrés :**

| Mesure | Baseline (sans VPN) | Via VPN (OpenVPN) | Écart |
|---|---|---|---|
| **Latence moyenne** (ms) | 1,528 | 3,550 | **× 2,3** (+132 %) |
| **Débit TCP** (Mbits/s) | 805 | 8,81 | **÷ 91** (−98,9 %) |
| **Gigue UDP** (ms) | 0,773 | 5,112 | **× 6,6** (+562 %) |
| **Perte UDP @ 100M** (%) | 0 | 92 | catastrophique |
| **Temps HTTP moyen** (ms) | 7,4 | 11,1 | +50 % |

> ⚠️ **Nuance importante** : la dégradation extrême du débit (÷91) tient en partie aux conditions de virtualisation sans accélération matérielle AES-NI. En production sur machines physiques modernes, la dégradation attendue serait plus proche de −30 à −50 %.

### Volet sécurité (2 batteries — `nmap`)

| # | Scénario | Objectif |
|---|---|---|
| 5 | **Cartographie de la surface d'attaque** | Énumération des hôtes/services visibles depuis un point d'entrée public |
| 6 | **Déplacement latéral** | Démonstration : depuis le client connecté, atteinte de ressources non nécessaires à sa mission |

**Synthèse des résultats sécurité :**

**Surface d'attaque de la passerelle VPN**
​```
bash $ sudo nmap -sU -p 1194 192.168.50.30 PORT STATE SERVICE 1194/udp open|filtered openvpn # Service identifiable nativement par nmap $ sudo nmap -sS -p 1-1000 192.168.50.30 PORT STATE SERVICE 22/tcp open ssh # Administration également exposée

​```

→ **2 ports exposés**, dont le service OpenVPN **directement identifiable** par signature. Cela ouvre la voie à la recherche de CVE spécifiques (ex. CVE-2018-7544, CVE-2018-9336) et aux tentatives de brute-force ciblées.

**Déplacement latéral depuis le client compromis**
​```
bash $ sudo nmap -sn 192.168.50.0/24 # Depuis vm-client, tunnel actif Nmap scan report for 192.168.50.10 # Découverte Nmap scan report for 192.168.50.20 # Découverte Nmap scan report for 192.168.50.30 # Découverte $ sudo nmap -sS -p 1-1000 192.168.50.20 PORT STATE SERVICE 22/tcp open ssh # Service atteignable 80/tcp open http # Service atteignable

​```

→ **Cartographie complète du réseau interne** obtenue depuis le client authentifié. Services SSH et HTTP directement atteignables sur la ressource cible. Un attaquant ayant compromis les identifiants VPN d'un utilisateur légitime disposerait donc d'une **visibilité totale** et pourrait tenter des attaques applicatives (injection HTTP, brute-force SSH).

## Résultats & enseignements

- **Le modèle périmétrique élargit la surface d'attaque** : une fois le tunnel monté, le client accède au-delà de ce dont il a strictement besoin, ce qui rend le **déplacement latéral** possible en cas de compromission. Les scans nmap ont confirmé expérimentalement cette limite structurelle.
- **Le VPN introduit un surcoût de performance** mesurable et significatif (latence ×2,3, débit TCP ÷91 dans nos conditions), à mettre en balance avec le niveau de sécurité apporté — surtout au regard du fait que cette sécurité ne bloque pas le déplacement latéral.
- **L'approche ZTNA** répond à ces limites par un accès **ressource par ressource**, au **moindre privilège**, avec **vérification continue** de l'identité et du contexte — conformément au NIST SP 800-207. Le réseau interne n'est jamais exposé (« *network goes dark* »), ce qui empêche par conception la cartographie et le déplacement latéral démontrés ici sur le modèle VPN.

**Positionnement pratique :** le VPN classique conserve une pertinence pour des cas d'usage simples et internes (accès administrateur ponctuel, environnements de faible criticité, contexte où le poste client est parfaitement maîtrisé). Dès lors qu'on parle de télétravail à grande échelle, de BYOD, ou de ressources sensibles exposées à des utilisateurs multiples, l'approche ZTNA devient **structurellement plus adaptée**.

## Perspectives — Phase 2

La deuxième phase du projet, planifiée à la suite, consiste à déployer une architecture ZTNA basée sur **Pomerium** (solution open source) sur la même infrastructure, puis à répéter les 6 batteries de mesures dans des conditions strictement identiques. Cette comparaison directe permettra de valider quantitativement l'apport du modèle Zero Trust sur les deux dimensions (performance et sécurité).

## Stack technique

`Debian 13` · `VirtualBox` · `OpenVPN 2.6` · `Easy-RSA` · `iptables` · `iperf3` · `nmap` · `curl` · `tcpdump` · `NIST SP 800-207` · `CISA ZTMM`

---

## Livrables

- 📄 [Rapport scientifique](docs/rapport.pdf) (~50 pages) — méthodologie ADS complète 
- 🖼️ [Poster de soutenance](docs/poster.pdf) — restitution visuelle des enjeux et résultats
- 🎤 Soutenance orale — 3 min + Q/R + 10 min de questions/réponses avec vulgarisation adaptée à un public non spécialiste

## Note

Projet mené dans un cadre pédagogique, incluant une **restitution vulgarisée** des enjeux du Zero Trust auprès d'un public non spécialiste.

*Auteur : Sylvère Elat — Étudiant ingénieur Systèmes, Réseaux & Cybersécurité (CESI La Rochelle)*
