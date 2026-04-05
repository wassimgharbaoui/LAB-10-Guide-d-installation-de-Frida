# Guide d'installation et de prise en main de Frida

## Présentation de Frida

**Frida** est un framework d'instrumentation dynamique open source, largement utilisé dans le domaine de la sécurité mobile et du reverse engineering. Il permet d'injecter des scripts JavaScript directement dans des processus en cours d'exécution, aussi bien sur Android, iOS, Windows, macOS que Linux.

Dans le contexte de la sécurité des applications mobiles, Frida est un outil incontournable qui permet de :

- Intercepter et modifier des appels de fonctions à la volée
- Contourner des mécanismes de protection (SSL pinning, root detection, etc.)
- Inspecter la mémoire et les modules chargés par une application
- Modifier le comportement d'une application sans accès à son code source

Ce guide détaille l'installation complète de Frida sur un poste de travail Windows, puis son déploiement sur un émulateur Android pour effectuer des tests d'instrumentation dynamique.

> **Cadre légal :** Frida est un outil réservé à des fins d'audit de sécurité, de recherche ou de tests sur des applications dont vous êtes propriétaire ou pour lesquelles vous disposez d'une autorisation explicite. Toute utilisation sur des applications tierces sans autorisation est illégale.

---

## Prérequis

Avant de commencer, assurez-vous de disposer des éléments suivants :

- Python 3.x installé sur votre machine
- `pip` fonctionnel et associé au bon interpréteur Python
- Android Studio avec un émulateur Android configuré (AVD)
- ADB (Android Debug Bridge) accessible depuis le terminal
- Une connexion Internet pour télécharger les binaires Frida

---

## 1. Préparation de l'environnement Python et installation de Frida

La première étape consiste à installer les deux packages Python nécessaires :

- **`frida`** : le module Python qui expose l'API de Frida pour écrire des scripts d'instrumentation
- **`frida-tools`** : une suite d'utilitaires en ligne de commande (dont `frida-ps`, `frida-trace`, etc.)

```bash
pip install frida frida-tools
```

<img width="982" height="552" alt="Screenshot 2026-04-05 185407" src="https://github.com/user-attachments/assets/20446be9-85c8-4b23-9b8b-e9f399e73ce9" />

---

## 2. Résolution des conflits de version Python

Sur une machine où plusieurs versions de Python sont installées simultanément (Python 2.x et Python 3.x par exemple), `pip` peut pointer vers le mauvais interpréteur par défaut. Dans ce cas, il est nécessaire de forcer l'installation via l'interpréteur Python souhaité.

**Commande recommandée pour forcer le bon interpréteur :**

```bash
python -m pip install frida frida-tools
```

Ou de manière encore plus explicite :

```bash
python3 -m pip install frida frida-tools
```

Cette approche garantit que les packages sont installés pour l'interpréteur Python 3 actif dans votre environnement, et non pour une version système potentiellement obsolète.

<img width="1034" height="375" alt="Screenshot 2026-04-05 185416" src="https://github.com/user-attachments/assets/955be07a-b475-462a-8b2f-c80692ca9b16" />

---

## 3. Vérification de l'installation

Une fois l'installation terminée, vérifiez que Frida est correctement installé en affichant sa version :

```bash
frida --version
```

Si la commande retourne un numéro de version (par exemple `16.x.x`), l'installation est réussie. Cette étape est importante car elle confirme que l'outil est accessible depuis le PATH système et prêt à être utilisé.

<img width="567" height="46" alt="Screenshot 2026-04-05 185423" src="https://github.com/user-attachments/assets/c8ab4dc3-e91a-4e3e-95d9-89e0df37106d" />

---

## 4. Vérification de la communication avec l'émulateur — Listage des processus

Avant de déployer le serveur Frida sur l'émulateur, il est utile de lister les processus en cours d'exécution sur l'appareil via ADB pour s'assurer que la communication fonctionne correctement.

```bash
frida-ps -U
```

L'option `-U` indique à Frida d'utiliser l'appareil USB ou l'émulateur connecté. Cette commande retourne la liste de tous les processus actifs avec leur PID et leur nom. Elle est utile pour identifier le nom exact du package de l'application cible avant de lancer un script.

