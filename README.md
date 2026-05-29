# LAB 14 : Bypass Root Detection sur Android
## Techniques dynamiques avec Frida, Objection et hooks natifs

## 1. Introduction

Ce laboratoire a pour objectif de contourner dynamiquement la détection de root d’une application Android à l’aide de plusieurs approches complémentaires :

- un script Frida Java ;
- un script Frida natif ciblant certaines fonctions libc ;
- Objection avec la commande `android root disable`.

L’application utilisée est :

```text
RootBeer Sample
Package : com.scottyab.rootbeer.sample
APK : RootBeer_Sample-0.9.apk
```

RootBeer Sample est une application de test qui vérifie plusieurs indicateurs de root sur un appareil Android : présence de `su`, présence de `busybox`, propriétés système dangereuses, chemins système modifiés, applications de gestion du root et checks natifs.

---

## 2. Objectifs du lab

Les objectifs sont :

- vérifier le fonctionnement d’ADB ;
- démarrer `frida-server` sur l’émulateur Android ;
- valider l’injection Frida avec un script simple ;
- créer un script Java de bypass root ;
- intercepter les checks Java comme `Build.TAGS`, `File.exists()` et `Runtime.exec()` ;
- neutraliser les méthodes RootBeer ;
- ajouter des hooks natifs sur `open`, `openat`, `access`, `stat`, `lstat` et `fopen` ;
- utiliser Objection pour automatiser le bypass avec `android root disable` ;
- comparer les résultats avant et après bypass.

---

## 3. Environnement de travail

| Élément | Valeur |
|---|---|
| Système hôte | Windows |
| Terminal | PowerShell |
| Appareil Android | Émulateur Android |
| Version Android utilisée | Android 11 |
| Outil d’instrumentation | Frida |
| Version Frida observée | 17.9.11 |
| Surcouche Frida | Objection |
| Application cible | RootBeer Sample |
| Package cible | `com.scottyab.rootbeer.sample` |
| APK utilisé | `RootBeer_Sample-0.9.apk` |

---

## 4. Prérequis

Les prérequis nécessaires sont :

- Python 3.8 ou supérieur ;
- pip ;
- ADB / Android Platform Tools ;
- Frida côté PC ;
- `frida-server` côté Android ;
- Objection ;
- un émulateur Android avec le débogage USB activé ;
- l’APK RootBeer Sample.

Commandes de vérification :

```powershell
python --version
pip --version
adb version
frida --version
objection --version
```


## 5. Vérification ADB

Sous Windows, le chemin ADB utilisé est :

```powershell
$ADB = "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
```

Vérification de l’émulateur :

```powershell
& $ADB devices
```

Résultat attendu :

```text
List of devices attached
emulator-5554    device
```

Cette étape confirme que l’émulateur Android est détecté par ADB.



## 6. Installation de RootBeer Sample

L’APK utilisé est situé sur le PC dans le chemin suivant :

```text
C:\Users\halim\Downloads\RootBeer_Sample-0.9.apk
```

Installation de l’application :

```powershell
$APK = "C:\Users\halim\Downloads\RootBeer_Sample-0.9.apk"
& $ADB install -r $APK
```

Résultat attendu :

```text
Success
```

Identification du package :

```powershell
& $ADB shell pm list packages | Select-String -Pattern "rootbeer"
```

Résultat attendu :

```text
package:com.scottyab.rootbeer.sample
```
<img width="803" height="98" alt="image" src="https://github.com/user-attachments/assets/c2b160eb-fb63-43b7-954a-ee8a3b47d882" />


## 7. Démarrage de frida-server

L’architecture CPU de l’émulateur est vérifiée avec :

```powershell
& $ADB shell getprop ro.product.cpu.abi
```

Exemple de résultat :

```text
x86_64
```

Le binaire `frida-server` correspondant à l’architecture et à la version Frida côté PC est poussé vers l’émulateur :

```powershell
& $ADB push .\frida-server /data/local/tmp/frida-server
& $ADB shell chmod 755 /data/local/tmp/frida-server
```

Lancement de `frida-server` :

```powershell
& $ADB shell "/data/local/tmp/frida-server -l 0.0.0.0:27042"
```

