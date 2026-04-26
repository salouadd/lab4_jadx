# lab4_jadx

🔐 Rapport d’analyse statique — UnCrackable Level 1
📌 Informations générales
Champ	Valeur
📅 Date d'analyse	26 avril 2026
📦 APK analysés	app-debug.apk, UnCrackable-Level1-real.apk
📱 Package principal	sg.vantagepoint.uncrackable1
📚 Provenance	OWASP Mobile Security Testing Guide — Challenge officiel
🛠️ Outils utilisés	JADX GUI, dex2jar, JD-GUI, Python 3 (pycryptodome)
🧠 Résumé exécutif

Cette analyse statique a permis d’identifier 5 vulnérabilités majeures dans l’application.

🚨 Problèmes critiques :
🔑 Clé AES hardcodée dans le bytecode
🔓 Secret récupérable via analyse statique
🛡️ Protections anti-tamper facilement contournables

👉 Niveau de risque global : 🔴 ÉLEVÉ

⚡ Actions prioritaires
❌ Ne jamais stocker de clés cryptographiques dans le code
🌐 Déplacer la logique sensible côté serveur
🔒 Renforcer l’obfuscation et protections runtime
🔍 Constats détaillés
🔴 Constat #1 : Clé AES hardcodée

Sévérité : ÉLEVÉE

📄 Description

La clé AES est stockée en clair dans le code.

byte[] key = b("8d127684cbc37c17616d806cf50473cc");

📍 Localisation :
sg.vantagepoint.uncrackable1.a → méthode a(String)

💥 Impact
Déchiffrement immédiat des données
Contournement total de la sécurité
✅ Remédiation
Utiliser Android Keystore System
Ne jamais hardcoder de clés
🔴 Constat #2 : Secret récupérable (AES-ECB)

Sévérité : ÉLEVÉE

📄 Description
byte[] encrypted = Base64.decode(
"5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=", 0);

➡️ Déchiffrement → "I want to believe"

🧪 PoC Python
from Crypto.Cipher import AES
import base64

key = bytes.fromhex('8d127684cbc37c17616d806cf50473cc')
encrypted = base64.b64decode('5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=')

cipher = AES.new(key, AES.MODE_ECB)
print(cipher.decrypt(encrypted))
⚠️ Problème supplémentaire
Utilisation de AES-ECB (mode faible ❌)
💥 Impact
Extraction du mot de passe sans exécution
✅ Remédiation
Vérification côté serveur
Utiliser AES-GCM / AES-CBC + IV aléatoire
🟡 Constat #3 : Anti-tamper contournable

Sévérité : MOYENNE

📄 Description
if (c.a() || c.b() || c.c()) {
    a("Root detected!");
}

if (b.a(getApplicationContext())) {
    a("App is debuggable!");
}

📍 Localisation : MainActivity.onCreate()

💥 Impact
Bypass facile (patch / Frida)
✅ Remédiation
Implémentation en NDK / JNI
Utiliser DexGuard / R8 / ProGuard
🟡 Constat #4 : Obfuscation insuffisante

Sévérité : MOYENNE

📄 Description
Code facilement lisible avec JADX
Structure logique claire
💥 Impact
Analyse rapide même pour débutant
✅ Remédiation
Control Flow Flattening
Code mort
Renommage agressif
🟡 Constat #5 : Build Debug

Sévérité : MOYENNE

📄 Description
APK compilé en mode debug
android:debuggable="true"
💥 Impact
Debugging possible (ADB)
Inspection mémoire
✅ Remédiation
Utiliser uniquement des builds release
⚖️ Comparaison des outils
Aspect	JADX GUI	JD-GUI
Structure Android	✅ Complète	❌ Java uniquement
Kotlin	✅ Excellent	❌ Mauvais support
Ressources	✅ Oui	❌ Non
Recherche	🔍 Globale	🔎 Limitée
Automatisation	✅ jadx-cli	❌ Non

👉 Conclusion : JADX est supérieur pour Android

🔐 Secrets découverts
Type	Valeur
🔑 Clé AES	8d127684cbc37c17616d806cf50473cc
🔒 Ciphertext	5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=
🧠 Secret final	I want to believe
🔄 Flux de vérification
Utilisateur input
      ↓
MainActivity.verify()
      ↓
a.a(string)
      ↓
AES-ECB Decrypt
      ↓
Compare avec "I want to believe"
      ↓
✅ Success / ❌ Fail
📂 Annexes
📜 Permissions

À compléter depuis :

resources/AndroidManifest.xml

Exemples :

android.permission.INTERNET
android.permission.ACCESS_NETWORK_STATE
🧩 Composants exportés
Composant	Type	Exporté
MainActivity	Activity	✅ Oui
🏁 Conclusion

L’application présente des failles critiques classiques en reverse engineering mobile :

🔓 Secrets exposés côté client
🔑 Mauvaise gestion cryptographique
🛡️ Protections faibles

👉 Ce challenge illustre parfaitement les erreurs à éviter en développement mobile sécurisé.
