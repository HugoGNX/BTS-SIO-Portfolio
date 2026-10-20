# App Congés

Application de gestion des demandes d'absence intégrée à l'intranet de l'entreprise.

L'application permet aux collaborateurs de soumettre des demandes d'absence et de suivre un processus de validation en plusieurs étapes impliquant :

* le Chef de service ;
* le Secrétariat ;
* la Direction Générale.

Une fois validée, la demande est approuvée définitivement, un document PDF est généré automatiquement et envoyé au demandeur par email.

---

# Objectifs

* Dématérialiser les demandes d'absence ;
* Centraliser le suivi des congés ;
* Assurer la traçabilité des validations ;
* Automatiser les notifications par email ;
* Réduire les échanges manuels ;
* Fournir un historique des demandes ;
* Permettre une validation hiérarchique en plusieurs étapes.

---

# Architecture

L'application est développée avec Django et intégrée au portail intranet.

Technologies utilisées :

* Python 3.x
* Django
* Bootstrap 5
* JavaScript
* FullCalendar
* Génération PDF
* Module interne `core.emailing`

Stockage actuel :

* SQLite en développement

Compatible avec une future migration SQL Server

---

# Modèle principal

## DemandeAbsence

Représente une demande d'absence ainsi que son cycle complet de validation.

### Informations du demandeur

| Champ    | Type       | Description                |
| -------- | ---------- | -------------------------- |
| `nom`    | CharField  | Nom du demandeur           |
| `prenom` | CharField  | Prénom du demandeur        |
| `email`  | EmailField | Adresse email du demandeur |

### Période d'absence

| Champ         | Type      | Description                     |
| ------------- | --------- | ------------------------------- |
| `date_debut`  | DateField | Date de début                   |
| `date_fin`    | DateField | Date de fin                     |
| `heure_debut` | TimeField | Heure de début (absence courte) |
| `heure_fin`   | TimeField | Heure de fin (absence courte)   |

### Types d'absence

| Valeur         | Libellé               |
| -------------- | --------------------- |
| `conges_payes` | Congés payés          |
| `rtt`          | RTT                   |
| `anciennete`   | Jour d'ancienneté     |
| `sans_solde`   | Sans solde            |
| `modulation`   | Modulation            |
| `souplesse`    | Compteur de souplesse |
| `a_recuperer`  | À récupérer           |
| `familial`     | Événement familial    |
| `autres`       | Autres                |

### Informations complémentaires

| Champ               | Type      | Description                            |
| ------------------- | --------- | -------------------------------------- |
| `motif_autres`      | TextField | Précision lorsque le type est "Autres" |
| `date_recuperation` | DateField | Date de récupération associée          |
| `justificatif`      | TextField | Justificatif d'événement familial      |

### Statuts

| Valeur                   | Libellé                    |
| ------------------------ | -------------------------- |
| `en_attente_chef`        | En attente Chef de service |
| `en_attente_secretariat` | En attente Secrétariat     |
| `en_attente_dg`          | En attente validation DG   |
| `approuve`               | Approuvé                   |
| `refuse`                 | Refusé                     |

### Suivi de la demande

| Champ             | Type          |
| ----------------- | ------------- |
| `statut`          | CharField     |
| `date_soumission` | DateTimeField |

### Validation Chef de service

| Champ              |
| ------------------ |
| `chef_nom`         |
| `chef_prenom`      |
| `chef_date`        |
| `chef_motif_refus` |
| `email_chef`       |

### Validation Secrétariat

| Champ                     |
| ------------------------- |
| `secretariat_nom`         |
| `secretariat_prenom`      |
| `secretariat_date`        |
| `secretariat_motif_refus` |
| `email_secretariat`       |

### Validation Direction Générale

| Champ            |
| ---------------- |
| `dg_nom`         |
| `dg_prenom`      |
| `dg_date`        |
| `dg_motif_refus` |
| `email_dg`       |

---

# Fonctionnement

## Soumission d'une demande

L'employé renseigne :

* nom ;
* prénom ;
* email ;
* période d'absence ;
* type d'absence ;
* informations complémentaires éventuelles.

Après validation :

* création de la demande ;
* statut = En attente Chef de service ;
* envoi automatique d'un email au chef de service.

---

## Validation Chef de service

Le responsable consulte la demande et peut :

* approuver ;
* refuser.

En cas de refus :

* saisie obligatoire d'un motif ;
* statut = Refusé ;
* notification au demandeur.

En cas d'approbation :

* statut = En attente Secrétariat ;
* notification au secrétariat.

---

## Validation Secrétariat

Le secrétariat vérifie la demande.

Actions possibles :

* valider ;
* refuser.

En cas de refus :

* saisie obligatoire d'un motif ;
* statut = Refusé ;
* notification au demandeur avec coupon PDF de refus en pièce jointe..

En cas de validation :

* statut = En attente DG ;
* notification à la Direction Générale.

---

## Validation Direction Générale

La DG effectue la validation finale.

Actions possibles :

* approuver définitivement ;
* refuser.

En cas de refus :

* saisie obligatoire d'un motif ;
* statut = Refusé ;
* notification au demandeur.

En cas d'approbation :

* statut = Approuvé ;
* génération automatique du PDF d'autorisation ;
* envoi du PDF au salarié ;
* archivage de la validation.

---

# Workflow de validation

```text
Employé
   |
   v
En attente Chef de service
   |
   +---- Refus ----> Refusé + PDF refus envoyé
   |
   v
En attente Secrétariat
   |
   +---- Refus ----> Refusé + PDF refus envoyé
   |
   v
En attente DG
   |
   +---- Refus ----> Refusé + PDF refus envoyé
   |
   v
Approuvé
   |
   +---- Génération PDF autorisation
   |
   +---- Notification email + PDF en pièce jointe
```

