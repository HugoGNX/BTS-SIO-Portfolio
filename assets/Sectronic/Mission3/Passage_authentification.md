# Plan d'attaque - Intégration de l'authentification
## Application Congés - Intranet Sectronic

**Version :** 1.0  
**Statut :** En attente du système d'authentification intranet  
**Priorité globale :** Haute  


---

## Préambule

Ce document décrit l'ensemble des modifications à réaliser dans l'application `conges/`
lorsque le système d'authentification de l'intranet Sectronic sera disponible.

L'objectif est de passer d'une application **ouverte et sans contrôle d'accès** à une
application **sécurisée, isolée par utilisateur et par rôle**, sans changer la logique
métier existante.

Chaque étape est indépendante et numérotée. Elles doivent être réalisées **dans l'ordre**.

> ⚠️ **Avant de commencer :** S'assurer que le système d'authentification de l'intranet
> est opérationnel et que chaque employé possède un compte avec `first_name`, `last_name`
> et `email` correctement renseignés. ( Ou des champs correspondant ex: nom, prenom, email et adapter en fonction)

---

## Glossaire des rôles

| Rôle | Description |
|---|---|
| `employe` | Tout collaborateur qui soumet des demandes d'absence |
| `chef` | Chef de service, valide en premier |
| `secretariat` | Secrétariat, valide en second |
| `dg` | Direction Générale, validation finale |

---

## Vue d'ensemble des fichiers impactés

| Fichier | Impact |
|---|---|
| `models.py` | Ajout du lien vers l'utilisateur Django |
| `views.py` | Protection des vues + filtrage par utilisateur/rôle |
| `forms.py` | Suppression des champs nom/prénom/email (pré-remplis automatiquement) |
| `urls.py` | Aucune modification de structure, mais les vues seront protégées |
| `utils.py` | Aucune modification nécessaire |
| `templates/home.html` | Filtrer pour n'afficher que les demandes de l'utilisateur connecté |
| `templates/tableau_chef.html` | Filtrer pour n'afficher que les demandes de l'équipe du chef |
| `templates/tableau_secretariat.html` | Vérification du rôle secrétariat |
| `templates/tableau_dg.html` | Vérification du rôle DG |
| `templates/valider_chef.html` | Suppression des champs nom/prénom/email_secretariat |
| `templates/valider_secretariat.html` | Suppression des champs nom/prénom/email_dg |
| `templates/valider_dg.html` | Suppression des champs nom/prénom du validateur |
| `templates/soumettre_demande.html` | Suppression des champs nom/prénom/email/email_chef |
| `conges/templatetags/` | Création d'un templatetags `has_group` pour les templates |
| `settings.py` | Ajout de `LOGIN_URL`, `LOGIN_REDIRECT_URL`, `LOGOUT_REDIRECT_URL` |

---

## ÉTAPE 1 - Vérification de la hiérarchie

**Objectif :** Vérifier que les groupes existent déjà.
1. Les employés simples n'ont pas besoin d'être dans un groupe spécifique.
 L'accès à `home` et `soumettre_demande` est accordé à tout utilisateur
 authentifié (`@login_required` sans restriction de groupe).
2. Verifier que les groupes `chef`, `secretariat`, `dg` existent bien.
3. S'assurer que chaque utilisateur a bien `first_name`, `last_name` et `email`
correctement renseignés dans son profil.

> **Si ce n'est pas fait :**  
 Créer les groupes et affecter les utilisateurs via
l'interface d'administration.

Se connecter à `/admin/` avec un compte superutilisateur et effectuer les
actions suivantes :

1. Aller dans **Authentification et autorisation > Groupes**
2. Créer les trois groupes : `chef`, `secretariat`, `dg`
3. Aller dans **Authentification et autorisation > Utilisateurs**
4. Pour chaque utilisateur concerné, l'affecter au bon groupe dans
   l'onglet **Permissions** > **Groupes**

> Les employés simples n'ont pas besoin d'être dans un groupe spécifique.
> L'accès à la vue `home` et `soumettre_demande` est accordé à tout
> utilisateur authentifié (`@login_required` sans restriction de groupe).
---

## ÉTAPE 2 - Modifier le modèle `DemandeAbsence`

**Fichier :** `conges/models.py`

