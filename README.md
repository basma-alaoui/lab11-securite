LAB 11 — Bypass de la Détection Root Android avec Frida
Hooks Java & Natif (JNI)
      
       Présentation

Ce laboratoire montre comment contourner les mécanismes de détection de root présents dans certaines applications Android sécurisées.

L’analyse est réalisée avec :

Frida
Android Debug Bridge (ADB)
Hooks Java
Hooks Natifs (JNI)

L’application étudiée est :

OWASP MSTG Uncrackable Level 2
        
        Objectifs du Lab
Comprendre comment les applications Android détectent le root
Identifier les contrôles Java et Natifs
Utiliser Frida pour injecter des hooks dynamiques
Neutraliser les protections anti-root
Empêcher la fermeture automatique de l’application
Préparer l’analyse du comportement interne de l’application

      Vérifications Initiales
frida --version
python -c "import frida; print(frida.__version__)"
adb devices

Les versions Frida PC et Android doivent être identiques.

         Étape 1 — Déploiement de frida-server
Identifier l’architecture CPU
adb shell getprop ro.product.cpu.abi
Envoyer le serveur Frida
adb push frida-server /data/local/tmp/
Donner les permissions
adb shell chmod 755 /data/local/tmp/frida-server
Démarrer frida-server
adb shell "/data/local/tmp/frida-server &"
        
         Étape 2 — Redirection des ports Frida
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
         
          Étape 3 — Vérification de la connexion
Lister les applications
frida-ps -Uai

L’application cible doit apparaître dans la liste.

           Étape 4 — Observation du comportement par défaut

Sans instrumentation :

l’application détecte le root,
une alerte de sécurité apparaît,
l’application se ferme immédiatement.

Cette protection empêche toute analyse dynamique classique.

             Étape 5 — Création du script de bypass

Le script bypass_root.js utilise une approche hybride.

 Hooks Java

Les hooks Java servent à :

bloquer l’affichage de l’alerte,
empêcher System.exit() de fermer l’application.
Exemple :
Java.perform(function () {

    var MainActivity = Java.use("sg.vantagepoint.uncrackable2.MainActivity");

    MainActivity.a.implementation = function () {
        console.log("Alerte root interceptée");
    };

    var System = Java.use("java.lang.System");

    System.exit.implementation = function () {
        console.log("System.exit bloqué");
    };
});
Hooks Natifs (JNI)

Certaines protections utilisent du code natif (libc.so).

Le hook natif cible ici :

strncmp

Cela permet plus tard d’intercepter les comparaisons du mot de passe secret.

Exemple :
setInterval(function () {

    var strncmp = Module.findExportByName("libc.so", "strncmp");

    if (strncmp) {

        Interceptor.attach(strncmp, {

            onEnter: function (args) {
                console.log("strncmp appelé");
            }

        });

    }


               Étape 6 — Injection du script
frida -U -f owasp.mstg.uncrackable2 -l bypass_root.js
              
               Étape 7 — Validation du bypass

Après injection :

l’application reste ouverte,
l’alerte root disparaît,
l’interface devient accessible,
le champ de saisie du secret est utilisable.

Le bypass est donc confirmé.


