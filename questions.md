# a. Contexte

## 2 Réflexion liminaire

2.1 Cohérence concurrente et synchronisation

Les problèmes de concurrence surviennent lorsque plusieurs clients modifient simultanément l'état partagé (canaux, messages), ce qui peut causer des incohérences (ex: un canal vide alors qu'il contient des utilisateurs). Pour éviter cela, le serveur utilise un verrou (threading.Lock) protégeant etat_serveur. Cependant, des erreurs réseau (déconnexions brutales) pourraient nécessiter un nettoyage supplémentaire pour maintenir la cohérence des données.

2.2 Modularité et séparation des responsabilités

Le serveur assure plusieurs responsabilités fonctionnelles : la gestion des connexions clients, le traitement des commandes, ainsi que la diffusion des messages. Il gère également les logs et la durabilité des données. La frontière logique sépare la gestion métier (canaux, utilisateurs) de la communication réseau, bien que certaines méthodes comme envoyer_message combinent les deux aspects. Enfin, la gestion des erreurs est centralisée dans IRCHandler(), permettant un traitement cohérent des commandes invalides et des retours clients.

2.3 Scalabilité et capacité à évoluer

L'ajout de nouvelles commandes, sont simplifié grâce à la structure actuelle : il suffit d'étendre la méthode handle() dans IRCHandler pour y intégrer la commande. Cependant, le système présente des limitations en termes de montée en charge. Le verrou global (etat_serveur["lock"]) peut être surchargé, et la gestion d'un thread par client ne permet pas de supporter au-delà de quelques centaines de connexions. Pour y remédier, des solutions comme les pools de threads ou une réécriture asynchrone pourraient être envisagées.

2.4 Portabilité de l’architecture

Pour adapter le système en HTTP, la gestion des canaux et utilisateurs pourrait être réutilisée, mais en remplaçant les sockets par un framework HTTP. Il faudrait alors implémenter des endpoints REST pour exposer les fonctionnalités. Plusieurs microservices pourraient être des candidats naturels : la gestion des utilisateurs, la messagerie (canaux et messages) et les logs. Cette architecture découplée permettrait une meilleure scalabilité. Les différents services communiqueraient entre eux via des API, tout en partageant une base de données commune pour assurer la cohérence des données. 

2.5 Fiabilité, tolérance aux erreurs, robustesse

Le serveur gère les déconnexions brutales en détectant les exceptions levées dans la méthode handle(). En cas d'échec d'envoi de message (à cause d’un socket cassé), le système intercepte ces erreurs. Cependant, il n'existe actuellement aucun mécanisme d'accusé réception pour garantir la bonne réception des messages. La seule trace fiable des activités provient des logs, qui enregistrent l'ensemble des événements mais n'offrent pas de solution de reprise pour les messages perdus. 

2.6 Protocole : structuration et évolutivité 

Le protocole suit une structure minimaliste non documentée avec des règles implicites : chaque ligne correspond à une commande /n, avec un préfixée /, et avec des arguments séparés par des espaces. Sa conception minimaliste le rend robuste : les commandes incomplètes comme l'envoi d’un message sans texte sont ignorées ou encore un nom de canal invalide est accepté sans validation. 
On pourrait imaginer une spécification formelle avec une grammaire ABNF (Augmented Backus-Naur Form) qui permettrait de documenter le protocole, d’éviter les ambiguïtés et de faciliter l'implémentation. Contrairement à REST/HTTP qui est basé sur requêtes/réponses, il est orienté ligne de commande, il maintient une connexion persistante et est basé sur des événements asynchrones. Cette approche est typique des protocoles temps-réel comme IRC, mais moins adaptée aux architectures web modernes.

---

# b. Analyse
## 2 Ce que vous devez faire maintenant 
### 2.1 Analyse du code

Qui traite les commandes ?

La méthode handle de la classe IRCHandler interprète les commandes comme /msg, /join, etc. Elle lit les commandes envoyées par le client via self.rfile.readline(), les décode, et les traite en fonction de leur préfixe. 
Les méthodes de IRCHandler (comme set_pseudo, rejoindre_canal, envoyer_message, etc.) accèdent à etat_serveur. L'accès est protégé par un verrou (etat_serveur["lock"]) pour éviter les conflits entre threads.

Où sont stockées les infos ?

Le canal courant d’un utilisateur est stocké dans etat_serveur["utilisateurs"][pseudo]["canal"]. 
Les flux de sortie associés à chaque client sont stockés dans etat_serveur["utilisateurs"][pseudo]["wfile"]. 

Qui peut planter ?

Si un client quitte sans envoyer /quit la méthode handle le détecte via if not ligne: break. Le pseudo est alors supprimé de etat_serveur["utilisateurs"] ainsi que du canal associé. 
Les échecs de write() sont capturés par try-except dans les méthodes. Le serveur continue de fonctionner même si un client ne reçoit pas un message. 
Les canaux vides ne sont pas supprimés automatiquement. Ils restent dans etat_serveur["canaux"] jusqu’à ce qu’un utilisateur les rejoigne ou qu’une logique de nettoyage soit ajoutée.

### 2.2 À produire