<img width="593" height="654" alt="Screenshot 2026-04-05 185430" src="https://github.com/user-attachments/assets/f0106b0d-42e9-465f-96de-0ddcb140359c" />

---

## 5. Démarrage de l'émulateur et accès root

Frida nécessite des **privilèges root** sur l'appareil Android cible pour pouvoir injecter du code dans les processus. Les émulateurs Android Studio (AVD) basés sur des images système de type `Google APIs` (sans Play Store) permettent l'accès root via ADB.

**Procédure :**

1. Démarrer l'émulateur depuis Android Studio ou en ligne de commande
2. Ouvrir un terminal et accéder au shell de l'émulateur avec élévation de privilèges :

```bash
adb shell
su
```

La commande `su` bascule l'utilisateur vers root. Sur un émulateur standard, cette commande est acceptée sans restriction. Sur un appareil physique, le déverrouillage du bootloader et l'installation d'une ROM rootée sont nécessaires.

<img width="529" height="490" alt="Screenshot 2026-04-05 185437" src="https://github.com/user-attachments/assets/2372000b-aec1-4150-8cc2-fc95b6c0175b" />

---

## 6. Déploiement de frida-server sur l'émulateur Android

Le composant **frida-server** est un binaire natif Android qui doit être poussé et exécuté sur l'appareil cible. C'est lui qui reçoit les connexions entrantes de Frida côté hôte et qui orchestre l'injection dans les processus Android.

### Téléchargement de frida-server

Téléchargez la version de `frida-server` qui correspond **exactement** à la version de Frida installée sur votre machine hôte. Les versions doivent impérativement correspondre, sinon la connexion échouera.

Rendez-vous sur : [https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)

Choisissez l'archive correspondant à l'architecture de votre émulateur :

| Architecture émulateur | Fichier à télécharger |
|---|---|
| x86 (émulateur standard) | `frida-server-x.x.x-android-x86.xz` |
| x86_64 (émulateur 64 bits) | `frida-server-x.x.x-android-x86_64.xz` |
| arm64 (appareil physique récent) | `frida-server-x.x.x-android-arm64.xz` |

Pour vérifier l'architecture de votre émulateur :

```bash
adb shell getprop ro.product.cpu.abi
```

### Transfert du binaire vers l'émulateur

```bash
adb push frida-server /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
```

La commande `chmod 755` rend le binaire exécutable. Sans cette étape, Android refuserait de lancer le programme.

<img width="1001" height="76" alt="Screenshot 2026-04-05 185446" src="https://github.com/user-attachments/assets/a0be4a80-0fff-4784-aa3c-4fac415f84a2" />

---

## 7. Identification de l'application cible

Avant de lancer un test d'instrumentation, il est nécessaire de connaître le **nom de package exact** de l'application à analyser. La commande suivante liste toutes les applications installées sur l'émulateur :

```bash
frida-ps -Uai
```

- `-U` : utiliser l'appareil USB/émulateur
- `-a` : afficher uniquement les applications (pas les processus système)
- `-i` : inclure les applications non démarrées

Pour filtrer sur un nom précis :

```bash
frida-ps -Uai | grep -i "pizzarecipes"
```

Notez le nom de package retourné (ex. `com.example.pizzarecipes`), il sera utilisé dans les commandes Frida ultérieures.

<img width="868" height="649" alt="Screenshot 2026-04-05 185454" src="https://github.com/user-attachments/assets/05dc29fd-45ec-4521-b429-61ad0ca9f8a2" />

---

## 8. Lancement de frida-server

Depuis le shell root de l'émulateur, démarrez frida-server en arrière-plan :

```bash
adb shell
su
/data/local/tmp/frida-server &
```

Le caractère `&` permet de lancer le processus en arrière-plan pour libérer le terminal. Si vous souhaitez voir les logs en temps réel pour diagnostiquer un problème, lancez sans `&` dans un terminal dédié.

**Vérification du bon démarrage :**

Depuis un autre terminal sur la machine hôte :

```bash
frida-ps -U
```

Si la liste de processus s'affiche sans erreur, frida-server est opérationnel et la communication est établie.