Dans un deuxième terminal, la connexion est vérifiée :

```powershell
frida-ps -Uai
```


## 8. Test d’injection Frida avec `hello.js`

Avant de créer un bypass, un script simple a été utilisé pour vérifier que Frida peut injecter du JavaScript dans l’application.

Création du fichier :

```powershell
notepad .\hello.js
```

Contenu :

```javascript
Java.perform(function () {
    console.log("[+] Script injecté : Java.perform OK");
});
```

Lancement :

```powershell
frida -U -f com.scottyab.rootbeer.sample -l .\hello.js
```

Résultat attendu :

```text
Spawned `com.scottyab.rootbeer.sample`
[+] Script injecté : Java.perform OK
```

<img width="993" height="360" alt="image" src="https://github.com/user-attachments/assets/22c5990c-f715-474d-88d6-f06e57b05416" />


## 9. État initial avant bypass

L’application RootBeer Sample est lancée normalement :

```powershell
& $ADB shell monkey -p com.scottyab.rootbeer.sample -c android.intent.category.LAUNCHER 1
```

Avant bypass, l’application peut détecter l’environnement comme rooté à travers plusieurs contrôles :

```text
Root Management Apps
Potentially Dangerous Apps
Root Cloaking Apps
TestKeys
BusyBoxBinary
SU Binary
2nd SU Binary check
For RW Paths
Dangerous Props
Root via native check
SELinux Flag Is Enabled
Magisk specific checks
```

<img width="569" height="774" alt="Capture d&#39;écran 2026-05-29 005523" src="https://github.com/user-attachments/assets/2823c0a7-db08-4fe5-8fb7-f0cd617a3dc8" />


## 10. Script Java `bypass_root_basic.js`

Le premier script permet de neutraliser les checks Java classiques :

- modification de `Build.TAGS` ;
- hook des méthodes RootBeer ;
- interception de `java.io.File.exists()` ;
- blocage de certaines commandes `Runtime.exec()` ;
- neutralisation de `RootBeerNative.checkForRoot()`.

Création du fichier :

```powershell
notepad .\bypass_root_basic.js
```

Résumé des éléments hookés :

| Élément hooké | Objectif |
|---|---|
| `android.os.Build.TAGS` | Retourner `release-keys` |
| `com.scottyab.rootbeer.RootBeer` | Forcer les méthodes de détection à `false` |
| `com.scottyab.rootbeer.RootBeerNative` | Forcer le check natif à `0` |
| `java.io.File.exists()` | Masquer les chemins `su` et `busybox` |
| `Runtime.exec()` | Bloquer `su`, `which su`, `busybox` |

Lancement du script :

```powershell
& $ADB shell am force-stop com.scottyab.rootbeer.sample
frida -U -f com.scottyab.rootbeer.sample -l .\bypass_root_basic.js
```

Résultat observé :

```text
Spawned `com.scottyab.rootbeer.sample`. Resuming main thread!
[*] Root bypass Java started
[+] Build.TAGS -> release-keys
[+] RootBeer Java hooks installed
[+] RootBeerNative hook installed
[+] File.exists hook installed
[+] Runtime.exec hooks installed
[+] Bypass Java installé
[+] File.exists bypass: /system/bin/busybox
[+] File.exists bypass: /system/xbin/busybox
[+] File.exists bypass: /data/local/su
[+] File.exists bypass: /data/local/bin/su
[+] File.exists bypass: /data/local/xbin/su
[+] File.exists bypass: /sbin/su
[+] File.exists bypass: /su/bin/su
[+] File.exists bypass: /system/bin/su
[+] File.exists bypass: /system/xbin/su
[+] RootBeerNative.checkForRoot() -> 0
```
<img width="1012" height="711" alt="image" src="https://github.com/user-attachments/assets/0dff413e-a0e9-4c3b-a3c4-73019cdd0673" />




## 11. Script natif `bypass_native.js`

Certaines applications peuvent réaliser des checks root côté natif avec du code C/C++ en appelant des fonctions système comme :

```text
open
openat
access
stat
lstat
fopen
```