2.2.1 Un schéma d’architecture fonctionnelle
![Schema d'architecture](Schema.HEIC)

2.2.2 (en option pour ceux qui vont plus vite) Une fiche protocole

/nick <pseudo> : Attribue un pseudo unique à l’utilisateur
Exemple : <utilisateur> /nick Ginette12
Réponse : <serveur> Bienvenue, Ginette12 ! / <serveur> Pseudo déjà pris. 

/join <canal> : Rejoint un canal
Exemple : <utilisateur> /join canard
Réponse : <serveur> Canal canard rejoint. ou 

/msg <texte> : Envoie un message au canal courant
Exemple : <utilisateur> /msg Bonjour tout le monde, je m’appelle Ginette !
Réponse : <serveur> [canard] Ginette12: Bonjour tout le monde, je m’appelle Ginette !
/ <serveur> Vous n'avez pas rejoint de canal.

/read : Lit les messages 
Exemple : <utilisateur> /read 
Réponse : <serveur> Toutes les discussions sont en direct, pas de lecture différée

/ : Affiche les 10 dernières lignes du log 
Exemple : <utilisateur> /log 
Réponse : <serveur> Roger12 a rejoint le canal canard.  
Message de Roger12 sur #canard : Qui a des nœud papillon avec des canards comme moi ?
/ <serveur> Erreur lors de la lecture des logs

/alert <texte> : Envoie un message d’alerte
Exemple : <utilisateur> /alert Maintenance prévue à 15h
Réponse : <serveur> [ALERTE MANUELLE] Ginette12: Maintenance prévue à 15h
<serveur> Alerte envoyée. 
/ <serveur> Vous n'avez pas les droits pour envoyer une alerte.

/quit : Déconnexion propre
Exemple : <utilisateur> /quit 
Réponse : <serveur> Au revoir !

# c.Limites

Le serveur IRC actuel repose sur une architecture simple et rapide, fondée sur un protocole texte et une gestion des données en mémoire. Bien adaptée à des volumes modestes, cette solution montre rapidement ses limites face aux exigences modernes : montée en charge, modularité, testabilité, et tolérance aux pannes. Cette synthèse met en lumière les atouts de ce serveur, ses faiblesses structurelles, ainsi que les évolutions nécessaires pour migrer vers une architecture web ou microservices.

Pour commencer, nous allons aborder ce que le serveur fait bien. Le serveur IRC est conçu pour être léger et fonctionner entièrement en mémoire, ce qui garantit une latence très faible dans le traitement des messages et une réactivité élevée. La communication directe via les sockets TCP et les messages en texte clair est efficace tant que le nombre d’utilisateurs reste modéré. Il assure correctement les fonctions essentielles du protocole IRC : gestion des utilisateurs, des canaux, et transmission de messages. Sa conception centralisée permet de garantir une cohérence immédiate de l’état du système, tant qu’il reste sur une seule instance et sans exigences fortes en termes de fiabilité ou de persistance.

Par la suite, nous allons voir ce que le serveur cache ou gère mal. L’architecture actuelle, centrée sur une mémoire volatile, sans mécanisme de réplication ou de répartition de charge, ne permet pas de faire face à un nombre important d’utilisateurs ou à des défaillances réseau. La concurrence d’accès aux structures de données internes n’est pas gérée, ce qui peut entraîner des comportements incohérents ou des pertes de messages. En cas d’erreur réseau, d’échec d’écriture dans les journaux ou de déconnexion brutale, le serveur ne dispose d’aucune stratégie de tolérance aux pannes. Par ailleurs, les différentes responsabilités sont entremêlées dans les mêmes fonctions : lecture et écriture sur le socket, traitement métier, journalisation. Cette absence de séparation nuit à la maintenance, à la testabilité, et rend difficile toute adaptation vers d’autres protocoles comme HTTP ou WebSocket. Le couplage fort empêche aussi toute évolution vers une architecture modulaire. En outre, le protocole IRC repose sur une grammaire textuelle non formalisée, ce qui rend le parsing fragile, la détection d’erreurs peu fiable et l’ajout de nouvelles commandes difficile, voire risqué.

Enfin, nous allons parler de ce qu’il faudrait refactorer pour évoluer vers un système web ou microservices. Il est essentiel de séparer clairement les responsabilités entre les différentes couches : le parsing et la génération des messages (gestion du protocole), la logique métier (utilisateurs, canaux, règles de modération), la communication réseau (sockets, WebSocket, HTTP), et enfin le logging, idéalement structuré au format JSON. Ce découpage permettrait de tester chaque couche indépendamment et de faciliter l’intégration de nouveaux protocoles plus modernes, plus robustes, et plus interopérables. Par ailleurs, remplacer les fichiers JSON ou l’état en mémoire par une base de données (relationnelle ou NoSQL) offrirait une persistance robuste, une meilleure reprise après incident, et la possibilité de répartir la charge entre plusieurs instances. Pour gagner en scalabilité, certaines fonctions devraient être externalisées sous forme de microservices : authentification, modération, gestion de l’historique des messages. Ces services exposeraient des APIs (REST ou gRPC), permettant une communication claire et une architecture évolutive. Enfin, pour rendre l’ensemble devops-compatible, il serait nécessaire d’automatiser les tests, de centraliser les logs, de mettre en place un monitoring temps réel, d’utiliser la conteneurisation (par exemple avec Docker), et d’introduire une configuration dynamique.

En conclusion, le serveur IRC actuel est efficace pour des usages simples et un nombre limité d’utilisateurs, grâce à sa simplicité et son fonctionnement rapide en mémoire. Cependant, son couplage fort, sa gestion rudimentaire des erreurs, son protocole obsolète et son architecture monolithique freinent toute évolution vers des systèmes plus modernes. Pour rendre ce type de serveur compatible avec les exigences actuelles (scalabilité, fiabilité, modularité), il est indispensable de repenser profondément son architecture en adoptant une séparation claire des responsabilités, un protocole structuré, une persistance fiable, et une distribution des services via des APIs bien définies.