<img width="899" height="22" alt="Screenshot 2026-04-05 185500" src="https://github.com/user-attachments/assets/f601f8f7-5848-48a8-a31f-7bb01e4fcba1" />

---

## 9. Premier script Frida — hello.js

Un script Frida est un fichier JavaScript injecté dans le processus cible. Il s'exécute dans le contexte de l'application et peut accéder à ses classes, méthodes et données en temps réel.

Créez un fichier nommé `hello.js` avec le contenu suivant :

```javascript
Java.perform(function () {
    console.log("[*] Script Frida injecté avec succès !");

    // Enumération des classes de l'application cible
    Java.enumerateLoadedClasses({
        onMatch: function(className) {
            if (className.indexOf("com.example") !== -1) {
                console.log("[+] Classe détectée : " + className);
            }
        },
        onComplete: function() {
            console.log("[*] Enumération terminée.");
        }
    });
});
```

`Java.perform()` est le point d'entrée obligatoire de tout script Frida ciblant le runtime Java/Android. Tout le code d'instrumentation doit se trouver à l'intérieur de cette fonction.


<img width="875" height="266" alt="Screenshot 2026-04-05 185506" src="https://github.com/user-attachments/assets/bf77f0f9-7023-4a73-a4dc-33a3be125d76" />

---

## 10. Exécution du script sur l'application cible

Pour injecter le script dans une application en cours d'exécution, utilisez la commande suivante :

```bash
frida -U -l hello.js -f com.example.pizzarecipes
```

**Détail des options :**

| Option | Description |
|---|---|
| `-U` | Connexion à l'appareil USB ou émulateur |
| `-l hello.js` | Chemin vers le script JavaScript à injecter |
| `-f com.example.pizzarecipes` | Lance l'application avec Frida dès le démarrage (spawn) |

Si l'application est déjà en cours d'exécution et que vous souhaitez vous y attacher sans la redémarrer :

```bash
frida -U -l hello.js com.example.pizzarecipes
```

La différence est importante : `-f` tue et relance l'application (utile pour intercepter le démarrage), tandis que l'attachement direct ne perturbe pas l'exécution en cours.


<img width="721" height="432" alt="Screenshot 2026-04-05 185511" src="https://github.com/user-attachments/assets/9544809e-9a3c-49e8-89ce-b88e198605f5" />

---

## 11. Inspection avancée du processus Android

Frida permet également d'inspecter des propriétés bas niveau du processus ciblé. Voici un script couvrant plusieurs cas d'usage d'inspection :

```javascript
Java.perform(function () {

    // 1. Vérifier l'architecture du processus
    var arch = Process.arch;
    console.log("[*] Architecture du processus : " + arch);

    // 2. Identifier le module principal de l'application
    var mainModule = Process.enumerateModules()[0];
    console.log("[*] Module principal : " + mainModule.name);
    console.log("    Base address : " + mainModule.base);
    console.log("    Taille       : " + mainModule.size + " bytes");

    // 3. Inspecter une bibliothèque système critique
    var libssl = Process.findModuleByName("libssl.so");
    if (libssl !== null) {
        console.log("[*] libssl.so chargée à : " + libssl.base);
    } else {
        console.log("[-] libssl.so non détectée dans ce processus.");
    }

    // 4. Vérifier la présence d'une fonction sensible
    var openFunc = Module.findExportByName(null, "open");
    if (openFunc !== null) {
        console.log("[*] Fonction 'open' trouvée à : " + openFunc);
    }

});
```

Ces quatre opérations constituent une base solide pour l'analyse de n'importe quelle application Android :

- **Architecture** : détermine si l'on est sur un processus 32 ou 64 bits
- **Module principal** : identifie le binaire de l'application et son adresse de base en mémoire
- **Bibliothèques système** : permet de localiser des bibliothèques comme `libssl.so` pour intercepter les communications TLS
- **Fonctions exportées** : utile pour identifier des points d'accroche (hooks) potentiels


<img width="988" height="596" alt="Screenshot 2026-04-05 185520" src="https://github.com/user-attachments/assets/45a7fb84-c9e1-4af9-8a6f-5840cf21c9b4" />

---