Le script natif sert à intercepter ces appels et à faire échouer les recherches vers certains chemins suspects.

Chemins suspects ciblés :

```text
/system/bin/su
/system/xbin/su
/sbin/su
/system/su
/system/bin/busybox
/system/xbin/busybox
/data/local/bin/su
/data/local/xbin/su
/data/local/su
/su/bin/su
/proc/mounts
/proc/self/mounts
```

Fonctions natives hookées :

```text
open
openat
access
stat
lstat
fopen
```

---

## 14. Problème rencontré avec Frida 17

Lors du premier lancement du script natif, une erreur est apparue :

```text
TypeError: not a function
    at hookLibc
```

Cette erreur était liée à l’utilisation d’une ancienne API dans le script natif :

```javascript
Module.findExportByName(...)
```

Avec Frida 17, le script a été corrigé en utilisant une méthode compatible :

```javascript
const mod = Process.getModuleByName("libc.so");
const addr = mod.findExportByName(exportName);
```

ou en fallback :

```javascript
Module.findGlobalExportByName(exportName);
```

Interprétation :

```text
Le problème ne venait pas de frida-server, mais du script natif. La correction a permis d’adapter le hook natif à Frida 17.9.11.
```



## 15. Lancement combiné Java + natif

Après correction du script natif, l’application est lancée avec les deux scripts :

```powershell
& $ADB shell am force-stop com.scottyab.rootbeer.sample
frida -U -f com.scottyab.rootbeer.sample -l .\bypass_root_basic.js -l .\bypass_native.js
```

Ou avec la version améliorée :

```powershell
frida -U -f com.scottyab.rootbeer.sample -l .\bypass_root_basic_v2.js -l .\bypass_native.js
```

Résultat attendu :

```text
[+] Bypass Java installé
[*] Native root bypass started
[+] Hooked libc: open
[+] Hooked libc: openat
[+] Hooked libc: access
[+] Hooked libc: stat
[+] Hooked libc: lstat
[+] Hooked libc: fopen
[+] Native hooks installed
```
<img width="1047" height="880" alt="image" src="https://github.com/user-attachments/assets/931497af-8890-402c-b0ef-2f3876146665" />


## 16. Validation dans RootBeer Sample après Frida

Après l’injection des scripts Frida, l’application doit afficher :

```text
NOT ROOTED
```

ou présenter tous les checks en vert.

<img width="449" height="783" alt="Capture d&#39;écran 2026-05-29 005554" src="https://github.com/user-attachments/assets/f5dade46-25de-4ee1-abb3-d7e7b8dbd700" />


Interprétation :

```text
La détection de root est neutralisée dynamiquement.
Les chemins suspects sont masqués.
Les méthodes RootBeer retournent des valeurs non suspectes.
Les checks natifs sont interceptés.
```

---

## 17. Bypass avec Objection

Objection est une surcouche Frida qui permet d’automatiser plusieurs hooks.

L’application est d’abord arrêtée :

```powershell
& $ADB shell am force-stop com.scottyab.rootbeer.sample
```

Lancement d’Objection :

```powershell
objection -g com.scottyab.rootbeer.sample explore
```

Dans la console Objection :

```bash
android root disable
```

Résultat observé :

```text
(agent) Registering job ... Name: root-detection-disable
(agent) RootBeer->detectRootCloakingApps() check detected, marking as false.
(agent) RootBeer->detectTestKeys() check detected, marking as false.
(agent) RootBeer->checkForBinary() check detected, marking as false.
(agent) RootBeer->checkSuExists() check detected, marking as false.
(agent) RootBeer->checkForDangerousProps() check detected, marking as false.
(agent) RootBeerNative->checkForRoot() check detected, marking as 0.
```

<img width="975" height="312" alt="Capture d&#39;écran 2026-05-29 002747" src="https://github.com/user-attachments/assets/e51f8958-a997-4783-903c-0e3bf3d8cc5c" />


## 18. Validation après Objection

Après l’exécution de :

```bash
android root disable
```

RootBeer Sample affiche :

```text
NOT ROOTED
```

Tous les checks visibles sont en vert.

