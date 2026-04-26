# lab4_jadx

Rapport d'analyse statique — UnCrackable Level 1 & app-debug.apk
Informations générales
ChampValeurDate d'analyse26 avril 2026APK analysésapp-debug.apk, UnCrackable-Level1-real.apkPackage principalsg.vantagepoint.uncrackable1ProvenanceOWASP Mobile Security Testing Guide (MSTG) — Challenge officielOutils utilisésJADX GUI, dex2jar, JD-GUI, Python 3 (pycryptodome)

Résumé exécutif
Cette analyse statique a révélé 5 vulnérabilités potentielles dans l'application UnCrackable-Level1.
Les principales préoccupations concernent :

Une clé AES hardcodée directement dans le bytecode, permettant le déchiffrement complet du secret sans exécuter l'application.
Un secret de vérification récupérable par analyse statique (texte clair après déchiffrement : I want to believe).
Des mécanismes anti-tamper contournables (détection root et debug réalisés côté Java, neutralisables statiquement).

Le niveau de risque global est évalué comme ÉLEVÉ.
Actions prioritaires recommandées

Ne jamais stocker de clés cryptographiques en clair dans le code source ou le bytecode.
Implémenter la vérification des secrets côté serveur plutôt que côté client.
Renforcer les mécanismes anti-tamper avec des techniques natives (NDK/JNI) et des solutions d'obfuscation avancées.


Constats détaillés
Constat #1 : Clé AES hardcodée dans le bytecode
Sévérité : 🔴 Élevée
Description : La clé de chiffrement AES est stockée en clair sous forme hexadécimale directement dans la classe a. Toute personne disposant de JADX peut l'extraire en quelques secondes sans exécuter l'application.
Localisation : sg.vantagepoint.uncrackable1.a — méthode a(String), ligne 10
javabyte[] key = b("8d127684cbc37c17616d806cf50473cc");
Impact potentiel : Déchiffrement immédiat de tous les secrets protégés par cette clé. Contournement total de la protection de l'application.
Remédiation recommandée : Ne jamais stocker de clés cryptographiques dans le code. Utiliser Android Keystore System pour dériver ou stocker les clés de façon sécurisée.