---

# Notifications

Les notifications sont gérées par le module interne :

```python
core.emailing
```

Emails envoyés automatiquement :

* nouvelle demande au chef de service ;
* validation chef vers secrétariat ;
* validation secrétariat vers DG ;
* approbation finale vers le salarié avec PDF d'autorisation en pièce jointe;
* refus vers le salarié avec PDF de refus en pièce jointe.

---

# Génération PDF

Deux types de coupons PDF sont générés automatiquement :

**Coupon d'autorisation** (demande approuvée) :


* généré lors de la validation finale par la DG ;
* envoyé automatiquement par email au salarié ;
* téléchargeable depuis la page de détail de la demande.


**Coupon de refus** (demande refusée) :


* généré lors d'un refus à n'importe quelle étape ;
* adapté automatiquement selon le rôle du validateur (Chef de service, Secrétariat ou DG) ;
* envoyé automatiquement par email au salarié ;
* téléchargeable depuis la page de détail de la demande.

---

# Calcul de la durée d'absence
Un fichier `utils.py` centralise le calcul de la durée d'une absence. La fonction 
`calculer_duree_absence` gère deux cas:
* absence sur une seule journée avec horaires renseignés: calcul en heures et minutes;
* absence sur plusieurs jours: calcul en jours ouvrés (lundi au vendredi, week-ends exclus).

La durée est affichée dans les pages de détail et dans les interfaces de validation.

---

# Sécurisation des formulaires

Afin d'éviter les soumissions multiples accidentelles, une protection a été mise en place
côté JavaScript. Lors du clic sur le bouton de confirmation, le formulaire est soumis et les
interactions utilisateur sont immédiatement bloquées via la propriété `pointerEvents`. Cette
solution empêche efficacement les doubles clics et les envois multiples de notifications par
email.

---

# Calendrier des absences

L'application utilise FullCalendar afin d'afficher les absences validées sous forme de
calendrier interactif accessible depuis les tableaux de bord du Chef de service, du 
Secrétariat et de la Direction Générale.

Les données sont transmises ai JavaScript via une balise `<script
type="application/json">` afin d'éviter les problèmes d'échappement des caractères
spéciaux.

Cette fonctionnalité pourra être enrichie lorsque les rôles utilisateurs seront mis en place
afin de filtrer automatiquement les collaborateurs visibles selon le responsable connecté.

---

# Routes principales

Les routes ci-dessous utilisent :

* `<pk>` : identifiant d'une demande d'absence

| URL                         | Description                        |
| --------------------------- | ---------------------------------- |
| `/conges/`                  | Accueil                            |
| `/conges/soumettre/`        | Création d'une demande             |
| `/conges/annuler/<pk>/`     | Annulation d'une demande           |
| `/conges/detail/<pk>/`      | Détail d'une demande               |
| `/conges/chef/`             | Tableau de bord Chef de service    |
| `/conges/chef/<pk>/`        | Validation Chef de service         |
| `/conges/secretariat/`      | Tableau de bord Secrétariat        |
| `/conges/secretariat/<pk>/` | Validation Secrétariat             |
| `/conges/dg/`               | Tableau de bord Direction Générale |
| `/conges/dg/<pk>/`          | Validation Direction Générale      |

---

# Fonctionnalités actuellement disponibles

## Employé

* création d'une demande avec modal de confirmation;
* consultation du détail ;
* suivi du statut ;
* annulation d'une demande en cours (statut En attente Chef uniquement);
* voir les demandes aprouvées;
* voir les demandes refusées avec motif;
* téléchagement du coupon PDF (autorisation ou refus).


## Chef de service

* consultation des demandes en attente ;
* validation ou refus avec motif;
* vois les absences refusées;
* voir les absences validées et leur état d'avancement;
* planning de toutes les absences approuvées (FullCalendar);
* consultation du détail d"une demande;
* visibilité sur les demandes en cours et futurs de l'employé concerné.

## Secrétariat

* contrôle administratif ;
* validation ou refus avec motif;
* voir les absences refusées;
* voir les absences validées et leur état d'avancement;
* planning de toutes les absences approuvées (FullCalendar);
* consultation du détail d"une demande;
* visibilité sur les autres demandes en cours et futures de l'employé concerné.

## Direction Générale

* validation finale ou refus;
* voir les absences refusées;
* voir les absences validées;
* planning de toutes les absences approuvées (FullCalendar);
* consultation du détail d'une demande;
* visibilité sur les autres demandes en cours et futures de l'employé concerné;
* génération et envoi du PDF d'autorisation.

---

# Limitations actuelles

L'application fonctionne actuellement sans système d'authentification.

Par conséquent :

* les informations sont saisies manuellement ;
* les rôles ne sont pas vérifiés ;
* les tableaux de bord ne sont pas isolés par utilisateur ;
* les accès aux URLs ne sont pas protégés.

Ces limitations seront corrigées lors de l'intégration du système de connexion de l'intranet.

---

# Fonctionnalités prévues

* authentification intranet ;
* gestion des rôles ;
* restriction des accès ;
* notifications intranet ;
* filtrage du planning par équipe pour le chef de service ;
* statistiques et indicateurs.

---

# Développeur

Application Django :

```text
conges/
```

Templates :

```text
conges/templates/conges/
```

Fichiers statiques :

```text
conges/static/conges/
```

Tests :

```bash
python manage.py test conges
```