## 12. Modification du comportement de l'application — Exemple pratique

L'un des cas d'usage les plus démonstratifs de Frida est la **modification à la volée du comportement d'une méthode Java**. L'exemple suivant illustre comment intercepter une méthode d'incrémentation et en modifier la logique : au lieu d'incrémenter d'une unité à chaque appui sur le bouton, on force une incrémentation de trois unités.

```javascript
Java.perform(function () {

    // Cibler la classe contenant la logique du compteur
    var TargetClass = Java.use("com.example.pizzarecipes.MainActivity");

    // Hooker la méthode d'incrémentation
    TargetClass.increment.implementation = function () {
        console.log("[*] Méthode increment() interceptée !");

        // Appel original (incrémente de 1)
        this.increment();

        // Appels supplémentaires pour forcer +3 au total
        this.increment();
        this.increment();

        console.log("[*] Valeur forcée à +3 au lieu de +1");
    };

});
```

**Ce que cela démontre :**

- Frida peut **remplacer complètement** l'implémentation d'une méthode Java à l'exécution
- Il est possible d'appeler la méthode originale (`this.increment()`) autant de fois que souhaité
- Cette technique est utilisée en pentest pour contourner des vérifications de licence, des mécanismes anti-triche ou des contrôles d'authentification


<img width="963" height="631" alt="Screenshot 2026-04-05 185529" src="https://github.com/user-attachments/assets/53c0dffc-d243-4dc4-a7ab-6ef9f30626fc" />

---

## 13. Retour de Frida — Confirmation de l'instrumentation

Lorsqu'un script Frida est injecté et s'exécute correctement, la console affiche les messages définis dans les appels `console.log()` du script. Ces sorties confirment que :

- La connexion entre la machine hôte et frida-server est active
- Le script a bien été injecté dans le processus cible
- Les hooks définis sont opérationnels et interceptent les appels attendus

Cette étape de validation est essentielle avant de passer à des scripts plus complexes.

<img width="1025" height="630" alt="Screenshot 2026-04-05 185538" src="https://github.com/user-attachments/assets/02dfa6fa-42e5-4ee3-98f4-d8604b8a68af" />

---

## Récapitulatif des commandes essentielles

| Commande | Description |
|---|---|
| `pip install frida frida-tools` | Installer Frida et ses outils sur la machine hôte |
| `frida --version` | Vérifier la version installée |
| `frida-ps -U` | Lister les processus sur l'appareil connecté |
| `frida-ps -Uai` | Lister toutes les applications installées |
| `adb push frida-server /data/local/tmp/` | Transférer frida-server vers l'émulateur |
| `adb shell chmod 755 /data/local/tmp/frida-server` | Rendre frida-server exécutable |
| `/data/local/tmp/frida-server &` | Démarrer frida-server en arrière-plan |
| `frida -U -l script.js -f com.example.app` | Lancer l'application et injecter un script |
| `frida -U -l script.js com.example.app` | Attacher Frida à une application déjà démarrée |

---

## Dépannage courant

| Symptôme | Cause probable | Solution |
|---|---|---|
| `Failed to enumerate processes` | frida-server non démarré | Lancer `/data/local/tmp/frida-server &` en root |
| `Unable to connect to remote frida-server` | Version mismatch | Vérifier que frida et frida-server ont la même version |
| `spawn: unable to find process` | Package name incorrect | Vérifier avec `frida-ps -Uai` |
| `permission denied` sur frida-server | Permissions insuffisantes | Exécuter `chmod 755` et relancer en tant que root |
| `frida: command not found` | Frida non dans le PATH | Utiliser `python -m frida` ou vérifier le PATH |

---

## Ressources complémentaires

- Documentation officielle Frida : [https://frida.re/docs/](https://frida.re/docs/)
- Releases frida-server : [https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)
- OWASP MASTG — Tests dynamiques Android : [https://mas.owasp.org/MASTG/](https://mas.owasp.org/MASTG/)
- Frida CodeShare (scripts communautaires) : [https://codeshare.frida.re/](https://codeshare.frida.re/)

---

*Guide rédigé par **Wassim Gharbaoui** — Installation et prise en main de Frida pour l'analyse dynamique Android*
