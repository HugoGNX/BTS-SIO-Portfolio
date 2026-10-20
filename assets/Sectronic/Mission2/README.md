# App visiteur

Application de gestion des visiteurs pour les sites Sectronic.

L'application remplace le registre papier d'accueil et permet l'enregistrement des visiteurs, la signature électronique des consignes de sécurité, le suivi des présences en temps réel ainsi que la génération de listes d'évacuation.

## Objectifs

* Remplacer le registre papier des visiteurs
* Permettre un enregistrement autonome sur tablette tactile
* Faire signer électroniquement les consignes de sécurité
* Suivre les visiteurs présents sur chaque site
* Gérer les départs des visiteurs
* Générer une liste d'évacuation en cas d'urgence
* Envoyer la liste d'évacuation aux responsables du site
* Isoler les visiteurs par site

## Architecture

L'application est développée avec Django et intégrée au portail intranet Sectronic.

Technologies utilisées :

* Python 3.12
* Django
* Bootstrap 5
* signature_pad.js pour la signature électronique
* qrcode pour la génération des QR codes
* API Kelio pour la récupération des salariés présents

## Stockage

Actuellement :

* SQLite en environnement de développement
* Migration SQL Server prévue

Modèles principaux :

### Site

Représente un site Sectronic.

| Champ | Type | Description |
| :--- | :--- | :--- |
| `code` | SlugField | Identifiant URL unique (ex: montaigu). Utilisé dans l'URL. |
| `nom` | CharField | Nom affiché (ex: Sectronic Montaigu) |
| `consignes_url` | URLField | URL du PDF des consignes hébergé sur un service externe |
| `emails_evacuation` | TextField | Emails séparés par virgules (obligatoire) |
| `responsable_nom` | CharField | Nom du responsable |
| `responsable_email` | EmailField | Email du responsable |

### Visiteur

Représente une visite.

| Champ | Type | Description |
| :--- | :--- | :--- |
| `site` | ForeignKey | Site d'accueil (PROTECT — suppression interdite si visiteurs liés) |
| `nom` | CharField | Nom du visiteur |
| `prenom` | CharField | Prénom du visiteur |
| `societe` | CharField | Société du visiteur |
| `hote` | CharField | Personne visitée |
| `heure_arrivee` | DateTimeField | Rempli automatiquement à la création |
| `heure_sortie` | DateTimeField | Null si encore présent |
| `sortie_auto` | BooleanField | True si marqué sorti par le job automatique |
| `signature` | TextField | Signature en base64 |

## Fonctionnement

### Arrivée d'un visiteur

1. Saisie des informations :

   * nom
   * prénom
   * société
   * personne visitée

2. Consultation des consignes de sécurité

3. Affichage d'un QR Code permettant de récupérer les consignes

4. Signature électronique

5. Validation de la visite

6. Apparition dans la liste des personnes présentes

### Arrivée de plusieurs visiteurs

Le mode groupe permet d'ajouter rapidement plusieurs visiteurs de la même société :

* conservation de la société
* conservation de l'hôte
* signature individuelle obligatoire

### Départ d'un visiteur

Le visiteur sélectionne :

* « Je pars »

L'heure de sortie est enregistrée et le visiteur disparaît de la liste des présents.

### Evacuation

La fonctionnalité d'évacuation est accessible depuis l'intranet Sectronic.

La page évacuation affiche :

- les visiteurs présents sur le site ;
- les salariés présents remontés par Kelio ;
- les informations nécessaires au contrôle des présences.

Un email peut être envoyé aux responsables du site avec la liste complète des personnes présentes.

## Multi-sites

Chaque tablette est associée à un site unique.

Exemple :

/visiteur/montaigu/

Les visiteurs sont isolés par site.

Un visiteur enregistré à Nantes n'apparaît jamais sur Montaigu.

## Routes principales

Les routes ci-dessous utilisent :

- `<code_site>` : identifiant du site
- `<id>` : identifiant du visiteur

| URL                                     | Description        |
| --------------------------------------- | ------------------ |
| /visiteur/`<code_site>`/                    | Accueil tablette   |
| /visiteur/`<code_site>`/nouveau/            | Création visiteur  |
| /visiteur/`<code_site>`/`<id>`/consignes/       | Consignes          |
| /visiteur/`<code_site>`/`<id>`/signature/       | Signature          |
| /visiteur/`<code_site>`/`<id>`/confirmation/    | Confirmation       |
| /visiteur/`<code_site>`/`<id>`/sortie/          | Départ             |
| /visiteur/`<code_site>`/evacuation/              | Liste d'évacuation |
| /visiteur/`<code_site>`/evacuation/envoyer/ | Envoi email        |
| /visiteur/historique/                   | Historique         |
| /visiteur/detail/id/                    | Détail visite      |

## Administration

Les nouveaux sites sont créés depuis l'administration Django.

Informations requises :

* code
* nom
* emails d'évacuation

Une fois créé, le site devient immédiatement disponible via :

/visiteur/code/

## Tests

Exécution :

```bash
python manage.py test visiteur
```

Tests couverts :

| Classe de test | Ce qu'elle vérifie |
| :--- | :--- |
| `SiteTestCase` | Création d'un site et de ses champs |
| `VisiteurTestCase` | Création d'un visiteur lié à un site |
| `SortieVueTestCase` | La vue sortie remplit bien le champ heure_sortie |
| `SortieAutoTestCase` | Le job daily marque sortie_auto=True |
| `TestIsolationSiteViaView` | Un visiteur de Nantes n'apparaît pas sur Montaigu |
| `EvacuationViewKelioMockTests` | Page évacuation avec Kelio simulé |
| `SignatureValidationTestCase` | Test de validation de la signature |

## Job automatique

Fichier :

core/jobs/daily/visiteur_auto_sortie.py

Chaque jour à 23h59 :

* fermeture automatique des visites non clôturées
* heure_sortie = 23:59
* sortie_auto = True

Exécution manuelle :

```bash
python manage.py runjobs daily --only visiteur_auto_sortie
```

## Déploiement

Les tablettes Android sont configurées en mode kiosque.

URL à utiliser :

```text
http://<serveur>/visiteur/<code_site>/
```

## Développeur

App Django :

```text
visiteur/
```

Templates :

```text
visiteur/templates/visiteur/
```

Fichiers statiques :

```text
visiteur/static/visiteur/
```

Tests :

```bash
python manage.py test visiteur
```