Constat #2 : Secret de vérification récupérable par analyse statique (AES-ECB)
Sévérité : 🔴 Élevée
Description : Le secret attendu est chiffré en Base64 et stocké dans le bytecode. Combiné à la clé hardcodée (Constat #1), le secret peut être déchiffré hors ligne. De plus, l'algorithme utilisé est AES-ECB, un mode réputé faible car il ne dispose pas de vecteur d'initialisation (IV), rendant le chiffrement déterministe.
Localisation : sg.vantagepoint.uncrackable1.a — méthode a(String)
javabyte[] encrypted = Base64.decode("5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=", 0);
// Résultat déchiffré : "I want to believe"
Preuve de concept (Python) :
pythonfrom Crypto.Cipher import AES
import base64

key = bytes.fromhex('8d127684cbc37c17616d806cf50473cc')
encrypted = base64.b64decode('5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=')
cipher = AES.new(key, AES.MODE_ECB)
print(cipher.decrypt(encrypted))
# b'I want to believe\x0f\x0f...'
Secret découvert : I want to believe
Impact potentiel : N'importe quel analyste peut retrouver le mot de passe secret sans interaction avec l'application.
Remédiation recommandée : Effectuer la vérification côté serveur. Si une vérification client est indispensable, utiliser AES-GCM ou AES-CBC avec un IV aléatoire, et dériver la clé via PBKDF2 ou une HSM.

Constat #3 : Mécanismes anti-tamper contournables (Root & Debug Detection)
Sévérité : 🟡 Moyenne
Description : L'application implémente une détection de root (3 méthodes : c.a(), c.b(), c.c()) et une détection du mode debug (b.a(context)). Ces protections sont implémentées entièrement en Java/Kotlin, ce qui les rend trivialement contournables par patch du bytecode ou via des outils comme Frida.
Localisation : sg.vantagepoint.uncrackable1.MainActivity — méthode onCreate()
javaif (c.a() || c.b() || c.c()) {
    a("Root detected!");   // → System.exit(0)
}
if (b.a(getApplicationContext())) {
    a("App is debuggable!"); // → System.exit(0)
}
Impact potentiel : Un attaquant peut patcher le bytecode pour NOP ces vérifications et exécuter l'application sur un device rooté ou en mode debug, facilitant l'analyse dynamique.
Remédiation recommandée : Implémenter les vérifications anti-tamper en code natif (NDK/JNI). Utiliser des solutions spécialisées (ProGuard, R8, DexGuard, Guardsquare) pour l'obfuscation et la protection runtime.

Constat #4 : Obfuscation insuffisante des classes critiques
Sévérité : 🟡 Moyenne
Description : Bien que certaines classes soient obfusquées avec des noms courts (a, b, c), la logique de l'application reste parfaitement lisible après décompilation avec JADX. Les noms de méthodes et la structure du code permettent de comprendre immédiatement le flux de vérification.
Localisation : Packages sg.vantagepoint.a et sg.vantagepoint.uncrackable1
Impact potentiel : Facilite considérablement l'analyse statique. Un analyste débutant peut comprendre la logique de l'application en moins de 30 minutes.
Remédiation recommandée : Appliquer une obfuscation agressive avec renommage de méthodes, insertion de code mort, et aplatissement du flux de contrôle (control flow flattening).

Constat #5 : Application compilée en mode Debug
Sévérité : 🟡 Moyenne
Description : Le fichier analysé est app-debug.apk, indiquant une version de debug. L'application elle-même détecte ce flag et affiche "App is debuggable!" — mais cela confirme que l'APK ne devrait pas être distribué en production dans cet état.
Localisation : AndroidManifest.xml — attribut android:debuggable="true" (implicite pour les builds debug)
Impact potentiel : Permet l'attachement d'un débogueur (ADB), l'inspection de la mémoire en runtime, et facilite le contournement des protections.
Remédiation recommandée : S'assurer que seuls les builds release (avec android:debuggable="false") sont distribués. Automatiser cette vérification dans le pipeline CI/CD.

Comparaison JADX GUI vs JD-GUI (Task 6)
AspectJADX GUIJD-GUIStructure affichéeStructure Android complète (Manifest, ressources, code, assets)Uniquement la structure Java (packages, classes)Support KotlinExcellente gestion du bytecode Kotlin, syntaxe lisibleDifficultés avec Kotlin — syntaxe parfois illisible ou incorrecteAccès aux ressourcesAccès direct à strings.xml, AndroidManifest.xml, network_security_config.xmlAucun accès aux ressources AndroidObfuscationTente de reconstruire les noms logiques, indique les classes anonymesConserve les noms obfusqués sans contexte supplémentaireRecherche globaleRecherche full-text dans tout le projet (code + ressources)Recherche limitée au code Java ouvertIntégration workflowExport en ligne de commande (jadx-cli), idéal pour l'automatisationOutil purement graphique
Conclusion : JADX GUI est l'outil le plus adapté pour l'analyse d'APKs Android, notamment grâce à son accès aux ressources et à son support Kotlin. JD-GUI reste utile comme outil secondaire pour une seconde opinion sur du code Java pur.

Annexes
Permissions demandées

⚠️ À compléter après consultation de AndroidManifest.xml dans JADX
(Navigation : Inputs > Files > UnCrackable-Level1-real.apk > resources > AndroidManifest.xml)

Permissions typiques à documenter :

android.permission.INTERNET (si présente)
android.permission.ACCESS_NETWORK_STATE (si présente)

Composants exportés
ComposantTypeExportéIntent-filterNotesMainActivityActivity✅ Ouiandroid.intent.action.MAINPoint d'entrée principal de l'application
Résumé des secrets découverts
SecretValeurMéthode de découverteClé AES8d127684cbc37c17616d806cf50473ccAnalyse statique (JADX)Texte chiffré5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=Analyse statique (JADX)Secret finalI want to believeDéchiffrement AES-ECB (Python)
Flux de vérification du secret (résumé)
Utilisateur saisit un texte
        ↓
MainActivity.verify(View)
        ↓
a.a(string)   ← sg.vantagepoint.uncrackable1.a
        ↓
Déchiffrement AES-ECB
  Clé   : 8d127684cbc37c17616d806cf50473cc
  Cipher: 5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=
        ↓
Comparaison : str.equals("I want to believe")
        ↓
✅ Success! / ❌ Nope...