**Objectif :** Associer chaque demande à l'utilisateur Django qui l'a créée,
afin de pouvoir filtrer les demandes par utilisateur connecté.

**Modification 1 - Ajouter l'import en haut du fichier :**

```python
from django.conf import settings
```

**Modification 2 - Ajouter le champ dans la classe `DemandeAbsence`, dans la section
"Infos demandeur", juste avant le champ `nom` :**

```python
utilisateur = models.ForeignKey(
    settings.AUTH_USER_MODEL,
    on_delete=models.SET_NULL,
    null=True,
    blank=True,
    related_name='demandes_absence',
)
```

> **Pourquoi `SET_NULL` et pas `CASCADE` ?**  
> Si un compte utilisateur est supprimé, on conserve l'historique de ses
> demandes dans la base de données. Elles resteront visibles pour les
> administrateurs mais ne seront plus liées à un utilisateur actif.

**Après la modification, générer et appliquer la migration :**

```bash
python manage.py makemigrations conges
python manage.py migrate
```

---

## ÉTAPE 3 - Créer un templatetag `has_group`

**Objectif :** Permettre de vérifier le groupe d'un utilisateur directement
dans les templates Django pour adapter la navigation selon le rôle.

**Créer le dossier et les fichiers suivants :**

```
conges/                                       
└── templatetags/                            
    ├── __init__.py   (fichier vide)
    └── conges_tags.py
```

**Contenu de `conges_tags.py` :**

```python
from django import template

register = template.Library()

@register.filter
def has_group(user, group_name):
    """Vérifie si un utilisateur appartient à un groupe donné.
    
    Usage dans un template :
        {% if request.user|has_group:"chef" %}
    """
    return user.groups.filter(name=group_name).exists()
```

**Déclarer le templatetag dans `conges/apps.py` :**

Vérifier que le fichier `apps.py` contient bien :

```python
from django.apps import AppConfig

class CongesConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'conges'
```

**Charger le templatetag dans les templates qui en auront besoin :**

```django
{% load conges_tags %}
```

---

## ÉTAPE 4 - Modifier le formulaire `DemandeAbsenceForm`

**Fichier :** `conges/forms.py`

**Objectif :** Supprimer les champs que l'utilisateur saisit manuellement
aujourd'hui (nom, prénom, email, email_chef) car ils seront désormais remplis
automatiquement depuis le compte connecté et depuis la hiérarchie.

**Modification 1 - Supprimer les champs de la liste `fields` :**

Retirer `'nom'`, `'prenom'`, `'email'` et `'email_chef'` de la liste `fields`.
La liste doit devenir :

```python
fields = [
    'date_debut',
    'date_fin',
    'heure_debut',
    'heure_fin',
    'type_absence',
    'motif_autres',
    'date_recuperation',
    'justificatif',
]
```

**Modification 2 - Supprimer les widgets correspondants :**

Dans le dictionnaire `widgets`, supprimer les entrées `'nom'`, `'prenom'`,
`'email'` et `'email_chef'`.

**Modification 3 - Supprimer les labels correspondants :**

Dans le dictionnaire `labels`, supprimer les entrées `'nom'`, `'prenom'`,
`'email'` et `'email_chef'`.

**Modification 4 - Mettre à jour la méthode `clean()` :**

Supprimer ces quatre lignes dans `clean()` :

```python
nom = cleaned_data.get('nom')
prenom = cleaned_data.get('prenom')
email = cleaned_data.get('email')
email_chef = cleaned_data.get('email_chef')
```
Dans la méthode `clean()`, les variables `nom`, `prenom` , `email` et `email_chef` ne
seront plus extraites du formulaire.

