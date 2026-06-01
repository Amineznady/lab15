# LAB-15 — Inspection du Trafic TLS/HTTPS Android avec Frida et Burp Suite

**Auteur :** Mohamed Amine Znady  
**Filière :** Cycle Ingénieur — CIR (Systèmes d'Information Distribués), EMSI Marrakech  
**Module :** Sécurité Mobile & Analyse de Trafic Android  

---

## Objectif du lab

Intercepter le trafic réseau d'une application Android bancaire via **Burp Suite** comme proxy, et utiliser **Frida** pour injecter un bypass SSL Pinning si l'application refuse les certificats tiers.

L'application cible est **InsecureBankv2**, une fausse application bancaire Android conçue pour être analysée dans un cadre pédagogique.

> Lab réalisé dans un environnement de test contrôlé à des fins pédagogiques.

---

## Environnement

| Élément | Détail |
|---|---|
| **Système** | Windows + PowerShell |
| **Émulateur** | Android Studio Emulator 5554 |
| **Frida** | 17.8.0 |
| **Proxy** | Burp Suite Community Edition |
| **Backend** | AndroLabServer (Python 2.7) |
| **Package cible** | `com.android.insecurebankv2` |

---

## Structure du projet

```
LAB15-SSL-Pinning-Frida/
│
├── README.md
├── hello.js                    ← test d'injection Frida
└── sslpin_bypass_universal.js  ← bypass SSL pinning
```

---

## Étape 1 — Vérifier que Frida détecte l'application

```bash
frida-ps -Uai
```

InsecureBankv2 doit apparaître dans la liste avec son PID :

```
PID   Name            Identifier
7288  InsecureBankv2  com.android.insecurebankv2
```

---

## Étape 2 — Test d'injection rapide

Avant les scripts complexes, on valide que Frida peut s'injecter dans le processus :

```bash
frida -U -f com.android.insecurebankv2 -l hello.js
```

Sortie attendue :
```
Connected to Android Emulator 5554
Spawned `com.android.insecurebankv2`. Resuming main thread!
[+] Script injecté: Java.perform OK
```

---

## Étape 3 — Configurer Burp Suite pour écouter le trafic Android

Dans Burp Suite :

```
Proxy > Proxy settings > Proxy listeners

Bind to port    : 8080
Bind to address : All interfaces
```

> **All interfaces** est indispensable — cela permet à Burp d'écouter sur l'adresse IP locale que l'émulateur Android utilise pour router son trafic.

---

## Étape 4 — Pointer l'émulateur vers Burp

Dans l'émulateur Android :

```
Settings > Network & Internet > Wi-Fi > AndroidWifi > Edit

Proxy          : Manual
Proxy hostname : 192.168.11.130   ← IP de la machine Windows
Proxy port     : 8080
```

---

## Étape 5 — Installer le certificat CA de Burp sur Android

Sans cette étape, Android rejette les connexions HTTPS interceptées par Burp.

Depuis le navigateur de l'émulateur, ouvrir :
```
http://burp
```

Télécharger `cacert.der`, puis l'installer :

```
Settings > Security > Encryption & credentials > Install a certificate > CA certificate
```

Confirmation attendue :
```
CA certificate installed
```

---

## Étape 6 — Lancer le serveur backend

InsecureBankv2 requiert le serveur **AndroLabServer** — attention, il tourne sous **Python 2.7 uniquement** (incompatible Python 3) :

```bash
cd D:\Desktop\Android-InsecureBankv2\AndroLabServer
py -2 app.py
```

Sortie attendue :
```
The server is hosted on port: 8888
```

Si les dépendances manquent :
```bash
py -2 -m pip install -r requirements.txt
```

> Accéder à `http://192.168.11.130:8888` retourne `Not Found` — comportement normal, le serveur répond uniquement aux routes applicatives comme `/login`.

---

## Étape 7 — Connecter l'application au serveur

Dans InsecureBankv2, renseigner l'adresse du backend :

```
Server IP   : 192.168.11.130
Server Port : 8888
```

Connexion avec les identifiants de test → l'écran principal s'affiche (Transfer, View Statement, Change Password).

---

## Étape 8 — Observer les requêtes dans Burp Suite

Dans `Proxy > HTTP history`, les requêtes de l'application apparaissent. La plus intéressante :

```http
POST /login HTTP/1.1
Host: 192.168.11.130:8888
Content-Type: application/x-www-form-urlencoded
User-Agent: Apache-HttpClient/UNAVAILABLE (java 1.4)

username=jack&password=********
```

> Les identifiants transitent **en clair** dans le corps de la requête HTTP — aucun chiffrement, aucun encodage.

---

## Étape 9 — Bypass SSL Pinning avec Frida

Pour les applications implémentant du SSL Pinning (vérification stricte du certificat serveur), on injecte un script de bypass :

```bash
frida -U -f com.android.insecurebankv2 -l sslpin_bypass_universal.js
```

Sortie attendue :
```
[+] Universal SSL Pinning Bypass chargé
[+] Cible : com.android.insecurebankv2
[+] SSL bypass: SSLContext.init patché
[+] SSL bypass: TrustManagerImpl.verifyChain patché
[-] OkHttp CertificatePinner non trouvé
[+] SSL bypass: WebViewClient.onReceivedSslError patché
[+] Hooks SSL installés
```

> Le message `OkHttp CertificatePinner non trouvé` n'est pas une erreur — InsecureBankv2 n'utilise pas OkHttp pour le pinning. Les autres hooks sont actifs.

---

## Résumé du flux d'interception

```
InsecureBankv2 (émulateur Android 5554)
   │
   ├─ [Étape 1-2]  Frida détecte et s'injecte dans le processus
   ├─ [Étape 3-4]  Burp écoute sur :8080 / proxy Android configuré
   ├─ [Étape 5]    Certificat CA Burp installé sur Android
   ├─ [Étape 6-7]  Backend AndroLabServer lancé sur :8888
   ├─ [Étape 8]    Trafic HTTP intercepté → credentials en clair dans Burp
   └─ [Étape 9]    SSL Pinning bypassé via sslpin_bypass_universal.js ✓
```

---

*Mohamed Amine Znady — EMSI Marrakech, 2024-2025*
