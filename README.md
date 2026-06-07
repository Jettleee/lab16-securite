# Lab 16 – Bypass SSL Pinning avec Objection sur appareil physique

**Auteur :** Charaf
**Application cible :** `sg.vp.owasp_mobile.omtg_android` — challenge `OMTG_NETW_004`
**Outils :** Burp Suite, ADB, Frida, Objection

---

## Introduction

Ce lab pousse plus loin le lab précédent : au lieu d'un émulateur et d'un script Frida custom, j'ai travaillé sur un appareil Android physique et utilisé Objection pour désactiver le SSL Pinning en une commande. L'objectif final est le même — voir le trafic HTTPS de l'application en clair dans Burp.

---

## Environnement

- OS : Windows
- Appareil : Android physique (USB)
- Proxy : Burp Suite Community
- Package cible : `sg.vp.owasp_mobile.omtg_android`

---

## 1. Point de départ — interception bloquée

Sans configuration, toute tentative d'interception HTTPS échoue. L'appareil rejette le certificat de Burp et le trafic reste chiffré.

<img width="834" height="265" alt="image" src="https://github.com/user-attachments/assets/43a16650-6471-4893-a72d-9fc66c8dda61" />


---

## 2. Installation du certificat CA Burp

Depuis le navigateur de l'appareil, j'ai accédé à `http://burp` pour télécharger le certificat CA, puis je l'ai installé dans les paramètres Android sous **Sécurité → Certificats CA**.

<img width="369" height="227" alt="image" src="https://github.com/user-attachments/assets/ab496a1f-41df-4595-a5ca-9e8d8a36fff1" />

---

## 3. Validation du proxy

Pour confirmer que la configuration est correcte, j'ai ouvert le navigateur et navigué vers un site HTTPS. La requête apparaît bien dans Burp — le certificat CA est reconnu.

<img width="838" height="337" alt="image" src="https://github.com/user-attachments/assets/5bdc104f-fa26-4089-b242-deceacc1efc1" />


---

## 4. Identification du package cible

Après avoir installé l'APK via ADB, j'ai utilisé Frida pour localiser le package exact de l'application :

```powershell
frida-ps -Uai
```

```
sg.vp.owasp_mobile.omtg_android
```

![Package identifié](screens/08-frida-ps-package.png)

---

## 5. Bypass SSL Pinning avec Objection

Le certificat CA seul ne suffit pas — l'application implémente du SSL Pinning et rejette tout certificat autre que le sien. J'ai lancé Objection sur le package cible :

```powershell
objection -g sg.vp.owasp_mobile.omtg_android explore
```

Puis dans la console Objection :

```
android sslpinning disable
```

Objection injecte des hooks qui neutralisent les mécanismes de vérification TLS côté Java — `TrustManager`, `HostnameVerifier`, `OkHttp`, etc.

<img width="759" height="278" alt="image" src="https://github.com/user-attachments/assets/98d94b46-9600-47b2-bbd0-acab4c6adc1d" />


---

## 6. Résultat — trafic HTTPS intercepté

Avec le pinning désactivé, l'application accepte le certificat de Burp. Les requêtes HTTPS apparaissent dans l'onglet Proxy avec headers et contenu visibles en clair.

<img width="836" height="294" alt="image" src="https://github.com/user-attachments/assets/ee7ac19c-3f72-43a1-8042-046a88632483" />


---

## Conclusion

Ce lab montre la différence entre deux obstacles distincts : le certificat CA règle le problème au niveau du système, mais le SSL Pinning est un verrou applicatif supplémentaire. Objection neutralise ce second verrou par instrumentation dynamique sans toucher à l'APK. Sur un appareil physique, la démarche est identique à l'émulateur — seule la connexion USB et la configuration ADB changent.