<img width="449" height="783" alt="Capture d&#39;écran 2026-05-29 005554" src="https://github.com/user-attachments/assets/f9663294-2b7d-4b2b-b143-c9b012e3ad0c" />

Interprétation :

```text
Objection a intercepté automatiquement les méthodes RootBeer.
Les méthodes de détection retournent false ou 0.
La détection root est contournée.
```

---



## 19. Résumé des résultats

| Test | Résultat |
|---|---|
| ADB détecte l’émulateur | Réussi |
| frida-server lancé | Réussi |
| `frida-ps -Uai` fonctionne | Réussi |
| `hello.js` injecté | Réussi |
| Script Java Frida lancé | Réussi |
| `Build.TAGS` modifié | Réussi |
| `File.exists()` intercepté | Réussi |
| Chemins `su` et `busybox` masqués | Réussi |
| `Runtime.exec()` hooké | Réussi |
| RootBeer Java hooké | Réussi |
| RootBeerNative hooké | Réussi |
| Hook natif libc corrigé | Réussi |
| Objection `android root disable` | Réussi |
| Résultat final | `NOT ROOTED` |

---

## 20. Comparaison des méthodes

| Méthode | Avantage | Limite |
|---|---|---|
| Frida Java | Contrôle précis des méthodes Java | Peut rater des checks natifs |
| Frida natif | Intercepte les appels système natifs | Demande plus de connaissances techniques |
| Objection | Simple et rapide | Moins personnalisable |
| frida-trace | Découverte des fonctions réellement appelées | Ne bypass pas directement |

---

## 21. Commandes principales utilisées

### Déclarer ADB

```powershell
$ADB = "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
```

### Vérifier l’émulateur

```powershell
& $ADB devices
```

### Identifier le package

```powershell
& $ADB shell pm list packages | Select-String -Pattern "rootbeer"
```

### Lancer frida-server

```powershell
& $ADB shell "/data/local/tmp/frida-server -l 0.0.0.0:27042"
```

### Vérifier Frida

```powershell
frida-ps -Uai
```

### Lancer le test hello.js

```powershell
frida -U -f com.scottyab.rootbeer.sample -l .\hello.js
```

### Lancer le bypass Java

```powershell
frida -U -f com.scottyab.rootbeer.sample -l .\bypass_root_basic.js
```

### Lancer le bypass Java amélioré

```powershell
frida -U -f com.scottyab.rootbeer.sample -l .\bypass_root_basic_v2.js
```

### Lancer le bypass Java + natif

```powershell
frida -U -f com.scottyab.rootbeer.sample -l .\bypass_root_basic_v2.js -l .\bypass_native.js
```

### Lancer Objection

```powershell
objection -g com.scottyab.rootbeer.sample explore
```

### Désactiver le root avec Objection

```bash
android root disable
```

---


## 22. Conclusion

Ce lab a permis de contourner la détection de root d’une application Android en utilisant plusieurs techniques dynamiques.

Dans un premier temps, un script Frida Java a été injecté dans l’application RootBeer Sample. Ce script a permis de modifier `Build.TAGS`, d’intercepter les appels `File.exists()`, de bloquer certaines commandes `Runtime.exec()` et de neutraliser plusieurs méthodes RootBeer.

Ensuite, une partie native a été ajoutée avec un script qui hooke des fonctions libc comme `open`, `openat`, `access`, `stat`, `lstat` et `fopen`. Une erreur liée à l’API Frida 17 a été rencontrée puis corrigée en adaptant la recherche des symboles natifs.

Enfin, le bypass a été validé avec Objection grâce à la commande `android root disable`. Les logs Objection montrent que les méthodes RootBeer ont été interceptées et forcées à retourner des valeurs négatives.

Le résultat final obtenu est :

```text
NOT ROOTED
```

Ce résultat confirme que le bypass dynamique de la détection root a été réalisé avec succès.

---

## 23. Remarque éthique

Les techniques présentées dans ce lab doivent être utilisées uniquement dans un cadre autorisé :

- application de test ;
- environnement pédagogique ;
- audit de sécurité encadré ;
- appareil appartenant à l’auditeur.

Elles ne doivent jamais être utilisées sur des applications tierces sans autorisation explicite.
