# App Congés - TODO

## Gestion des rôles et authentification

### Problème

L'application fonctionne actuellement sans système d'authentification.

Toutes les informations sont saisies manuellement par les utilisateurs et aucune restriction d'accès n'est appliquée selon le rôle de la personne connectée.

### Conséquences

Actuellement :

* n'importe quel utilisateur peut accéder aux pages Employé, Chef de service, Secrétariat et DG ;
* aucune page n'est protégée ;
* aucune vérification ne permet de garantir que l'utilisateur possède réellement le rôle affiché ;
* les noms, emails et informations de validation peuvent être saisis librement ;
* les actions de validation ne sont pas associées à une identité vérifiée.

### Impact futur

Lorsque le système de connexion de l'intranet sera disponible, une refonte importante de l'application sera nécessaire :

* modèles Django ;
* routes ;
* vues ;
* formulaires ;
* templates ;
* règles de sécurité.

Chaque utilisateur devra uniquement accéder aux informations correspondant à son rôle.

### Priorité

Haute

---

## Isolation des données par utilisateur

### Problème

Toutes les demandes sont actuellement visibles par tous les utilisateurs.

### Conséquences

Un employé peut consulter :

* les demandes des autres employés ;
* les statuts ;
* les dates ;
* les informations de validation.

### Solution envisagée

Associer chaque demande à l'utilisateur connecté.

L'employé ne devra voir que :

* ses demandes en cours ;
* son historique ;
* les validations associées.

### Priorité

Haute

---

## Isolation des validations Chef de service

### Problème

Tous les chefs de service partagent actuellement le même tableau de validation.

### Conséquences

Un chef peut :

* consulter les demandes d'équipes qu'il ne supervise pas ;
* approuver ou refuser des demandes qui ne lui appartiennent pas.

### Solution envisagée

Associer chaque employé à son responsable hiérarchique.

Le tableau de bord chef devra uniquement afficher :

* les demandes de son équipe ;
* son historique de validation ;
* son planning d'équipe.

### Priorité

Haute

---

## Isolation des validations Secrétariat

### Problème

Toutes les demandes sont visibles dans un tableau unique.

### Solution envisagée

Restreindre les accès selon les droits attribués lors de l'authentification.

### Priorité

Moyenne

---

## Sécurisation des URLs

### Problème

Les URLs sont accessibles directement sans contrôle.

Exemples :

```text
/conges/chef/
/conges/secretariat/
/conges/dg/
```

### Conséquences

Toute personne connaissant l'URL peut accéder aux interfaces de validation.

### Solution envisagée

Mettre en place :

* authentification ;
* permissions ;
* contrôle des rôles ;
* protections Django (`login_required`, groupes, permissions).

### Priorité

Haute

---

## Calendrier des absences par responsable

### Problème

Le calendrier affiche actuellement l'ensemble des demandes.

### Solution envisagée

Après intégration des comptes utilisateurs :

* filtrer les événements selon le responsable ;
* afficher uniquement les collaborateurs rattachés au chef connecté.

### Priorité

Moyenne

---

## Notifications intranet

### État actuel

Les emails automatiques fonctionnent correctement.

### Reste à faire

Mettre en place les notifications directement dans l'intranet afin de compléter les emails.

### Priorité

Faible

---

##
