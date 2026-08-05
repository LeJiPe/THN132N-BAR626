# L'énigme Oregon : comment j'ai fini par percer le CRC des sondes THN132N/THN122

## Le problème

Avec des amis, nous possédons tous un récepteur météo Oregon Scientific **BAR626**. Le souci, c'est que ses sondes extérieures d'origine du constructeur — THN132N, THN122,  — tombent en panne au bout de quelques années, et sont aujourd'hui introuvables sur le marché. Même les fabricants chinois, pourtant coutumiers du fait, n'ont jamais réussi à les copier : aucune sonde du commerce actuelle n'est compatible avec ce récepteur.

Ce n'est pas faute d'avoir cherché du côté de la communauté : sur GitHub et ailleurs, plusieurs personnes butent depuis longtemps sur le calcul du CRC de ces trames. Des tables de relevés partielles existent, mais aucune ne s'est révélée compatible à 100 % avec le récepteur.

Je ne suis ni informaticien ni électronicien de formation. Mais j'avais encore une dernière sonde THN132N vivante, et je me suis dit qu'il devait bien y avoir moyen de comprendre ce qu'Oregon avait imaginé.

## Le robot "Enigma Oregon"

Pour faire parler cette dernière sonde survivante, j'ai construit un robot de test : des potentiomètres à la place des CTN, montés sur des servomoteurs pour balayer toutes les valeurs de température possibles, et des relais pour piloter les canaux.

Le robot a tourné pendant **3 jours** — cliquetis des relais, crissement des servomoteurs, au grand dam de ma femme — et m'a permis de collecter un très grand nombre de trames réelles, émises par une vraie sonde, avec leur CRC correct.

## La chasse au CRC

J'ai commencé par soumettre ce paquet de trames à plusieurs IA, en espérant qu'elles trouvent la formule. Sans succès : "pas assez de données", "calcul trop long", "il faut chercher dans telle direction"... des pistes, jamais de formule magique. Je n'étais visiblement pas le premier à leur poser la question.

J'ai donc changé d'approche. Avec toutes ces trames, j'ai pu isoler facilement les données qui faisaient varier le CRC. Autre constat : le CRC étant codé sur 8 bits, plusieurs trames différentes peuvent partager le même CRC. Plutôt que de chercher sur l'ensemble du jeu de données, j'ai isolé **3 groupes de trames donnant le même CRC** — ce qui réduisait considérablement le champ de recherche d'une formule capable de reproduire ce résultat.

J'ai alors demandé à une IA de m'écrire de petits programmes pour tester différents algorithmes candidats sur ces groupes. Et un jour, un programme s'arrête : une formule donne enfin un résultat identique pour les trois groupes. Le résultat brut n'était pas le bon CRC final, il restait une transformation à trouver — assez simple en fait : un **XOR par 0x59**, suivi d'une **inversion des quartets** (nibbles).

J'ai testé la formule complète sur plusieurs dizaines de trames. Ça marchait.

## La confirmation

Il fallait absolument valider ça sur du matériel réel. J'ai implémenté l'algorithme sur un Arduino, et je l'ai fait émettre vers mon BAR626.

Miracle : le récepteur se synchronise. Je pouvais désormais lui faire afficher n'importe quelle valeur de mon choix.

## Construire une vraie sonde de remplacement

Fort de cet algorithme, je me suis lancé dans la fabrication d'une sonde de remplacement à base d'un **PIC12F1822** (trouvé dans mes tiroirs) et d'un capteur **TMP117**, dans l'idée de faire quelque chose d'ultra-simple et au moins aussi précis que la sonde d'origine. Le programme a été écrit en assembleur, sous MPLAB.

N'ayant ni oscilloscope ni analyseur logique, j'ai dû ruser pour caler les timings : tous mes relevés de trames ont été faits avec une simple carte son, et j'ai déterminé les timings par superposition entre les trames générées et celles attendues par le récepteur. Le BAR626 s'est révélé très sensible à ces timings, y compris à l'intervalle très précis entre chaque émission — au point que j'ai fini par démonter le récepteur lui-même et enregistrer son activation en fonction de la perte de signal, pour connaître exactement ce qu'il attendait.

Pour le réveil périodique du PIC, j'ai utilisé l'oscillateur interne du watchdog. Résultat : la sonde fonctionnait parfaitement.

## Le piège

Content du résultat, j'ai voulu en construire une deuxième pour des amis. Et là, patatras : le timing basé sur le watchdog était complètement dans les choux sur ce deuxième exemplaire. Le système n'était tout simplement pas reproductible d'une puce à l'autre — la dispersion de fréquence de l'oscillateur interne du watchdog était trop grande d'un composant à l'autre.

Il a fallu tout reprendre, en repartant sur un microcontrôleur moderne disposant d'un vrai RTC, en réutilisant la base déjà développée sur Arduino. C'est ce qui a donné naissance au projet actuel, une sonde bâtie autour d'un ATtiny1616.

## Et maintenant

L'algorithme de CRC est validé, la synchronisation avec le BAR626 fonctionne, et la sonde actuelle est en cours de finalisation. Plusieurs pistes d'évolution restent ouvertes : un mécanisme de changement de rollcode — détection d'une coupure d'alimentation au démarrage dans une fenêtre de 4 à 12 secondes, répétée deux fois, puis addition d'un nombre premier au rollcode — et pourquoi pas une compensation thermique du quartz pour les utilisateurs en environnements très froids.

Si vous vous êtes déjà arraché les cheveux sur ces sondes, j'espère que ce récit — et surtout la formule de CRC — vous fera gagner beaucoup de temps.
