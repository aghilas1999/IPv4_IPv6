# Projet réaliser a l'université d'aix marseille.
# 1. Configuration Réseau
On commence par considérer le réseau qui servira à déployer notre système.

## 1.1. Topologie et Adressage
![Capture d’écran du 2023-02-23 16-31-43](https://user-images.githubusercontent.com/59511179/220955044-2870d0c6-3634-4948-914c-db9ab0ef558c.png)
![Capture d’écran du 2023-02-23 16-32-34](https://user-images.githubusercontent.com/59511179/220955071-72e81131-3491-4c3f-9f84-4fb8d2a7ccfa.png)


Si l'on ne souhaite pas utiliser l'auto-configuration IPv6, on pourra utiliser fc00:1234:1::16 pour VM1-6 dans LAN1-6 et fc00:1234:2::36 pour VM3-6 dans LAN2-6.

Reprendre les fichiers Vagrantfile et ansible permettant de préparer votre système. Si vous n'aviez terminé la configuration au TP précédent, passez directement à la section suivante.

Vérifier que tout fonctionne bien en partant de VM neuves.

## 1.2. Un grand malheur !
La machine VM2-6 s'est arrêtée ! VM1-6 n'a donc plus accès directement à VM3-6.
![Capture d’écran du 2023-02-23 16-32-09](https://user-images.githubusercontent.com/59511179/220955113-efcca621-3d75-4260-8c5a-66092371138e.png)


Comment pourrait-on faire ?...

Nous allons relier nos deux îlots IPv6 via le réseau IPv4 en créant un tunnel IPv6 sur IPv4 entre VM1 et VM3 ...

Préalable au projet : avoir une collection de 5 VMs, configurées avec ansible, formant le réseau ci-dessus.

2.1. Création de l'interface
En vous inspirant du code exemple de la documentation (section 3), utiliser le code tunalloc.c qui crée une telle interface virtuelle pour créer une bibliothèque iftun qui a pour but de créer l'interface virtuelle, extrémité du tunnel. Le nom de l'interface pourra être défini à l'exécution. Prendre "tun0" pour les tests de cette partie.

NB l'interface n'est pas persistante par défaut et disparaît dès que le processus l'ayant créée termine : pensez à mettre votre programme en pause avant de configurer tun0.

## 2.2. Configuration de l'interface
Configurer l'interface tun0 avec l'adresse fc00:1234:ffff::1, mettre un masque adéquat. Ecrire un script configure-tun.sh reprenant vos commandes de configuration.
Routage : Suite à la disparition tragique de VM2-6, faut-il modifier les informations de routage sur VM1 ? ou sur VM1-6 ?
Faire un ping6 fc00:1234:ffff::1. Donner la capture sur tun0 (avec wireshark). Que constatez-vous ?
Faire ping6 fc00:1234:ffff::10. Que constatez-vous ?
Expliquez.
## 2.3. Récupération des paquets
A la création de tun0, un descripteur de fichier est renvoyé (cf l'exemple). C'est ce descripteur qui permet de lire ce qui est envoyé à l'interface par un appel read.

Compléter la bibliothèque iftun avec une fonction avec deux descripteurs de fichiers en paramètres src et dst, qui, étant donné un descripteur src correspondant à une interface TUN, recopie perpétuellement toutes les données lisibles sur src dans le fichier décrit par dst.
Tester avec dst=1 (sortie standard). Comme ce qui est recopié est du binaire, on filtrera la sortira du programme de test avec hexdump. Par exemple en faisant
VM# test_iftun tun0 | hexdump -C
Refaire ping6 fc00:1234:ffff::1 puis ping6 fc00:1234:ffff::10. Comparer et expliquer. Quel type de trafic voyez-vous? Refaire une capture avec wireshark dans le second cas et comparer avec ce qui est obtenu par votre programme test_iftun.
A quoi sert l'option IFF_NO_PI ? Que ce passe-t-il si vous ajoutez cette option lors de la création de l'interface ?
# 3. Un tunnel simple pour IPv6
Le but de cette partie est de créer un tunnel simple encapsulant un trafic IPv6 dans des paquets TCP/IPv4.



## 3.1. Redirection du trafic entrant
On utilisera par défaut le port 123.

Dans cette partie, on créera une bibliothèque extremite qui gérera le trafic entre extrémités du tunnel.

Ecrire une fonction ext-out qui crée un serveur écoutant sur le port 123, et redirige les données reçues sur la sortie standard.
Ecrire une fonction ext-in qui ouvre une connexion TCP avec l'autre extrémité du tunnel, puis lit le trafic provenant de tun0 et le retransmet dans la socket.
La commance nc (netcat) permet de transférer des données sur un port réseau. Pour le support IPv6 sur vos VMs, il faudra installer éventuellement netcat6. L'option -u permet d'envoyer en UDP (puisque votre tunnel est unidirectionnel pour l'instant).
VM$ nc -u fc00:1234:4::36 123 
Mettre en place une extrémité ext-in et une extrémité ext-out sur deux VMs distinctes.
Tester avec un "client" ping6 fc00:1234:ffff::10 pour injecter du trafic comme dans la partie précédente.
## 3.2. Redirection du trafic sortant
Compléter la fonction ext-out de la bibliothèque extremite pour créer une extrémité qui retransmet le trafic provenant de la socket TCP dans le tun0 local.
Que devrait-il se passer alors pour ce trafic ? Indication : Que se passe-t-il lorsque l'on écrit dans le fichier correspondant à (au descripteur de) tun0?
Vérifier avec l'un de vos paquets capturés dans le cas précédent. Pensez à le modifier éventuellement (avec un éditeur hexadécimal) pour que cela corresponde à tous vos paramètres réseau.
Proposer des tests de connectivité. Tester et vérifier !
Indication : La commande nc (netcat) permet également de transférer le contenu d'un fichier sur un port réseau

VM$ nc IPDEST 123 < fichier.bin
Indication : Que se passe-t-il si le paquet est modifié sans modifier le CRC? Comment pourrait-on calculer le nouveau CRC ?

## 3.3. Intégration Finale du Tunnel
Compléter la bibliothèque pour pouvoir avoir un flux bidirectionnel
Assurez-vous que les communications sont bien asynchrones.
## 3.4. Mise en place du tunnel entre VM1 et VM3 : Schémas
On veut utiliser VM1 et VM3 pour pouvoir faire un tunnel entre LAN3-6 et LAN4-6.

Compléter le schéma simplifié en expliquant en détail le parcours complet d'un paquet.
## 3.5. Mise en place du tunnel entre VM1 et VM3 : Système
Créer un exécutable tunnel64d qui, en s'appuyant sur la bibliothèque extremite crée un service offrant un tunnel TCP bidirectionnel.
Cet exécutable lira sa configuration dans un fichier de la forme suivante:
# interface tun
tun=tun0
# adresse locale 
inip=
inport=
options=
# adresse distante
outip=
outport=
Il pourra appeler le script de configuration réseau avec ces paramètres une fois que l'interface est créée. Il est également possible de faire des appels systèmes directs avec ioctl.
Tester en configurant et lançant tunnel64d sur deux VMs différentes.
Comparer le trafic direct et celui passant par le tunnel.
# 4. Validation Fonctionnelle
Il s'agit des tests minimaux à mettre en oeuvre pour valider votre tunnel.

## 4.1. Configuration
Sur VM1-6, VM1,VM2,VM3,VM3-6, afficher les paramètres réseaux significatifs avec ip addr et ip route.

## 4.2. Couche 3
Depuis VM1-6, effectuer ping6 fc00:1234:4::36. Proposer éventuellement d'autre tests au niveau "Réseau".

## 4.3. Couche 4
Depuis VM1-6, créer un fichier "msg.txt" puis l'envoyer sur le service echo de VM3-6.

VM1-6$ echo "Test" > msg.txt
VM1-6$ nc fc00:1234:4::36  VOTREPORTECHO < msg.txt
## 4.4. Couche 4 : bande passante
Afin de tester les performances réseaux, nous allons utiliser l'utilitaire iperf3, qui permet de tester les performances d'une connexion.

Ce banc d'essai fonctionne en mode client/serveur, lancer le serveur sur VM3-6

VM3-6$ iperf3 -s
Tester avec un petit, un moyen, un gros et un très gros tampon.

VM1-6$ iperf3 -6 -c fc00:1234:4::36 -n 1 -l 10
VM1-6$ iperf3 -6 -c fc00:1234:4::36 -n 1 -l 2K
VM1-6$ iperf3 -6 -c fc00:1234:4::36 -n 1 -l 128K
VM1-6$ iperf3 -6 -c fc00:1234:4::36 -n 1 -l 1M
# 5. Améliorations
Il est attendu que soient implémentées au moins une de ces améliorations.

## 5.1. Tests Avancés Couche 4
Proposer éventuellement d'autres tests en couche 4 afin de valider votre système.

Indication : En consultant notamment la documentation de iperf3, on pourra s'intéreser à l'évolution de la bande passante au cours du temps, à la gestion de la congestion, à la fragmentation, au MTU, à la gestion des TTL, etc...

## 5.2. Configuration ansible
Peut-on mettre en place la gestion du tunnel via ansible. Quelles seraient les difficultés ? Proposer une solution et la mettre en oeuvre.

## 5.3. Une amélioration simple
Il est possible d'ajouter en tête du paquet la taille de celui-ci sur deux octets avant de l'envoyer dans le tunnel. A la sortie du tunnel, cette information est utilisée pour attendre tous les octets avant de les envoyer à l'interface virtuelle.

Compléter la bibliothèque traitement en ajoutant des méthodes taillentete_in et taillentête_out permettant de mettre en place ce prétraitement.
La taille du paquet étant donné dans l'entête IP, écrire un traitement tailleauto qui n'ajoute pas d'entête mais tient compte de la taille par lecture du champ taille de l'entête du paquet encapsulé.
## 5.4. Fragmentation et Réassemblage
Au lieu de mettre en attente les données comme dans le cas précédent, on peut gérer explicitement la fragmentation au niveau des extrémités du tunnel.

Ajoute des méthodes frag_in et frag_out permettant de mettre en place un tel traitement.
## 5.5. Tunnels entre VMs
Comment faire pour réaliser un tunnel entre VMs situés sur des hôtes différents ?
Indication : Penser à utiliser la redirection de port de virtualbox.
Faire une démonstration de communication par un tel tunnel entre un serveur echov6 et un client echov6 situé sur des VMs qui n'hébergent pas tunnel64d.
## 5.6. Annonce de route IPv6
A l'établissement du tunnel, chaque extrémité lance le service radvd pour annoncer la route vers la plage IPv6 correspondant à l'autre extrémité.

# 6. FAQ
## 6.1. Comment ouvrir un shell en root ?
Utiliser sudo -s

## 6.2. Java, ce n'est pas la fête pour les appels système
Le code java est compilé vers une machine virtuelle JAVA, ce qui rend le code portable d'un système d'exploitation à l'autre. Mais si vous souhaitez faire des appels système, il faudra sans doute utiliser JNA, ou encore JNI pour accéder directement au système d'exploitation sous-jacent.

Notez également que dans tous les cas, vous pouvez faire les appels système en C, puis, par un appel exec(), passer la main (et surtout les descripteurs de fichiers) à un programme écrit dans un autre langage.

## 6.3. Les interfaces TUN n'apparaissent pas dans l'interface graphique
Il faudra utiliser ip link show pour les voir. Vous pouvez les configurer comme toute interface réseau.

## 6.4. Comment envoyer un seul paquet ICMP avec ping6 ?
Utiliser l'option -c : ping6 -c 1 123.45.67.89

## 6.5. Les paquets lus sur tun0 sont incomplets ?
Attention le paramètre MTU de votre interface (en général 1500) doit correspondre à votre propre traitement : si vous utilisez un tampon intermédiaire de taille inférieure au MTU, les données manquées seront perdues et vos paquets tronqués (c'est le même pĥénomène pour des sockets en mode datagramme).

Attention la MTU choisie doit être compatible avec IPv6.

## 6.6. Les commandes de configuration de tun0 échouent : ``RTNETLINK answers: Network is down``
Il est possible qu'une manipulation malencontreuse ait désactivé tun0. Vous pouvez le voir avec

$ ip addr show tun0
5: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 500
[...]
Pour remettre l'interface en fonctionnement, il faut faire

$ ip link set tun0 up
## 6.7. J'ai mis en place la redirection de ports sur virtualbox mais le ping/ping6 ne fonctionne pas ?
La redirection de port pour passer un NAT/PAT ne fonctionne que pour le protocole TCP.

Liens
Equipe DALGO
LIS
AMU
