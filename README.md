# Guide Complet — Bypass de Root Detection avec Objection

**Auteur :** Maryam IKHERAZEN  
**Environnement :** Windows 11 — Genymotion Android 9.0 (Pie) x86 — Frida 17.10.1 — Objection 1.12.5

---

## Sommaire

1. [Vue d'ensemble — C'est quoi Objection ?](#1-vue-densemble--cest-quoi-objection-)
2. [Prérequis et vérifications initiales](#2-prérequis-et-vérifications-initiales)
3. [Étape 1 — Installer Objection](#3-étape-1--installer-objection)
4. [Étape 2 — Préparer l'appareil et démarrer frida-server](#4-étape-2--préparer-lappareil-et-démarrer-frida-server)
5. [Étape 3 — Démarrer Objection sur l'app cible](#5-étape-3--démarrer-objection-sur-lapp-cible)
6. [Étape 4 — État initial et reconnaissance](#6-étape-4--état-initial-et-reconnaissance)
7. [Étape 5 — Exécution du bypass et validation](#7-étape-5--exécution-du-bypass-et-validation)
8. [Étape 6 — Ce que fait android root disable](#8-étape-6--ce-que-fait-android-root-disable)
9. [Étape 7 — Bonus natif avec frida-trace](#9-étape-7--bonus-natif-avec-frida-trace)
10. [Concepts clés à retenir](#10-concepts-clés-à-retenir)
11. [Comparatif Frida / Medusa / Objection](#11-comparatif-frida--medusa--objection)
---

## 1. Vue d'ensemble — C'est quoi Objection ?

### Le contexte

Dans les labs précédents, on a utilisé **Frida** pour écrire des scripts JavaScript manuellement (bypass_root.js, bypass_native.js) et **Medusa** pour charger des modules prêts à l'emploi au format `.med`. Les deux nécessitent soit d'écrire du code, soit de connaître la structure des modules.

**Objection** adopte une approche différente : c'est une **CLI interactive** construite au-dessus de Frida qui expose des commandes de haut niveau directement dans un shell. Pas besoin d'écrire de JavaScript — on tape `android root disable` et Objection installe lui-même les hooks Frida nécessaires.

```
[App Android]  ←→  [frida-server]  ←→  [Objection CLI]
                                          ↑
                                   Console interactive
                                   android root disable
                                   android sslpinning disable
                                   android hooking ...
```

---

## 2. Prérequis et vérifications initiales

### Ce dont on a besoin

- PC Windows/macOS/Linux avec droits admin/sudo
- Python 3.8+ et `pip`
- ADB (Android Platform Tools)
- Genymotion Android 9.0 x86 avec `frida-server` déployé (voir Lab 1)
- L'app cible : **RootBeer Sample** — `com.scottyab.rootbeer.sample`

### Vérifications rapides

```powershell
python --version
pip --version
adb devices
frida --version
```

**Ce qu'on attend :**

- `adb devices` → l'émulateur Genymotion apparaît comme `device`
- `frida --version` → `17.10.1`

<img width="876" height="180" alt="image" src="https://github.com/user-attachments/assets/1fa0b42c-5ed3-45de-b1fc-364e457ffb63" />

---

## 3. Étape 1 — Installer Objection

### Installation

```powershell
# Méthode pip classique
pip install --upgrade objection
```

### Vérification

```powershell
objection version
# → objection: 1.12.5
```
<img width="395" height="41" alt="image" src="https://github.com/user-attachments/assets/dc42c912-2811-49c8-991d-26d68507aa69" />
---

## 4. Étape 2 — Préparer l'appareil et démarrer frida-server

### Procédure (identique aux labs précédents)

```powershell
# 1. Vérifier l'architecture du device
adb shell getprop ro.product.cpu.abi
# → x86

# 2. Lancer frida-server (déjà déployé depuis Lab 1)
adb shell "/data/local/tmp/frida-server -l 0.0.0.0 &"

# 3. Vérifier la visibilité des apps
frida-ps -Uai
```

**Pourquoi forwarder 27043 ?** Objection utilise ce port en plus de 27042 pour ses communications internes avec l'agent injecté.

### Résultat de frida-ps -Uai

<img width="1297" height="615" alt="image" src="https://github.com/user-attachments/assets/b86db55d-e073-466c-85c4-b7645f4b8901" />

RootBeer Sample est visible (PID `-` = app non encore lancée).
Après lancement de RootBeer, elle reçoit un PID:
<img width="1075" height="65" alt="image" src="https://github.com/user-attachments/assets/636d7338-9dba-4afe-8aa4-41b482aa7064" />

---

## 5. Étape 3 — Démarrer Objection sur l'app cible

### Deux stratégies disponibles

**Spawn** — démarrer l'app sous Objection (hooks appliqués avant le code de l'app) :

```powershell
objection -n com.scottyab.rootbeer.sample -s start --startup-command "android root disable"
```

**Attach** — s'attacher à une app déjà ouverte (méthode utilisée dans ce lab) :

```powershell
# Lancer l'app manuellement sur Genymotion, puis :
objection -n com.scottyab.rootbeer.sample start
```

> **Note sur la syntaxe :** Objection 1.12.5 utilise `-n` (name) et `start` au lieu de l'ancienne syntaxe `-g` et `explore`. Le flag `--startup-command` reste valide pour le mode spawn.

### Résultat attendu

<img width="700" height="268" alt="image" src="https://github.com/user-attachments/assets/57cd75f9-7bba-458a-8740-c37730f3e421" />


Le prompt `com.scottyab.rootbeer.sample (run) on (Android: 9) [usb] #` confirme :
- Package cible correct
- Android 9 (notre Genymotion)
- Connexion USB/ADB active
- Mode `run` (attach sur app en cours)

---

## 6. Étape 4 — État initial et reconnaissance

### 6.1 État de l'app avant bypass

Avant toute instrumentation, l'app RootBeer Sample détecte correctement l'environnement rooté et affiche **ROOTED**.

<img width="452" height="962" alt="rooted" src="https://github.com/user-attachments/assets/8e221337-2496-46fa-b867-debe2356a944" />



### 6.2 Reconnaissance dans la console Objection

Depuis le prompt Objection, on explore l'app avant d'agir :

```bash
# Voir les commandes root disponibles
help android root

# Chercher les classes liées au root
android hooking search classes root

# Chercher les méthodes suspectes
android hooking search methods isRoot
```

**Observations :**

- `help android root` → retourne `No help found` — la commande existe mais sans fichier d'aide associé dans Objection 1.12.5. Elle est néanmoins fonctionnelle.
- `android hooking search methods isRoot` → retourne des résultats de `android.inputmethodservice` — aucune classe RootBeer visible à ce stade car les classes ne sont pas encore chargées en mémoire Java (mode attach, app non encore exécutée).
- `android hooking search classes RootBeer` → aucun résultat pour la même raison.

<img width="912" height="252" alt="image" src="https://github.com/user-attachments/assets/d805126b-56b6-463a-99ac-27d539887be7" />

---

## 7. Étape 5 — Exécution du bypass et validation

### 7.1 Commande de bypass

Dans la console Objection :

```bash
android root disable
```

**Résultat immédiat :**

<img width="731" height="42" alt="image" src="https://github.com/user-attachments/assets/915f4215-70ae-44d0-8255-4f252967425c" />


Objection enregistre un **job** — un hook persistant qui restera actif pendant toute la durée de la session. On peut vérifier les jobs actifs :

```bash
jobs list
```
<img width="793" height="250" alt="image" src="https://github.com/user-attachments/assets/3bc5ead5-cc3b-4ffa-928c-61785899eb85" />


### 7.2 Déclenchement des checks

On appuie sur le bouton **CHECK** (cadenas en bas à droite) dans l'app RootBeer. Les hooks installés par Objection interceptent chaque vérification au moment où elle s'exécute.

### 7.3 Résultats

| Check RootBeer | Avant bypass | Après bypass |
|---|---|---|
| Root Management Apps | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| Potentially Dangerous Apps | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| Root Cloaking Apps | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| TestKeys | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| BusyBoxBinary | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| SU Binary | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| 2nd SU Binary check | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| For RW Paths | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| Dangerous Props | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| Root via native check | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| SE linux Flag Is Enabled | ⊗ DÉTECTÉ | ✅ BYPASSÉ |
| Magisk specific checks | ⊗ DÉTECTÉ | ✅ BYPASSÉ |


**Score : 12/12 — NOT ROOTED ✅**

<img width="451" height="928" alt="image" src="https://github.com/user-attachments/assets/16ec5276-1ac6-42a5-94c6-9b6871289a4f" />

---

## 8. Étape 6 — Ce que fait android root disable

Derrière la commande `android root disable`, Objection installe plusieurs hooks Java via Frida qui neutralisent les techniques de détection root les plus courantes.

### 8.1 Hooks Java installés

#### a) Build.TAGS — test-keys

```javascript
// Ce que fait Objection
const Build = Java.use("android.os.Build");
Object.defineProperty(Build, "TAGS", {
    get: function() { return "release-keys"; }
});
```

Les ROMs officelles ont `Build.TAGS = "release-keys"`. Genymotion retourne `test-keys`. Objection redéfinit la propriété pour masquer cette information.

#### b) File.exists() — Chercher su et busybox

```javascript
File.exists.implementation = function () {
    const path = this.getAbsolutePath();
    if (suspiciousPaths.indexOf(path) !== -1) return false;
    return this.exists.call(this);
};
```

L'app vérifie l'existence de fichiers caractéristiques du root (`/system/xbin/su`, `busybox`, etc.). Objection retourne `false` pour ces chemins.

#### c) Runtime.exec() — Exécuter su

```javascript
Runtime.exec.overload("java.lang.String").implementation = function (cmd) {
    if (cmd.includes("su") || cmd.includes("busybox")) {
        return this.exec("echo");
    }
    return this.exec(cmd);
};
```

L'app essaie d'exécuter `su` directement. Objection remplace la commande par `echo`, inoffensive.

#### d) SystemProperties — Propriétés dangereuses

```javascript
SystemProperties.get.overload("java.lang.String").implementation = function (key) {
    if (key.contains("ro.debuggable")) return "0";
    if (key.contains("ro.secure"))     return "1";
    return this.get(key);
};
```

#### e) RootBeer.isRooted() — Méthode principale

Objection hookе directement la méthode `isRooted()` de RootBeer et force son retour à `false`, neutralisant ainsi l'ensemble du mécanisme de la bibliothèque.

### 8.2 Pourquoi 12/12 avec Objection alors que Frida pur faisait 9/12 ?

En Lab 11, les scripts Frida manuels ne couvraient pas tous les vecteurs — notamment `Dangerous Props` qui utilise des propriétés kernel lues via `/proc/sys`. Objection intègre des patches supplémentaires couvrant ces cas, en particulier en hookant `__system_property_get` au niveau natif pour les propriétés système critiques.

De plus, le mode **attach** utilisé ici a permis d'accrocher l'app après son initialisation complète, évitant les conditions de course qui pouvaient laisser passer certains checks en mode spawn.

---

## 9. Étape 7 — Bonus natif avec frida-trace

Objection cible principalement la couche Java. Pour identifier les appels natifs C/POSIX effectués par RootBeer, on utilise `frida-trace` en parallèle depuis un second terminal.

### 9.1 Trouver le PID exact

```powershell
frida-ps -U | findstr -i root
# → 2329  RootBeer Sample
```

### 9.2 Lancer frida-trace sur le processus

```powershell
frida-trace -U -p 2329 -i "open" -i "access" -i "stat" -i "openat"
```

`frida-trace` génère automatiquement des handlers JavaScript dans `__handlers__/libc.so/` et commence à instrumenter les 4 fonctions POSIX :

```
Instrumenting...
open:   Loaded handler at "C:\frida-lab\__handlers__\libc.so\open.js"
access: Loaded handler at "C:\frida-lab\__handlers__\libc.so\access.js"
stat:   Loaded handler at "C:\frida-lab\__handlers__\libc.so\stat.js"
openat: Loaded handler at "C:\frida-lab\__handlers__\libc.so\openat.js"
Started tracing 4 functions. Web UI available at http://localhost:60508/

```

### 9.3 Analyse des résultats

<img width="1057" height="478" alt="image" src="https://github.com/user-attachments/assets/431f8266-b766-49ae-bb5b-9f52813b033d" />

Les appels visibles dans la trace sont des accès aux **profils ART** (`.prof`) — des artefacts normaux du runtime Android, sans rapport avec la root detection.

**Pourquoi ne voit-on pas `/system/xbin/su` ou `/sbin/magisk` ?**

Objection a déjà neutralisé les vérifications au **niveau Java**, avant que le code natif ne soit atteint. Le flux est le suivant :

```
RootBeer.checkForSuBinary()   ← hooké par Objection → retourne false
        ↓ (jamais atteint)
open("/system/xbin/su")       ← appel natif bloqué en amont
```

En Lab 11 sans Objection, on observait 21+ chemins suspects dans frida-trace. Ici, les hooks Java d'Objection interceptent les vérifications avant qu'elles n'atteignent la couche JNI.

## 10. Concepts clés à retenir

### 10.1 Jobs Objection

Un **job** est un hook persistant enregistré dans la session Objection. Contrairement à un script Frida qui s'exécute une fois, un job reste actif et intercepte chaque appel de la fonction ciblée pendant toute la durée de la session.

```
android root disable
→ Registering job 684511. Name: root-detection-disable
→ Le job intercepte chaque appel à File.exists(), Build.TAGS, etc.
→ jobs list     # voir les jobs actifs
→ jobs kill 684511  # supprimer un job
```

### 10.2 Spawn vs Attach

| Mode | Quand utiliser | Avantage | Inconvénient |
|---|---|---|---|
| **Spawn** (`-s`) | Hooks nécessaires dès le démarrage | Intercepte les checks à l'initialisation | L'app peut détecter l'injection |
| **Attach** | App déjà lancée | Plus discret | Peut rater les checks au démarrage |

**Dans ce lab :** le mode attach a suffi car RootBeer exécute ses checks à la demande (bouton CHECK), pas uniquement au démarrage.

### 10.3 --startup-command

Pour enchaîner plusieurs actions automatiquement au spawn :

```powershell
objection -n com.scottyab.rootbeer.sample -s start \
  --startup-command "android root disable" \
  --startup-command "android sslpinning disable"
```

Les commandes s'exécutent dans l'ordre avant que l'app ne reprenne son exécution.

### 10.4 Couche Java vs couche native dans Objection

Objection cible principalement la **couche Java (ART)**. Pour les checks entièrement natifs (code C/C++ via JNI sans passerelle Java), il faut compléter avec des scripts Frida natifs — Objection ne fournit pas de module universel pour les hooks natifs bas niveau.

```
Couche Java  ←  Objection couvre nativement
Couche JNI   ←  Objection + scripts Frida natifs si nécessaire
```

---

## 11. Comparatif Frida / Medusa / Objection

| Critère | Frida pur | Medusa | Objection |
|---|---|---|---|
| Courbe d'apprentissage | Élevée | Moyenne | Faible |
| Hooks Java | Manuel (`Java.use()`) | Module .med | `android root disable` |
| Hooks natifs | `Interceptor.attach()` | Script séparé | Script Frida séparé requis |
| Exploration interactive | Console JS limitée | Non | Console riche + tab completion |
| Modules prêts | Non | 124 modules | Oui (root, ssl, hooking...) |
| Flexibilité | Totale | Limitée aux modules | Limitée aux commandes intégrées |
| Résultat RootBeer | 9/12 (Java+natif manuel) | 12/12 | 12/12 ✅ |

**Quand utiliser Objection :**
- Pentest rapide sans écriture de code
- Exploration interactive d'une app inconnue
- Bypass SSL pinning + root detection en une seule session
- Formation, démo, PoC rapide

**Quand préférer Frida pur :**
- App obfusquée sans lib tierce identifiable
- Besoin de contrôle précis sur les arguments/retours
- Développement de modules réutilisables
- Hooks natifs fins sur fonctions non exportées

La différence entre eux n'est pas le résultat final (12/12 dans les deux derniers) mais **la méthode** : Frida pur donne le contrôle maximal, Medusa apporte la modularité, Objection apporte la rapidité et l'interactivité. En pentest réel, les trois sont complémentaires.

---




