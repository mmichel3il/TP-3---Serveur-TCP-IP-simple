# Fiche de protocole – IRC CanaDuck

Complétez cette fiche pour décrire comment les clients interagissent avec le serveur IRC actuel.

## Format général

- Chaque commande est envoyée par le client sous forme d’une **ligne texte** terminée par `\n`.

## Commandes supportées

| **Commande**  | **Syntaxe exacte**        | **Effet attendu**                 | **Réponse du serveur**         | **Responsable côté serveur**  |
|---------------|---------------------------|-----------------------------------|--------------------------------|-------------------------------|
| `/nick`       | `/nick <pseudo>`          | Attribue un pseudo unique         | Message de bienvenue ou erreur | `set_pseudo()`                |
| `/join`       | `/join <canal>`           | Rejoint ou crée un canal          | Confirmation ou erreur         | `rejoindre_canal()`                              |
| `/msg`        | `/msg <texte>`            | Envoie un message au canal courant| Message diffusé aux membres    | `envoyer_messages()`                              |
| `/read`       | `/read `                  | Lit les messages                  | Message indiquant le direct    | `lire_messages()`                              |
| `/log`        | `/log `                   | Affiche les 10 derniers logs      | Logs ou erreur                 | `lire_logs()`                             |
| `/alert`      | `/alert <texte>`          | Envoie un message d'alerte (admin)| Alerte diffusée ou erreur      | `envoyer_alerte()`                              |
| `/quit`       | `/quit `                  | Déconnexion propre                | Message d'au revoir            | `handle()`                             |

## Exemples d’interactions

### Exemple 1 : choix du pseudo

```
Client > /nick ginette
Serveur > Bienvenue, ginette !
```

### Exemple 2 :
```

Client > /join general
Serveur > Canal general rejoint.

```
...

# Structure interne – Qui fait quoi ?

| Élément                      | Rôle dans l’architecture                            |
|------------------------------|-----------------------------------------------------|
| `IRCHandler.handle()`        | Lit et traite les lignes de commande                |
| `etat_serveur`               | Stocke l'état des utilisateurs et des canaux        |
| `log()`                      | Enregistre les événements dans un fichier           |
| `broadcast_system_message()` | Difffuse un message système à tous les utilisateurs |
| `changer_etat()`             | Charge l'état depuis le fichier JSON                |
| `sauvegarder_etat()`         | Sauvegarde l'état dans un fichier JSON              |

# Points de défaillance potentiels

> Complétez cette section à partir de votre lecture du code.

| **Zone fragile**                 | **Cause possible**             | **Conséquence attendue**         | **Présence de gestion d’erreur ?**  |
|----------------------------------|--------------------------------|----------------------------------|-------------------------------------|
| `wfile.write(...)`               | Connexion interrompue          | Message non délivré              | Oui, try et except                  |
| Modification d’`etat_serveur`    | Accès concurrent               | Incohérence des données          | Oui, verrou threading.Lock()        |
| Lecture du fichier log           | Fichier inexistant ou corrompu | Erreur de lecture                | Oui, try et except                  |
| Pseudo déjà pris (`/nick`)       | Pseudo dupliqué                | Message d'erreur                 | Oui, vérification explicite         |
| Utilisateur sans canal courant   | Commande /msg sans canal       | Message d'erreur                 | Oui, vérification explicite         |
|                                  | Role insuffisant               | Message d'erreur                 | Oui, vérification des roles         |

# Remarques ou cas particuliers

- Les commandes sont traitées **en texte brut**, sans structure formelle.
- Une mauvaise commande renvoie un message générique (`Commande inconnue.`).
- L'état des canaux est sauvegardé mais pas celui des utilisateurs connectés
- Seuls les utilisateurs avec rôle "admin" ou "moderator" peuvent envoyer des alertes
- Le système de messages est en temps réel, la commande /read ne fait rien d'utile
- Les logs affichent uniquement les 10 dernières lignes