Supprimer également le bloc de vérification de chevauchement (il sera
déplacé dans la vue à l'Étape 5) :

```python
# SUPPRIMER ce bloc entier :
if date_debut and date_fin:
    chevauchements = DemandeAbsence.objects.filter(
        nom=nom,
        ...
    )
    if chevauchements.exists():
        raise forms.ValidationError(...)
```

Supprimer également les blocs de mise en forme automatique du nom et prénom
puisqu'ils ne sont plus dans le formulaire :

```python
# SUPPRIMER ces lignes :
if nom:
    cleaned_data['nom'] = nom.upper()
if prenom:
    cleaned_data['prenom'] = prenom.capitalize()
```

---

## ÉTAPE 5 - Modifier les vues

**Fichier :** `conges/views.py`

Cette étape est la plus importante. Elle se découpe en plusieurs sous-étapes.

---

### ÉTAPE 5.1 - Importer les outils d'authentification

En haut du fichier `views.py`, ajouter ces imports :

```python
from django.contrib.auth.decorators import login_required, user_passes_test
from django.contrib.auth.models import Group
```

Définir des fonctions de test de rôle. Ces fonctions seront réutilisées
pour protéger les vues. Les ajouter juste après les imports, avant les vues :


```python
def est_chef(user):
    return user.groups.filter(name='chef').exists()

def est_secretariat(user):
    return user.groups.filter(name='secretariat').exists()

def est_dg(user):
    return user.groups.filter(name='dg').exists()
```

---

### ÉTAPE 5.2 - Vue `home`

**Objectif :** L'employé ne voit que ses propres demandes.

Ajouter le décorateur `@login_required` :

```python
@login_required
def home(request):
```

Modifier les trois requêtes pour filtrer par `utilisateur=request.user` :

```python
en_cours = DemandeAbsence.objects.filter(
    utilisateur=request.user,
    statut__in=['en_attente_chef', 'en_attente_secretariat', 'en_attente_dg']
)
approuve = DemandeAbsence.objects.filter(
    utilisateur=request.user,
    statut='approuve'
)
refuse = DemandeAbsence.objects.filter(
    utilisateur=request.user,
    statut='refuse'
)
```

---

### ÉTAPE 5.3 - Vue `soumettre_demande`

**Objectif :** Pré-remplir nom, prénom, email et email_chef depuis le compte
connecté et la hiérarchie. Ne plus les demander dans le formulaire.

Ajouter le décorateur `@login_required` :

```python
@login_required
def soumettre_demande(request):
```

Dans le bloc `if form.is_valid()`, avant `form.save()`, remplir les champs
depuis le compte utilisateur :

```python
demande = form.save(commit=False)
demande.nom = request.user.last_name.upper()
demande.prenom = request.user.first_name.capitalize()
demande.email = request.user.email
demande.utilisateur = request.user

# Récupération automatique de l'email du chef depuis la hiérarchie
# À adapter selon le modèle d'équipe disponible dans l'intranet
try:
    # Exemple si l'employé est rattaché à une équipe avec un responsable
    # equipe = Equipe.objects.get(operateurs=request.user)
    # chef = User.objects.get(username=equipe.RespHierarchique)
    # demande.email_chef = chef.email
    pass
except Exception:
    pass  # Laisser vide si non disponible, à compléter selon le modèle d'équipe

demande.save()
```

> **Prérequis :** Le système d'authentification doit stocker `last_name`,
> `first_name` et `email` dans le modèle utilisateur Django standard. Si
> l'intranet utilise un modèle personnalisé, adapter les noms de champs
> en conséquence.

Ajouter la vérification de chevauchement qui était dans le formulaire
(désormais déplacée ici, car on a accès à `request.user`) :

```python
chevauchements = DemandeAbsence.objects.filter(
    utilisateur=request.user,
    statut__in=['approuve', 'en_attente_chef', 'en_attente_secretariat', 'en_attente_dg'],
    date_debut__lte=demande.date_fin,
    date_fin__gte=demande.date_debut,
)
if chevauchements.exists():
    demande.delete()  # Annuler la sauvegarde
    messages.error(request, 'Vous avez déjà une demande en cours sur cette période.')
    return render(request, 'conges/soumettre_demande.html', {'form': form})
```

---

### ÉTAPE 5.4 - Vue `annuler_demande`

**Objectif :** Vérifier que l'utilisateur connecté est bien le propriétaire
de la demande avant de l'annuler.

Ajouter le décorateur `@login_required` :

```python
@login_required
def annuler_demande(request, pk):
```

Après `demande = get_object_or_404(DemandeAbsence, pk=pk)`, ajouter la vérification de la propriété :

```python
if demande.utilisateur != request.user:
    messages.error(request, "Vous n'êtes pas autorisé à annuler cette demande.")
    return redirect('conges:home')
```

---

### ÉTAPE 5.5 - Vues `tableau_chef` et `valider_chef`

**Objectif :** Accessible uniquement aux utilisateurs du groupe `chef`.

```python
@login_required
@user_passes_test(est_chef)
def tableau_chef(request):
    ...

@login_required
@user_passes_test(est_chef)
def valider_chef(request, pk):
    ...
```

Dans `valider_chef`, remplacer la récupération manuelle de `nom` et `prenom` :

```python
# AVANT (à supprimer)
nom = request.POST.get('nom', '').upper()
prenom = request.POST.get('prenom', '').capitalize()

# APRÈS (à mettre à la place)
nom = request.user.last_name.upper()
prenom = request.user.first_name.capitalize()
```

Remplacer aussi la récupération de l'email du secrétariat :

```python
# AVANT (à supprimer)
demande.email_secretariat = request.POST.get('email_secretariat', '')

# APRÈS — récupérer depuis le groupe secrétariat
try:
    groupe_secretariat = Group.objects.get(name='secretariat')
    premier_secretariat = groupe_secretariat.user_set.first()
    demande.email_secretariat = premier_secretariat.email if premier_secretariat else ''
except Group.DoesNotExist:
    demande.email_secretariat = ''
```

**Filtrage par équipe (à faire quand le modèle d'équipe sera disponible) :**

```python
# Quand le modèle d'équipe sera disponible, remplacer les requêtes par :
# employes_de_mon_equipe = User.objects.filter(responsable=request.user)
# en_cours = DemandeAbsence.objects.filter(
#     utilisateur__in=employes_de_mon_equipe,
#     statut='en_attente_chef'
# )
```

---

### ÉTAPE 5.6 - Vues `tableau_secretariat` et `valider_secretariat`

```python
@login_required
@user_passes_test(est_secretariat)
def tableau_secretariat(request):
    ...

@login_required
@user_passes_test(est_secretariat)
def valider_secretariat(request, pk):
    ...
```

Dans `valider_secretariat`, même principe :

```python
# Nom/prénom depuis le compte connecté
nom = request.user.last_name.upper()
prenom = request.user.first_name.capitalize()

# Email DG depuis le groupe
try:
    groupe_dg = Group.objects.get(name='dg')
    premier_dg = groupe_dg.user_set.first()
    demande.email_dg = premier_dg.email if premier_dg else ''
except Group.DoesNotExist:
    demande.email_dg = ''
```

---

### ÉTAPE 5.7 - Vues `tableau_dg` et `valider_dg`

```python
@login_required
@user_passes_test(est_dg)
def tableau_dg(request):
    ...

@login_required
@user_passes_test(est_dg)
def valider_dg(request, pk):
    ...
```

Dans `valider_dg` :

```python
# Nom/prénom depuis le compte connecté
nom = request.user.last_name.upper()
prenom = request.user.first_name.capitalize()
```

---

### ÉTAPE 5.8 - Vue `detail_demande`

**Objectif :** Vérifier que l'utilisateur a le droit de consulter cette demande.

```python
@login_required
def detail_demande(request, pk):
```

Après `demande = get_object_or_404(DemandeAbsence, pk=pk)`, ajouter la 
vérification d'accés.  Un employé ne peut consulter que ses propres demandes.
Un chef ne peut consulter que les demandes de son équipe. Le secrétariat ou 
une DG peut consulter toutes les demandes :

```python
peut_voir = (
    demande.utilisateur == request.user or
    est_chef(request.user) or
    est_secretariat(request.user) or
    est_dg(request.user)
)
if not peut_voir:
    messages.error(request, 'Accès non autorisé.')
    return redirect('conges:home')
```

---

## ÉTAPE 6 - Modifier les templates

---

### ÉTAPE 6.1 - Template `soumettre_demande.html`

**Supprimer** les champs nom, prénom, email et email_chef du formulaire.
Comme on utilise `{% bootstrap_form form %}`, ils disparaîtront automatiquement
puisqu'ils ont été retirés du formulaire à l'Étape 4.

**Mettre à jour le modal de confirmation** pour afficher le nom depuis la session
plutôt que depuis un champ de formulaire. Remplacer dans le JavaScript :

```javascript
// AVANT (à supprimer)
const nom = document.querySelector('#id_nom').value;
const prenom = document.querySelector('#id_prenom').value;
document.getElementById('modal-nom').textContent = nom;
document.getElementById('modal-prenom').textContent = prenom;

// APRÈS
// Le nom vient directement du template Django, pas d'un champ de formulaire
```

Et dans le HTML du modal, remplacer les spans dynamiques par :

```html
<p class="fs-5 text-muted">
    <strong>{{ request.user.get_full_name }}</strong>
</p>
```

---

### ÉTAPE 6.2 - Templates `valider_chef.html`, `valider_secretariat.html`, `valider_dg.html`

**Dans `valider_chef.html`**, supprimer les trois blocs de saisie manuelle :

```html
<!-- À supprimer -->
<div class="col-md-4">
    <label for="nom" ...>Votre nom</label>
    <input type="text" name="nom" ...>
</div>
<div class="col-md-4">
    <label for="prenom" ...>Votre prénom</label>
    <input type="text" name="prenom" ...>
</div>
<div class="col-md-4">
    <label for="email_secretariat" ...>Email du secrétariat</label>
    <input type="email" name="email_secretariat" ...>
</div>
```

**Dans `valider_secretariat.html`**, supprimer les champs nom, prénom et email_dg.

**Dans `valider_dg.html`**, supprimer les champs nom et prénom.

**Dans les trois templates**, ajouter à la place un affichage informatif :

```html
<div class="alert alert-light border mb-3">
    <i class="bi bi-person-check"></i>
    Vous validez en tant que : <strong>{{ request.user.get_full_name }}</strong>
</div>
```

---

### ÉTAPE 6.3 - Adapter la navigation dans `base.html`

**Fichier :** `base.html` (template de base de l'intranet, hors de `conges/`)

Charger le templatetag en haut du fichier :

```django
{% load conges_tags %}
```

Adapter le menu de navigation pour afficher uniquement les liens
correspondant au rôle de l'utilisateur connecté :

```django
{% if request.user.is_authenticated %}

    {# Tous les employés connectés voient le lien vers l'accueil congés #}
    <a href="{% url 'conges:home' %}">Mes congés</a>

    {% if request.user|has_group:"chef" %}
    <a href="{% url 'conges:tableau_chef' %}">Tableau Chef</a>
    {% endif %}

    {% if request.user|has_group:"secretariat" %}
    <a href="{% url 'conges:tableau_secretariat' %}">Tableau Secrétariat</a>
    {% endif %}

    {% if request.user|has_group:"dg" %}
    <a href="{% url 'conges:tableau_dg' %}">Tableau DG</a>
    {% endif %}

{% endif %}
```
> **Note :** Cela dépendra des groupes créer.

---

## ÉTAPE 7 - Modifier `settings.py`

**Fichier :** `settings.py` du projet

S'assurer que les paramètres suivants sont correctement définis
(adapter les URLs selon la configuration de l'intranet) :

```python
LOGIN_URL = '/login/'            # URL de la page de connexion de l'intranet
LOGIN_REDIRECT_URL = '/conges/'  # Redirection après connexion réussie
LOGOUT_REDIRECT_URL = '/login/'  # Redirection après déconnexion
```

---

## ÉTAPE 8 - Calendrier par équipe

**Fichier :** `conges/views.py`

**Objectif :** Le calendrier du chef n'affiche que les absences des membres
de son équipe, et non toutes les absences de l'entreprise.

Cette étape dépend de l'implémentation du modèle d'équipe dans l'intranet.
Elle ne peut être réalisée que lorsque le lien entre un employé et son
responsable sera disponible en base de données.

Dans `tableau_chef`, remplacer :

```python
# AVANT
absence_approuvees = DemandeAbsence.objects.filter(statut='approuve')

# APRÈS (adapter selon le modèle d'équipe disponible)
employes_de_mon_equipe = User.objects.filter(responsable=request.user)
absence_approuvees = DemandeAbsence.objects.filter(
    statut='approuve',
    utilisateur__in=employes_de_mon_equipe
)
```

---

## ÉTAPE 9 - Tests à réaliser

Après chaque étape, effectuer les tests suivants avant de passer à la
suivante.

| Test | Comportement attendu |
|---|---|
| Accéder à `/conges/` sans être connecté | Redirection vers la page de login |
| Accéder à `/conges/chef/` avec un compte employé | Redirection ou page 403 |
| Accéder à `/conges/chef/` avec un compte chef | Affichage du tableau de bord |
| Accéder à `/conges/secretariat/` avec un compte chef | Redirection ou page 403 |
| Soumettre une demande | La demande est associée à l'utilisateur connecté |
| Consulter `/conges/` avec deux comptes différents | Chaque compte ne voit que ses propres demandes |
| Tenter d'annuler la demande d'un autre employé via l'URL | Redirection avec message d'erreur |
| Valider en tant que chef | Le champ `chef_nom` est rempli depuis le compte, non saisi manuellement |
| Valider en tant que secrétariat | L'email de la DG est récupéré automatiquement depuis le groupe |
| Calendrier du chef | N'affiche que les absences de son équipe (après Étape 8) |
| Télécharger le PDF d'une demande qui n'appartient pas à l'utilisateur | Redirection avec message d'erreur |

---

## Récapitulatif des modifications par fichier

| Fichier | Modifications |
|---|---|
| `models.py` | +1 champ `utilisateur` (ForeignKey vers User) + migration |
| `forms.py` | Retrait de `nom`, `prenom`, `email`, `email_chef` des champs, widgets, labels et `clean()` + suppression du bloc chevauchement |
| `views.py` | `@login_required` sur toutes les vues + `@user_passes_test` sur chef/secretariat/dg + filtrage par `utilisateur` + suppression des saisies manuelles de nom/prénom/emails dans les validations + vérification de chevauchement déplacée depuis le formulaire |
| `soumettre_demande.html` | Suppression des champs nom/prénom/email/email_chef + mise à jour du modal |
| `valider_chef.html` | Suppression des inputs nom/prénom/email_secretariat + affichage du compte connecté |
| `valider_secretariat.html` | Suppression des inputs nom/prénom/email_dg + affichage du compte connecté |
| `valider_dg.html` | Suppression des inputs nom/prénom + affichage du compte connecté |
| `base.html` | Chargement de `conges_tags` + menu conditionnel selon le rôle |
| `conges/templatetags/conges_tags.py` | Création du filtre `has_group` |
| `settings.py` | `LOGIN_URL`, `LOGIN_REDIRECT_URL`, `LOGOUT_REDIRECT_URL` |
| Django Admin | Création des groupes `chef`, `secretariat`, `dg` et affectation des utilisateurs |

---

## Points d'attention particuliers

**Données existantes en base :**  
Les demandes créées avant l'activation de l'authentification auront
le champ `utilisateur` à `NULL` (d'où le `null=True` sur le ForeignKey).
Ces demandes resteront visibles uniquement pour les administrateurs.
Ne pas les supprimer sans validation du responsable.

**Ordre des validations dans le workflow :**  
La logique métier (chef → secrétariat → DG) ne change pas. Seule
la façon dont les données du validateur sont récupérées change
(compte connecté au lieu de saisie manuelle).

**Email du chef :**  
Actuellement saisi par l'employé dans le formulaire. Après l'authentification,
il sera récupéré automatiquement depuis le modèle d'équipe de l'intranet.
Cette partie dépend du modèle d'équipe disponible et peut nécessiter
une adaptation selon l'implémentation choisie.

**Emails vers secrétariat et DG :**  
Actuellement saisis dans les formulaires de validation. Après l'authentification,
ils seront récupérés depuis les groupes Django. Prévoir une période de transition
si les comptes ne sont pas tous créés le même jour.

**Calendrier des absences :**  
Le calendrier du chef devra être filtré pour n'afficher que les collaborateurs
de son équipe. Cette fonctionnalité dépend de l'implémentation du modèle
d'équipe dans l'intranet (Étape 8).

**`user_passes_test` et redirection :**  
Par défaut, `@user_passes_test` redirige vers la page de login si le test
échoue, même si l'utilisateur est connecté. Pour rediriger vers une page
d'erreur plus explicite, utiliser `raise_exception=True` :

```python
from django.core.exceptions import PermissionDenied

def est_chef(user):
    if user.groups.filter(name='chef').exists():
        return True
    raise PermissionDenied
```

---

*Document v1.0 - À vérifier avant de commencer les modifications.*