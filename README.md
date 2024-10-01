# TLSonLAN specification
Ecrit par Raphaël Roumezin, version 0.1

## Introduction
TLSonLAN est un modèle qui permet la sécurité d'équipement sur un même sous-réseau, malgré la non-fiabilité des connexions, et leur expositions à des attaques de type MITM (Man In The Middle). Il est typiquement destiné à des petites installations de réseau où les membres on besoin de communiquer de facon sécurisée, et sans avoir besoin d'échanger des fichiers de certificat entre les machines.

Cette méthode implique des vérifications par l'utilisateur en comparant deux chaînes de caractères, écrites sur l'appareil en train de rejoindre le réseau et l'appareil hébergant le serveur d'authorité (typiquement le routeur/NAT). Elle résulte en l'obtention par le client d'un certificat TLS, signé par l'authorité du réseau, et dont il possède la clé privée.

Ce document n'est qu'un concept que j'ai imaginé pour mon apprentissage. Je ne prétends pas avoir les compétences pour écrire une spécification correctement ou pour créer un modèle fonctionnel et fiable dans un environnement en production.

## Limitations
Ce protocole considère que les échanges en réseau ne sont pas fiables, et peuvent être écoutés et/ou réécrits, et que des machines puissent en usurper d'autres.
Accepter qu'un certificat TLS soit délivré à une machine par une authorité implique aussi que la machine fera confiance aux autres certificats émis par cette authorité. Pour contrer cette problématique, l'utilisateur doit vérifier manuellement que l'authorité qui délivre son certificat soit celle qu'il souhaite en comparant un court code généré à partir du certificat de l'authorité et celui délivré.

## Démarche
1. Le client ce connecte au réseau, et effectue une démarche DHCP, durant laquelle il a recu l'adresse IP du serveur d'authorité.
2. Le client se connecte au serveur d'authorité, en utilisant sa propre adresse IP et MAC. Cet échange est fait en TLS, en utilisant le certificat de l'authorité. Le client fera temporairement et exceptionellement confiance à n'importe quel certificat délivré à cette étape.
3. Le client génère une clé privée, puis un CSR, et envoie le CSR au serveur.
4. A partir de ce moment là, le client affichera une boite de dialogue similaire à celle-ci:
```
================================================================
 Ce réseau supporte le protocole TLSonLAN. Pour l'utiliser,
 comparer ce code avec celui affiché sur le routeur:

                     A1B2 C3D4 E5F6 A7B8

 Si le code correspond, appuyer sur VALIDER. Si il ne
 correspond pas ou si vous ne souhaitez pas utiliser le
 protocole TLSonLAN, appuyer sur ANNULER.

================================================================
|            ANNULER           |            VALIDER            |
================================================================
```
L'appareil du serveur d'authorité affichera lui aussi un message typique:
```
================================================================
 Un appareil souhaite rejoindre le réseau avec TLSonLAN.

                       IP : 10.1.2.3

                     A1B2 C3D4 E5F6 A7B8

================================================================
|            REFUSER           |            ACCEPTER           |
================================================================
```
5. Si l'un des deux pairs annule la demande, il le signifiera à l'autre, ce qui aura pour effet de fermer la connexion et de fermer la boite de dialogue ouverte.
6. Dès que le serveur d'authorité approuve la demande, il signera le CSR et enverra la chaîne de certificat complète au client. Celui-ci ne l'installera pas tant que l'utilisateur n'a pas validé la demande.

## Annonce de la fonctionnalité
S'il existe un serveur d'authorité sur le réseau, le DHCP doit le signaler en indiquant son adresse IP ainsi que le hash du certificat d'authorité, avec l'algorithme ci-dessous:
```
hash = SHA1(Certificat d'authorité)
```

## Renouvelement
Les certificats ont une période de validité courte. Avant que le certificat actuel expire, il est possible d'en demander un nouveau sans avoir à refaire les confirmations de code. Pour cela, il faut refaire la démarche de demande de certificat, mais en fournissant son certificat actuel comme certificat client (mTLS).

## Format du certificat
Le certificat est au format X.509, et certaines entrées doivent être remplies de la facon imposé par cette specification:
- Common Name: Le nom d'hôte du titulaire. Elle doit correspondre à celle fournie par le DNS local du réseau.
- IP address: L'adresse IP du titulaire.
- Expiration: Définie par l'authorité. Puisque les certificats sont renouvelables, un mois est la durée conseillée.

Le certificat d'authorité doit être auto-signé, et il n'y a pas de contrainte pour le Common Name. Il est recommandé d'utiliser le SSID du réseau WiFi, si la connexion L1 est en WiFi.

## Génération du code unique
Le code unique est dérivé du hash du certificat d'authorité et du CSR du client. Le hash s'obtient ainsi:
```
h = SHA256(SHA1(CSR client) + SHA1(Certificat d'authorité))
```
Noter que les fonctions `SHA1()` et `SHA256()` mentionnées ci dessous retourne le résultat de la fonction de hashage indiquée, sous forme d'un groupe d'octets (20 octets pour le SHA1 et 32 pour le SHA256) et non pas de leur forme hexadécimale en chaine de caractères.

Le hash fait 32 octets, et les 8 premiers octets sont affichés sous forme héxadécimale, et séparés par des espaces tout les 2 octets.

Par exemple, un pour un hash dont la forme hexadécimale est `5a81483d96b0bc15ad19af7f5a662e14b275729fbc05579b18513e7f550016b1`,
Le code unique est `5A81 483D 96B0 BC15`.

