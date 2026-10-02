📮 fabrich.site — Mail

📮

**fabrich.site**Documentation serveur mail

Comprendre

- [Vue d'ensemble](#vue-ensemble)
- [Comment voyage un email](#comment-ca-marche)
- [SPF, DKIM, DMARC](#spf-dkim-dmarc)
- [L'architecture choisie](#architecture)

Mise en place

- [1. Vérifications préalables](#etape-1)
- [2. DNS chez LWS](#etape-2)
- [3. Reverse DNS chez Contabo](#etape-3)
- [4. Installer Postfix](#etape-4)
- [5. Signer avec DKIM (Rspamd)](#etape-5)
- [6. L'adresse d'expéditeur](#etape-6)
- [7. Envoyer depuis Node.js](#etape-7)

Référence

- [Vérifier et tester](#verification)
- [Glossaire](#glossaire)
- [Et après ?](#et-apres)

vps.fabrich.site\
37.60.234.6 · Contabo\
Ubuntu 24.04 LTS

Mis en place le 2 octobre 2026

# Construire son propre serveur mail

Comment **vps.fabrich.site** envoie désormais des emails au nom de **fabrich.site** — signés, vérifiés par trois protocoles distincts, et acceptés sans broncher par Gmail et Outlook.

Domaine **fabrich.site**

Serveur **Contabo · 37.60.234.6**

Réception **toujours chez LWS**

SPF **✓ pass**

DKIM **✓ pass**

DMARC **✓ pass**

00 — Résumé

## Ce qu'on a construit, en une phrase

Le VPS envoie les emails, LWS continue de les recevoir, et les deux se portent garants l'un de l'autre via le DNS — sans qu'aucun des deux ne sache qu'il coopère avec l'autre, hormis à travers trois petites lignes de texte publiées dans la zone DNS.

C'est un point qui mérite d'être posé avant d'entrer dans les détails : on n'a **pas** déplacé la boîte `contact@fabrich.site`. Elle vit toujours chez LWS, exactement comme avant. Ce qu'on a ajouté, c'est une deuxième porte de sortie — le VPS — autorisée à envoyer des emails en se faisant passer, légitimement, pour `fabrich.site`.

Cette documentation reprend tout ce qu'on a fait, dans l'ordre, en expliquant à chaque étape *pourquoi* on l'a fait et pas seulement *comment*. L'objectif est que, dans six mois, en relisant ceci, tout refasse sens — y compris si tu dois dépanner quelque chose ou l'expliquer à quelqu'un d'autre.

01 — Les bases

## Comment voyage un email

Avant de parler de SPF ou de DKIM, il faut un modèle mental simple de ce qui se passe quand un email part. On va filer une analogie postale tout du long, parce que c'est littéralement le même métier.

✉️

Imagine que tu postes une lettre. Tu écris un expéditeur sur l'enveloppe — n'importe qui peut écrire n'importe quoi, rien ne vérifie que c'est vrai. Tu la déposes dans une boîte aux lettres, un facteur la récupère, et un bureau de tri l'achemine vers le bureau de poste du destinataire, qui la distribue.

Un email suit exactement ce chemin. **Postfix**, le logiciel qu'on a installé, joue le rôle du bureau de tri : il prend le message, regarde à qui il est adressé, trouve le bureau de poste responsable de ce domaine (le serveur **MX**), et le lui livre directement, de serveur à serveur, sans intermédiaire.

Le problème, historiquement, c'est que n'importe qui peut écrire n'importe quel expéditeur sur une enveloppe électronique. Rien dans le protocole SMTP de base n'empêche ton VPS d'envoyer un email en prétendant être `pdg@banquenationale.fr`. C'est exactement ce qui a fait du spam et du phishing un problème à l'échelle d'internet pendant 20 ans.

SPF, DKIM et DMARC sont les trois réponses qu'internet a fini par se donner à ce problème. Ce sont des informations publiées dans le **DNS** du domaine — l'annuaire public d'un nom de domaine — que n'importe quel serveur receveur (Gmail, Outlook, LWS…) va consulter avant de décider s'il fait confiance au message.

02 — Les trois protocoles

## SPF, DKIM, DMARC — en clair

Ce sont les trois enregistrements qu'on a ajoutés ou modifiés dans la zone DNS de `fabrich.site`. Chacun répond à une question différente, et c'est leur combinaison qui donne confiance aux grands fournisseurs de messagerie.

### SPF

Sender Policy Framework

« Voici la liste des facteurs autorisés à déposer du courrier en mon nom. »

SPF est une simple liste d'adresses IP ou de serveurs autorisés à envoyer des emails pour un domaine. Quand LWS reçoit un message prétendant venir de `fabrich.site`, il vérifie que l'IP qui le lui envoie figure bien dans cette liste. On a ajouté `37.60.234.6` à la liste existante :

TXT · fabrich.site

```
v=spf1 mx:fabrich.site a:mail.fabrich.site a:mailphp.lws-hosting.com ip4:37.60.234.6 -all
```

Le `-all` final est strict : il dit « si l'expéditeur n'est sur aucune de ces entrées, rejette-le ». Un domaine ne peut avoir qu'**un seul** enregistrement SPF — d'où l'importance de modifier l'existant plutôt que d'en créer un deuxième, qui l'aurait rendu invalide.

### DKIM

DomainKeys Identified Mail

« Voici un cachet de cire sur l'enveloppe, qui prouve qu'elle n'a pas été altérée en chemin. »

DKIM fonctionne avec une paire de clés cryptographiques. Le VPS garde une **clé privée** secrète et l'utilise pour signer mathématiquement chaque email sortant. N'importe qui peut vérifier cette signature avec la **clé publique** correspondante, qu'on a publiée en DNS. Si le contenu du message a été modifié d'un seul caractère en chemin, la signature ne correspond plus et la vérification échoue.

Sur le VPS, c'est **Rspamd** — déjà installé sur le serveur — qui signe chaque message au vol, avec le sélecteur `vps` (le nom qui distingue cette clé de celle que LWS utilise déjà, sous le sélecteur `dkim`, pour ses propres envois).

### DMARC

Domain-based Message Authentication

« Et si le cachet est cassé, ou si l'expéditeur n'est pas sur la liste des facteurs — voici la consigne que le bureau de poste doit suivre. »

SPF et DKIM peuvent chacun passer ou échouer indépendamment. DMARC est la règle qui dit quoi faire du résultat : accepter quand même, mettre en quarantaine (spam), ou rejeter purement et simplement. Il demande aussi, en option, un rapport quotidien récapitulant les mails qui passent et ceux qui échouent.

TXT · \_dmarc.fabrich.site

```
v=DMARC1; p=quarantine; rua=mailto:contact@fabrich.site
```

`p=quarantine` signifie : « si SPF et DKIM échouent tous les deux, mets le message en spam plutôt que de le rejeter net ». C'est une position prudente, un bon compromis tant qu'un domaine n'a pas plusieurs mois de recul sur sa configuration.

03 — La décision clé

## L'architecture qu'on a choisie

La vraie question de ce projet n'était pas technique mais structurelle : `fabrich.site` avait déjà une boîte mail active chez LWS. Il fallait ajouter une capacité d'envoi depuis le VPS **sans rien casser** de ce qui existait.

ENVOI SORTANT

VPS Contabo

vps.fabrich.site · 37.60.234.6\
Postfix + Rspamd (DKIM)

→

DESTINATAIRE

Gmail, Outlook…

Vérifie SPF, DKIM, DMARC avant d'accepter

EXPÉDITEUR

N'importe qui

Écrit à contact@fabrich.site

→

RÉCEPTION (inchangée)

LWS

mail.fabrich.site · MX en place depuis toujours

Deux flux indépendants, un seul domaine. Le MX ne bouge jamais.

On a donc délibérément **laissé le MX pointer vers LWS**. C'est la seule décision qui garantissait que `contact@fabrich.site` continuerait à recevoir son courrier sans interruption. Le VPS n'a besoin d'aucun enregistrement MX pour envoyer — MX ne sert qu'à la réception.

Pour que le VPS ait sa propre identité sans entrer en conflit avec celle de LWS, on lui a donné un nom distinct : `vps.fabrich.site`, plutôt que de réutiliser `mail.fabrich.site` qui était déjà pris. Même logique pour la signature DKIM : sélecteur `vps` pour le serveur, `dkim` restant celui de LWS. Les deux clés cohabitent sans jamais se gêner.

---

Étape 1

## Les vérifications préalables

Avant d'installer quoi que ce soit, on a vérifié que rien, côté hébergeur, n'allait bloquer le projet. Deux points étaient critiques.

1

#### La réputation de l'IP

Une IP de VPS peut hériter de la mauvaise réputation d'un ancien locataire. On a vérifié `37.60.234.6` sur MXToolbox contre une soixantaine de listes noires (blacklists). Résultat : propre partout, y compris sur **Spamhaus ZEN**, la plus déterminante pour Gmail et Outlook.

2

#### Le port 25 sortant

De nombreux hébergeurs bloquent ce port par défaut, précisément pour empêcher les VPS compromis d'envoyer du spam en masse. On l'a testé en IPv4 et IPv6 :

bash

```
# IPv6 — a réussi
nc -vz gmail-smtp-in.l.google.com 25

# IPv4 — a réussi aussi
nc -4 -vz gmail-smtp-in.l.google.com 25
```

Les deux sont passés. Contabo n'a pas bloqué le port, donc aucun relais SMTP tiers n'a été nécessaire.

**Pourquoi tester les deux protocoles ?** Un serveur peut privilégier IPv6 par défaut sans que personne ne s'en aperçoive. Or Gmail est particulièrement strict sur les mails reçus en IPv6 sans reverse DNS ni SPF correctement configurés sur cette adresse. On a tranché en forçant Postfix à n'utiliser que l'IPv4 (`inet_protocols = ipv4`), pour n'avoir qu'une seule adresse à faire reconnaître.

Étape 2

## La configuration DNS chez LWS

Le DNS est la pièce centrale de tout le projet : c'est l'annuaire public où tout le monde — Gmail compris — va vérifier les affirmations du VPS. Voici l'état final de la zone, avec ce qui a changé.

| Type | Nom | Valeur | Rôle |
| --- | --- | --- | --- |
| A | `vps` | `37.60.234.6` | Ajouté — identité du VPS |
| TXT | `@` | `v=spf1 … ip4:37.60.234.6 -all` | Modifié — autorise le VPS à envoyer |
| TXT | `vps._domainkey` | `v=DKIM1; k=rsa; p=MIGfMA0…` | Ajouté — clé publique DKIM du VPS |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; rua=…` | Modifié — politique + rapports |
| MX | `@` | `10 mail.fabrich.site` | **Inchangé** — LWS reste le receveur |

Deux pièges évités en chemin, qui valent la peine d'être notés. D'abord, **ne jamais créer un deuxième enregistrement SPF** : un domaine ne peut en avoir qu'un, et on a donc toujours édité la ligne existante plutôt que d'en ajouter une nouvelle. Ensuite, LWS répartit ses enregistrements TXT sur plusieurs pages de son interface, classées par type — un détail facile à manquer en cherchant le SPF au milieu des enregistrements A.

Étape 3

## Le reverse DNS chez Contabo

Le DNS classique répond à la question « à quelle IP correspond ce nom ? ». Le **reverse DNS** (ou enregistrement **PTR**) répond à l'inverse : « à quel nom correspond cette IP ? ». C'est une vérification que presque tous les grands serveurs mail exigent avant d'accepter un message : si l'IP qui t'écrit n'a pas de nom cohérent, ou pire, qu'aucun PTR n'existe du tout, le message est traité avec une grande suspicion, voire rejeté direct.

🏠

C'est l'équivalent d'un facteur qui se présenterait à ta porte en disant « je viens du bureau de poste central », mais dont le badge, vérifié auprès de la poste, indiquerait en réalité une camionnette banalisée. Le reverse DNS, c'est le badge qui doit correspondre à ce que le facteur prétend être.

Dans le panneau Contabo, rubrique *Reverse DNS management*, on a fait pointer l'enregistrement PTR de `37.60.234.6` vers `vps.fabrich.site` — en laissant volontairement l'entrée IPv6 inchangée, puisque Postfix n'envoie qu'en IPv4.

bash · vérification

```
dig -x 37.60.234.6 +short
# → vps.fabrich.site.
```

Il fallait que l'enregistrement A `vps.fabrich.site` soit déjà propagé côté LWS avant cette étape : Contabo vérifie la cohérence avant d'accepter le PTR, sans quoi la demande est rejetée.

Étape 4

## Installer et configurer Postfix

Postfix était déjà présent sur le VPS. Sa configuration par défaut n'était pas dangereuse, mais pas adaptée non plus : elle acceptait les connexions de n'importe où, et croyait gérer `fabrich.site` elle-même. Quatre réglages ont tout changé.

bash

```
postconf -e "myhostname = vps.fabrich.site"
postconf -e "myorigin = fabrich.site"
postconf -e "mydestination = localhost"
postconf -e "inet_interfaces = loopback-only"
postconf -e "inet_protocols = ipv4"
echo "fabrich.site" > /etc/mailname
systemctl restart postfix
```

### Pourquoi chacun de ces réglages compte

·

#### `myhostname` & `myorigin`

Identité du serveur : le nom qu'il annonce aux autres serveurs mail (`myhostname`, qui doit correspondre au reverse DNS) et le domaine qu'il met dans les adresses sortantes quand rien d'autre n'est précisé (`myorigin`).

·

#### `mydestination = localhost`

C'est le réglage le plus contre-intuitif, et le plus important. Par défaut, Postfix pense qu'il gère lui-même le courrier de `fabrich.site`. Sans ce changement, un email adressé à `contact@fabrich.site` depuis le VPS resterait bloqué localement au lieu de partir chercher le vrai serveur (LWS). En limitant `mydestination` à `localhost`, on force Postfix à toujours aller consulter le MX réel pour `fabrich.site`, exactement comme on le veut.

·

#### `inet_interfaces = loopback-only`

Le réglage de sécurité le plus important de toute l'installation. Il fait que Postfix n'accepte de connexions **que depuis le VPS lui-même** — jamais depuis l'extérieur. Résultat concret : le port 25 écoute toujours, mais seulement sur `127.0.0.1`. Personne sur internet ne peut s'en servir comme relais ouvert pour envoyer du spam en ton nom. C'est ce qui permet de se passer entièrement d'authentification SMTP : on fait confiance à quiconque peut déjà exécuter du code sur le serveur, pas à n'importe qui sur le réseau.

·

#### `inet_protocols = ipv4`

Simplifie tout en forçant un seul protocole réseau. Une seule adresse à faire reconnaître par SPF, une seule à configurer en reverse DNS, un seul chemin possible pour un mail sortant.

Étape 5

## Signer les emails avec DKIM, via Rspamd

Le plan initial prévoyait OpenDKIM, l'outil le plus répandu pour cette tâche. Mais en diagnostiquant le serveur, on a découvert que **Rspamd** tournait déjà dessus — un moteur antispam plus moderne qui sait aussi signer en DKIM nativement. Pas besoin d'un deuxième logiciel : on a branché celui qui était déjà là.

1

#### Générer la paire de clés

bash

```
mkdir -p /var/lib/rspamd/dkim
rspamadm dkim_keygen -s vps -d fabrich.site \
  -k /var/lib/rspamd/dkim/fabrich.site.vps.key \
  > /var/lib/rspamd/dkim/fabrich.site.vps.txt
```

`-s vps` fixe le sélecteur (le nom qui distingue cette clé des autres), `-d fabrich.site` le domaine signé. Le fichier `.key` est la clé privée, à garder strictement sur le serveur ; le `.txt` contient la clé publique à publier en DNS.

2

#### Dire à Rspamd où trouver la clé

/etc/rspamd/local.d/dkim_signing.conf

```
selector = "vps";
path = "/var/lib/rspamd/dkim/$domain.$selector.key";
allow_username_mismatch = true;
sign_authenticated = true;
sign_local = true;
use_domain = "header";
use_esld = true;
```

`sign_local = true` est la ligne qui a demandé le plus d'ajustement : par défaut, Rspamd ne signe que le courrier des utilisateurs authentifiés, pas celui envoyé localement par un script ou par `root`. Sans elle, les emails partaient bien… mais sans signature.

3

#### Brancher Postfix sur Rspamd via un milter

Un **milter** est un point d'accroche qui permet à un programme externe d'inspecter — et de modifier — un email pendant que Postfix le traite, avant qu'il ne reparte vers sa destination.

bash

```
postconf -e "milter_default_action = accept"
postconf -e "milter_protocol = 6"
postconf -e "smtpd_milters = inet:127.0.0.1:11332"
postconf -e "non_smtpd_milters = inet:127.0.0.1:11332"
systemctl restart postfix
```

Le port `11332` est le *worker proxy* de Rspamd, déjà actif sur le serveur. Chaque email passe désormais par lui avant de sortir, le temps d'y apposer la signature.

4

#### Publier la clé publique en DNS

Le fichier `fabrich.site.vps.txt` généré à l'étape 1 contient la clé, déjà formatée. Chez LWS, on a créé un enregistrement TXT `vps._domainkey` avec cette valeur, collée en un seul bloc.

**Le bug qu'on a chassé :** le premier essai a signé les mails (`DKIM-Signature` bien présent) mais la vérification côté LWS renvoyait `dkim=none`. La cause : `rspamadm dkim_keygen` avait nommé le fichier `vps.fabrich.site.key` (sélecteur d'abord), alors que la configuration attendait `fabrich.site.vps.key` (domaine d'abord, convention standard de Rspamd). Un simple renommage a suffi — un bon rappel que deux noms de fichiers presque identiques peuvent cacher un décalage total de logique.

Étape 6

## Choisir l'adresse d'expéditeur

Par défaut, les emails envoyés depuis le VPS portaient l'adresse de l'utilisateur système qui les déclenchait — `root@fabrich.site`, pas franchement présentable pour un vrai destinataire. On voulait `noreply@fabrich.site`, mais sans verrouiller cette adresse pour toujours : une appli doit pouvoir plus tard envoyer en `contact@` ou `support@` si besoin.

La solution retenue a deux niveaux, du plus fiable au plus flexible.

### Niveau 1

recommandé

Le code de l'application précise lui-même son `From:` à l'envoi. C'est la méthode la plus propre — chaque fonctionnalité choisit son adresse sans dépendre de la config serveur.

### Niveau 2

filet de sécurité

Si un script oublie de préciser un expéditeur et envoie en `root` ou `www-data` brut, Postfix le corrige automatiquement en `noreply@fabrich.site`.

/etc/postfix/generic

```
root noreply@fabrich.site
www-data noreply@fabrich.site
```

bash

```
postmap /etc/postfix/generic
postconf -e "smtp_generic_maps = hash:/etc/postfix/generic"
systemctl restart postfix
```

Le point important : cette table ne réécrit **que** les noms courts sans domaine (`root`, `www-data`). Un email déjà envoyé en `contact@fabrich.site` ou `support@fabrich.site` ne matche aucune ligne et repart tel quel. SPF et DKIM restent valides quelle que soit la partie locale choisie, puisqu'ils vérifient le **domaine**, pas l'adresse précise.

Étape 7

## Envoyer un email depuis une application Node.js

Dernière brique : brancher une vraie application. Avec **Nodemailer**, il suffit de parler au Postfix local, sans authentification puisque `loopback-only` fait déjà office de porte verrouillée.

bash

```
npm install nodemailer
```

send.js

```
const nodemailer = require("nodemailer");

const transporter = nodemailer.createTransport({
  host: "127.0.0.1",
  port: 25,
  secure: false,
  tls: { rejectUnauthorized: false }
});

async function main() {
  const info = await transporter.sendMail({
    from: '"Fabrich" <noreply@fabrich.site>',
    to: "contact@fabrich.site",
    subject: "Test Node.js",
    text: "Ceci est un test envoyé depuis Node via Postfix local."
  });
  console.log("Message envoyé :", info.messageId);
}

main().catch(console.error);
```

Le détail qui fait tout fonctionner : `host: "127.0.0.1"` et `port: 25`, sans identifiants. Depuis le VPS lui-même, Postfix accepte la connexion sans poser de question ; depuis l'extérieur, il la refuserait net. C'est exactement la frontière de confiance qu'on a posée à l'étape 4.

Si l'application tourne dans un conteneur Docker, `127.0.0.1` à l'intérieur du conteneur ne pointe plus vers le Postfix de l'hôte — il faut alors passer par `host.docker.internal` ou un réseau Docker partagé.

---

Référence

## Vérifier et tester

Trois façons de contrôler, à des niveaux de profondeur croissants, que tout fonctionne vraiment.

✓

Les logs Postfix

`tail -f /var/log/mail.log` — cherche `status=sent` après chaque envoi de test.

✓

La source du mail reçu

Dans Gmail : menu ⋮ → « Afficher l'original ». Cherche la ligne `Authentication-Results` — elle doit afficher `dkim=pass`, `spf=pass`, `dmarc=pass`.

✓

mail-tester.com

Envoie un mail à l'adresse qu'il fournit, puis clique sur « Then check your score ». Vise 9 ou 10 sur 10 — le rapport détaille précisément ce qui cloche en cas de problème.

### Ce qu'on a observé, en vrai

Authentication-Results · mail97.lwspanel.com

```
dkim=pass header.d=fabrich.site header.s=vps
spf=pass (… designates 37.60.234.6 as permitted sender)
dmarc=pass (policy=quarantine) header.from=fabrich.site
```

Les trois lignes au vert, confirmées par le propre serveur de LWS — la meilleure preuve possible, puisque c'est un tiers totalement indépendant du VPS qui a fait la vérification.

Référence

## Glossaire

MX (Mail Exchanger)

L'enregistrement DNS qui indique quel serveur est responsable de **recevoir** le courrier d'un domaine. Chez fabrich.site, il pointe toujours vers LWS.

Reverse DNS / PTR

L'inverse d'un enregistrement A : il fait correspondre une IP à un nom, plutôt qu'un nom à une IP. Quasiment obligatoire pour qu'un serveur mail soit pris au sérieux.

Milter

Une interface qui permet à un programme externe (ici Rspamd) d'inspecter ou de modifier un email pendant que Postfix le traite — utilisée ici pour y apposer la signature DKIM.

Relais ouvert (open relay)

Un serveur mail mal configuré qui accepte d'envoyer des messages pour n'importe qui, depuis n'importe où. La hantise de tout administrateur mail, et ce que `loopback-only` empêche ici.

Liste noire (blacklist / DNSBL)

Un registre public d'adresses IP signalées comme sources de spam. Les grands fournisseurs de messagerie les consultent avant d'accepter un email.

TTL (Time To Live)

La durée pendant laquelle un enregistrement DNS reste en cache avant d'être revérifié. Explique pourquoi un changement DNS n'est jamais instantané.

Sélecteur DKIM

Le nom qui distingue une clé DKIM d'une autre pour un même domaine, permettant à plusieurs serveurs (ici LWS et le VPS) de signer sous des clés différentes sans conflit.

Pour aller plus loin

## Et après ?

Le serveur actuel sait **envoyer**. Trois pistes si les besoins évoluent.

→

#### Recevoir aussi sur le VPS

Impossible sans toucher au MX actuel, qui doit rester chez LWS. La voie propre serait un sous-domaine dédié (par exemple `@vps.fabrich.site`) avec son propre MX, indépendant de `contact@fabrich.site`. Un outil comme **docker-mailserver** ou **Mailcow** deviendrait alors pertinent, pour gérer aussi l'IMAP et de vraies boîtes.

→

#### Surveiller la réputation dans la durée

Une IP neuve pour l'envoi de mail met plusieurs semaines à construire sa réputation, notamment auprès de Microsoft (Outlook/Hotmail), historiquement plus lent à faire confiance. Monter en volume progressivement plutôt que d'un coup aide beaucoup.

→

#### Lire les rapports DMARC

Grâce à `rua=mailto:contact@fabrich.site`, un rapport agrégé arrivera régulièrement, listant tous les serveurs qui ont envoyé en se faisant passer pour `fabrich.site`, et s'ils ont passé SPF/DKIM. Un bon moyen de détecter une usurpation ou un oubli de configuration.

Documentation générée à partir de la mise en place réelle du serveur mail de **fabrich.site** sur un VPS Contabo (Ubuntu 24.04), DNS géré chez LWS. Dernière vérification en date : SPF, DKIM et DMARC tous confirmés `pass` par le serveur de réception LWS.