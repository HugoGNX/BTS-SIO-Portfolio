# TODO — App Visiteur

> Objectif : pointage visiteurs sur tablette Android (Chrome) à l'accueil de chaque site

---

## Contexte et objectif

Aujourd'hui, un **registre papier** à l'accueil sert à noter les visiteurs. C'est peu fiable : les visiteurs oublient souvent de remplir, l'écriture est parfois illisible, on ne sait pas en temps réel qui est présent, et il faut feuilleter le cahier pour reconstituer la liste en cas d'urgence. L'app `visiteur` remplace ce registre papier par une saisie sur tablette.

**Sectronic a plusieurs sites** (bâtiments / implantations), chacun avec son propre accueil et sa propre tablette. En cas d'évacuation, ce qui compte c'est qui est **dans le bâtiment concerné** — pas qui est sur l'autre site à 500 m. L'app est donc **multi-sites dès la v1** : un seul backend, plusieurs tablettes, chaque tablette est dédiée à un site.

On ne sait donc pas précisément qui sont les visiteurs présents dans nos locaux à un instant T, ni qui ils sont venus voir. En cas d'évacuation (incendie notamment), c'est un point critique : le responsable évacuation doit pouvoir consulter rapidement la liste des personnes dans le bâtiment — salariés (via Kelio) **et** visiteurs — **filtrée par site**.

L'app `visiteur` répond à ce besoin :

- À l'arrivée, chaque visiteur s'enregistre lui-même sur la tablette de **son site** (saisie tactile, sans login). La tablette est dédiée à un site : l'URL qu'elle ouvre encode déjà le site, le visiteur n'a pas à le choisir.
- Plusieurs personnes peuvent arriver ensemble (groupe d'une même société qui vient voir la même personne).
- Le visiteur lit les consignes de sécurité spécifiques à ce site (circulation, ESD, point de rassemblement) et **signe électroniquement** pour accepter.
- À leur départ, ils repassent à la tablette et cliquent sur leur nom dans la liste des présents pour pointer leur sortie.
- En cas d'oubli, un job automatique les marque sortis à 23h59 le jour même.
- Un bouton « Évacuation » dans l'intranet envoie par mail la liste consolidée (visiteurs présents **sur ce site** + salariés Kelio **sur ce site**) à la liste de destinataires propre à ce site.

À terme : savoir en permanence qui est dans nos locaux **et sur quel site**, avec preuve d'acceptation des consignes de sécurité.

---

## User stories (le « quoi », pas le « comment »)

### US1 — Enregistrement à l'arrivée

> En tant que **visiteur** arrivant à l'accueil d'un site, je veux pouvoir m'enregistrer en quelques tapotements sur la tablette pour ne pas avoir à attendre un salarié, signaler ma présence à la personne que je viens voir, et accepter les consignes de sécurité du site.

- Saisie : nom, prénom, société, personne visitée (champ libre).
- Le **site** est déterminé automatiquement par la tablette (encodé dans l'URL ouverte en mode kiosque).
- Affichage des consignes de sécurité (texte + pictogrammes) **propres au site**.
- Signature électronique (canvas tactile) — obligatoire pour valider.
- Si plusieurs visiteurs arrivent ensemble : possibilité d'ajouter rapidement plusieurs lignes dans la foulée sans re-taper société/hôte (mais chaque personne signe individuellement).
- Validation : retour visuel « Bienvenue [Prénom], merci de patienter à l'accueil ».

### US2 — Consultation des présents (sur ce site)

> En tant que **visiteur partant** ou **salarié curieux**, je veux voir la liste des visiteurs actuellement présents **sur ce site**, pour pointer ma sortie ou savoir qui est dans les locaux.

- Page d'accueil de la tablette = liste des visiteurs présents **du site de la tablette uniquement**.
- Affichage carte/tuile par visiteur : Nom Prénom, Société, Hôte, Heure d'arrivée.
- Tri par heure d'arrivée décroissante (les derniers arrivés en haut, plus visibles).

### US3 — Pointage de sortie

> En tant que **visiteur partant**, je veux pouvoir cliquer sur ma tuile dans la liste des présents pour signaler mon départ.

- Tap sur la tuile → confirmation (« Confirmez le départ de [Prénom Nom] »).
- Après confirmation : la tuile disparaît de la liste des présents, l'heure de sortie est enregistrée.
- Anti-fausse-manipulation : si plusieurs visiteurs présents avec le même prénom/nom, afficher la société pour différencier.

### US4 — Sortie automatique à 23h59

> En tant qu'**administrateur**, je veux que les visiteurs qui ont oublié de pointer leur sortie soient automatiquement marqués sortis en fin de journée, pour que la liste reparte propre le lendemain.

- Job quotidien à 23h59 qui marque comme sortis **tous les visiteurs encore présents du jour**, tous sites confondus.
- Une trace doit indiquer que c'est une sortie automatique (pour distinguer un vrai départ d'un oubli).

### US5 — Bouton Évacuation (par site)

> En tant que **responsable sécurité d'un site** (ou n'importe qui en cas d'urgence), je veux pouvoir générer instantanément la liste de toutes les personnes présentes dans **ce site**.

- Bouton bien visible « Liste évacuation » sur la page principale de la tablette.
- Action : envoie un mail à la liste de destinataires **du site** contenant :
  - La liste des visiteurs actuellement présents sur ce site (nom, société, hôte, heure d'arrivée).
  - La liste des salariés pointés sur Kelio comme étant sur ce site à cet instant.
- L'envoi doit fonctionner même si Kelio est injoignable (afficher « Kelio indisponible » et envoyer quand même les visiteurs).

---

## Étape 0 — Préparation (avant de coder une ligne)

### 0.1 — Installer l'environnement de dev

1. Cloner le repo `sectronic-portail`.
2. Créer un venv Python 3.12, installer `requirements.txt`.
3. Copier `.env.template` → `.env` et remplir les variables nécessaires.
4. Lancer `python manage.py migrate` puis `python manage.py runserver 0.0.0.0:8000`.
5. Ouvrir [http://localhost:8000](http://localhost:8000) — tu dois voir l'intranet.

✅ **Milestone 0.1** : tu peux naviguer dans l'intranet en local sur ta machine.

### 0.2 — Lire les bases de Django

Avant d'écrire du code, lis dans l'ordre (compte 1 à 2 jours, prends des notes) :

1. [Django Girls tutorial](https://tutorial.djangogirls.org/fr/) — tutoriel pas-à-pas, idéal débutant.
2. [Tutoriel officiel Django partie 1 à 4](https://docs.djangoproject.com/fr/5.0/intro/tutorial01/) — couvre projet/app/vues/modèles/admin.
3. **Le fichier [CLAUDE.md](CLAUDE.md) à la racine du projet** — c'est la convention maison. Lis surtout les sections « Frontend Conventions » et « Base Templates ».
4. **Relations entre modèles Django** : [doc ForeignKey](https://docs.djangoproject.com/fr/5.0/topics/db/examples/many_to_one/) — tu vas en avoir besoin pour relier `Visiteur` à `Site`.

### 0.3 — Explorer une app existante

Ouvre l'app [archives_papier/](archives_papier/) — c'est une app simple, bien rangée, qui te servira de modèle structurel. Lis dans l'ordre :

1. `archives_papier/models.py`
2. `archives_papier/views.py`
3. `archives_papier/urls.py`
4. `archives_papier/templates/archives_papier/*.html`

Comprends comment ces 4 fichiers communiquent. Si quelque chose n'est pas clair, **note la question** et viens en parler !

✅ **Milestone 0.2** : tu sais expliquer comment l'app `archives_papier` fonctionne, de la requête HTTP au HTML rendu.

---

## Étape 1 — Créer l'app Django

```bash
python manage.py startapp visiteur
```

Cette commande crée un dossier `visiteur/` avec plusieurs fichiers (`models.py`, `views.py`, `apps.py`, etc.). Ouvre-le et observe ce qui a été créé.

Ensuite, déclare l'app dans Django :

- Ajoute `"visiteur"` dans `INSTALLED_APPS` (fichier [sectronicportail/settings.py](sectronicportail/settings.py)).
- Lance le serveur (`python manage.py runserver`). Il ne doit pas planter.

✅ **Milestone 1** : l'app `visiteur` est déclarée, le serveur démarre sans erreur.

---

## Étape 2 — Concevoir les modèles (sur papier d'abord)

Il y a **deux modèles** à concevoir : `Site` et `Visiteur` (avec une relation entre les deux).

**Ne code rien tout de suite.** Prends une feuille (ou un fichier `.txt`) et réponds à ces questions :

### Modèle `Site`

1. Qu'est-ce qu'un site dans notre contexte (combien on en a, qu'est-ce qui les différencie) ?
2. Quels champs caractérisent un site ? Pense à :
   - Un identifiant court utilisable dans une URL (genre `montaigu`, `nantes`) → c'est un `SlugField`.
   - Un nom affichable (« Sectronic Montaigu »).
   - L'URL externe du PDF des consignes (peut différer d'un site à l'autre : zones ESD différentes, plan de rassemblement différent).
   - La liste des mails à prévenir en cas d'évacuation (texte multi-lignes ou champ séparé par virgules).
   - Optionnel : nom du responsable site, adresse, etc.

### Modèle `Visiteur`

1. Qu'est-ce qu'on stocke dans la base pour un visiteur ?
2. Quels champs sont obligatoires ? Lesquels sont optionnels ?
3. Quel est le type de chaque champ (texte court, texte long, date+heure, booléen, image…) ?
4. Comment sait-on si un visiteur est encore présent ou déjà parti ?
5. Comment sait-on si un visiteur a été marqué sorti automatiquement à 23h59 ou s'il a pointé lui-même ?
6. Comment on stocke la signature électronique (c'est une image, dessinée au doigt) ?
7. **Comment on relie le visiteur à son site ?** → réponse : `ForeignKey(Site, on_delete=models.PROTECT)`.

À la fin de cet exercice, tu dois avoir :
- Le modèle `Site` avec **4 à 6 champs**.
- Le modèle `Visiteur` avec **8 à 10 champs** (dont la FK vers `Site`).

---

## Étape 3 — Écrire les modèles en Python

Une fois la conception validée, ouvre `visiteur/models.py` et écris les deux classes : `Site(models.Model)` puis `Visiteur(models.Model)`.

Doc à lire avant :
- [Django Models field reference](https://docs.djangoproject.com/fr/5.0/ref/models/fields/)
- [ForeignKey](https://docs.djangoproject.com/fr/5.0/ref/models/fields/#foreignkey)

**Points d'attention pour notre projet** :

- Lire la section « Multi-Database Routing » de [CLAUDE.md](CLAUDE.md) : par défaut on met les modèles sur **SQL Server**. Mais **on va commencer sur SQLite (la base par défaut)** — c'est plus simple à manipuler en local et tu n'auras pas à te connecter au SQL Server de prod. On migrera plus tard.
- Ne crée **pas** d'attribut `_use_db` dans `Meta` pour l'instant. Sans cet attribut, le modèle ira automatiquement sur la base `default` (SQLite).
- Définis une méthode `__str__(self)` sur les deux modèles. Pour `Site` : `self.nom`. Pour `Visiteur` : `f"{prenom} {nom} ({societe})"`. C'est ce qui s'affiche dans l'admin.
- Pour la signature, on stocke en `TextField` (le base64 produit par le canvas est mis tel quel en base). C'est plus simple qu'un `ImageField` pour démarrer : pas de gestion `media/`, pas de `Pillow`, et la signature s'affiche directement avec `<img src="data:image/png;base64,{{ visiteur.signature }}">`. Les signatures pèsent ~10-50 Ko, c'est acceptable en base.
- Sur le `ForeignKey` vers `Site` : utilise `on_delete=models.PROTECT` — interdit de supprimer un site qui a des visiteurs (sinon on perd l'historique).

---

## Étape 4 — Créer la migration

```bash
python manage.py makemigrations visiteur
python manage.py migrate
```

- `makemigrations` génère un fichier Python dans `visiteur/migrations/` qui décrit comment créer les tables.
- `migrate` exécute cette migration : les tables `visiteur_site` et `visiteur_visiteur` sont créées.

Ouvre le fichier de migration généré et lis-le : tu dois comprendre ce qu'il fait (création de deux tables, dont une avec une FK vers l'autre). Si tu ne comprends pas, demande.

✅ **Milestone 4** : les tables `visiteur_site` et `visiteur_visiteur` existent dans la base. Pour vérifier :

```bash
python manage.py dbshell
.schema visiteur_site
.schema visiteur_visiteur
.quit
```

---

## Étape 5 — Enregistrer les modèles dans l'admin Django

1. Ouvre `visiteur/admin.py` et enregistre les deux modèles (`Site` et `Visiteur`).
2. `python manage.py runserver`, va sur [http://localhost:8000/admin/](http://localhost:8000/admin/), connecte-toi (il faut d'abord créer un utilisateur avec `createsuperuser`).
3. **Crée au moins 2 sites de test** (par exemple « Montaigu » et « Nantes ») — il faut les créer **avant** de créer des visiteurs, puisque le visiteur a une FK obligatoire vers un site.
4. Crée ensuite 3-4 visiteurs de test (la signature peut rester vide pour ces tests admin), en répartissant entre les deux sites.

✅ **Milestone 5** : tu peux créer / modifier / supprimer des sites et des visiteurs via l'admin Django.

---

## Étape 6 — Première vue et URL avec site

L'objectif : afficher la liste des visiteurs présents **d'un site donné** sur [http://localhost:8000/visiteur/montaigu/](http://localhost:8000/visiteur/montaigu/).

À faire dans l'ordre :

1. Créer `visiteur/urls.py`. La route principale doit prendre le code du site en paramètre : `path("<slug:site_code>/", views.index, name="index")`.
2. Inclure ces URLs dans [sectronicportail/urls.py](sectronicportail/urls.py) avec `path("visiteur/", include("visiteur.urls", namespace="visiteur"))`.
3. Écrire la vue `index(request, site_code)` dans `visiteur/views.py` :
   - Récupérer le `Site` correspondant via `get_object_or_404(Site, code=site_code)`.
   - Récupérer les visiteurs présents **de ce site uniquement** : `Visiteur.objects.filter(site=site, heure_sortie__isnull=True)`.
   - Passer le site et la liste au template.
4. Créer le template `visiteur/templates/visiteur/index.html` : il étend `base.html` (on passera à `base_light.html` plus tard) et affiche :
   - Le nom du site en haut (« Accueil — Sectronic Montaigu »).
   - Un tableau ou des cartes avec les visiteurs présents.

Doc à lire : [Tutoriel Django partie 3 — vues et templates](https://docs.djangoproject.com/fr/5.0/intro/tutorial03/) et [URL converters](https://docs.djangoproject.com/fr/5.0/topics/http/urls/#path-converters).

✅ **Milestone 6** : sur `/visiteur/montaigu/` tu vois la liste des visiteurs de Montaigu, et sur `/visiteur/nantes/` celle de Nantes. Aucun croisement entre les deux.

---

## Étape 7 — Formulaire d'enregistrement (sans signature pour l'instant)

L'objectif : un visiteur peut s'enregistrer via un formulaire HTML standard, sur la tablette de son site.

1. Lire la doc [Django Forms](https://docs.djangoproject.com/fr/5.0/topics/forms/) — partie « ModelForm » surtout.
2. Créer `visiteur/forms.py` avec une `VisiteurForm(forms.ModelForm)` qui expose les champs saisis par l'utilisateur (nom, prénom, société, hôte). **Pas** `site` (sera rempli automatiquement depuis l'URL), pas `heure_arrivee` (auto), pas `heure_sortie`, pas `signature` (étape suivante).
3. Créer une vue `nouveau(request, site_code)` dans `views.py` :
   - Récupère le `Site` depuis l'URL.
   - Sur GET : affiche le formulaire vide.
   - Sur POST : valide le formulaire, **attribue le site avant de sauvegarder** (`visiteur = form.save(commit=False); visiteur.site = site; visiteur.save()`), redirige vers la page d'accueil de ce site.
4. Créer le template `visiteur/templates/visiteur/nouveau.html`. Utilise `{% bootstrap_form form %}` (cf. autres apps pour l'exemple).
5. Ajouter la route `path("<slug:site_code>/nouveau/", views.nouveau, name="nouveau")` dans `visiteur/urls.py`.
6. Sur la page d'accueil, ajouter un gros bouton « S'enregistrer » qui pointe vers cette nouvelle URL en passant le site_code.

✅ **Milestone 7** : un visiteur peut s'enregistrer via le formulaire (sans signature). Il est bien associé au site de la tablette et apparaît dans la liste des présents de ce site.

---

## Étape 8 — Page consignes de sécurité + QR code PDF (par site)

Entre le formulaire d'identification et la signature, le visiteur doit lire les consignes de sécurité **de son site**. Il peut aussi flasher un QR code pour récupérer un PDF des consignes sur son téléphone.

**Contrainte importante** : l'intranet n'est pas accessible depuis Internet. Le téléphone du visiteur est sur son réseau mobile (4G/5G), donc il ne peut pas joindre notre serveur interne. Le PDF doit donc être **hébergé sur un service externe** (partage Sectronic). Notre app ne fait que **générer le QR code à partir de l'URL externe stockée sur le `Site`**.

### 8.1 — URL des consignes : portée par le modèle `Site`

Chaque site a son propre PDF de consignes (zones ESD différentes, plan d'évacuation différent…). L'URL est donc un **champ du modèle `Site`** (que tu as déjà ajouté à l'étape 2), pas une variable globale `.env`.

- Pour Montaigu : `https://www.sectronic.fr/consignes-montaigu.pdf`
- Pour Nantes : `https://www.sectronic.fr/consignes-nantes.pdf`

Renseigne ces valeurs dans l'admin Django sur chaque site.

### 8.2 — Page consignes

1. Créer un template `visiteur/templates/visiteur/consignes.html` qui affiche :
   - Le nom du site.
   - Le texte des consignes (résumé lisible sans scroll excessif sur la tablette).
   - Un **QR code** à côté/en-dessous, qui pointe vers `site.consignes_url`.
   - Le pictogramme « scannez pour emporter les consignes sur votre téléphone ».
2. Génération du QR code : librairie Python **[qrcode](https://pypi.org/project/qrcode/)** (ajouter à `requirements.txt`).
   - Génération côté serveur dans la vue `consignes(request, site_code, pk)` : créer l'image QR à partir de `visiteur.site.consignes_url`, l'encoder en base64, la passer au template.
   - Dans le template, afficher avec `<img src="data:image/png;base64,{{ qr_code_b64 }}" alt="QR code consignes">`.
   - Pas besoin de la pré-générer ni de la mettre en cache : c'est rapide (< 50 ms).
3. Demander le **texte exact** des consignes affichées pour chaque site et idéalement les **pictogrammes**.
4. Adapter le workflow : après validation du formulaire d'identification (étape 7), au lieu de rediriger vers l'accueil, rediriger vers `/visiteur/<site_code>/<id>/consignes/`.
5. La page consignes affiche le texte + le QR code + un bouton « J'ai lu et j'accepte → Signer » qui pointe vers `/visiteur/<site_code>/<id>/signature/` (étape suivante).

✅ **Milestone 8** : sur chaque site, les consignes et le QR code affichés correspondent bien au site, et le PDF flashé est le bon.

---

## Étape 9 — Signature électronique

L'objectif : permettre au visiteur de dessiner sa signature au doigt sur la tablette, et la stocker.

1. Choisir une lib JS de signature pad. Recommandation : **[signature_pad](https://github.com/szimek/signature_pad)** — librairie standalone, simple, sans dépendance.
   - Télécharger le fichier `signature_pad.umd.min.js` et le placer dans `visiteur/static/visiteur/js/`.
   - **Pas de CDN** (cf. [CLAUDE.md](CLAUDE.md) : tous les assets en local).
2. Créer la vue `signature(request, site_code, pk)` et la route associée.
3. Créer le template `signature.html` :
   - Un `<canvas>` qui prend la signature.
   - Un bouton « Effacer » pour recommencer.
   - Un bouton « Valider » qui envoie la signature en POST.
4. Quand le visiteur clique « Valider » : récupérer l'image du canvas (`signaturePad.toDataURL()`), l'envoyer au serveur via un POST (formulaire classique ou fetch).
5. Côté serveur : récupérer la dataURL (`data:image/png;base64,...`) envoyée par le client, stocker la partie base64 dans le champ `signature` (TextField) du modèle.
6. Refuser de valider si la signature est vide (`signaturePad.isEmpty()`).
7. Après validation : rediriger vers l'accueil du site avec un message « Bienvenue [Prénom], merci de patienter à l'accueil ».

✅ **Milestone 9** : workflow complet d'arrivée = identification → consignes → signature → tuile présente sur l'accueil du bon site.

---

## Étape 10 — Pointage de sortie

1. Ajouter une route `path("<slug:site_code>/<int:pk>/sortie/", views.sortie, name="sortie")` dans `visiteur/urls.py`.
2. Écrire la vue `sortie(request, site_code, pk)` qui :
   - Récupère le visiteur via `get_object_or_404(Visiteur, pk=pk, site__code=site_code)` (sécurité : un visiteur de Nantes ne peut pas être marqué sorti depuis la tablette de Montaigu).
   - Met à jour `heure_sortie = timezone.now()`.
   - Sauvegarde.
   - Redirige vers la page d'accueil du site.
3. Sur la page d'accueil, chaque tuile/ligne visiteur doit avoir un bouton « Je pars » qui poste sur cette URL.

**Sécurité de base** : la vue `sortie` doit accepter uniquement les requêtes POST (pas GET — sinon n'importe qui peut faire sortir un visiteur en cliquant sur un lien). Doc : [require_POST](https://docs.djangoproject.com/fr/5.0/topics/http/decorators/#django.views.decorators.http.require_POST).

**Confirmation avant sortie** : un `confirm()` JS natif suffit en v1 (« Confirmez le départ de [Prénom Nom] »). Pas besoin d'AJAX.

✅ **Milestone 10** : un visiteur peut pointer sa sortie sur la tablette de son site. Sa tuile disparaît de la liste des présents.

---

## Étape 11 — Passage en `base_light.html` (mode kiosque tablette)

La tablette tournera en mode kiosque, sans menu. On change donc la base template.

1. Dans tous les templates de l'app (`index.html`, `nouveau.html`, `consignes.html`, `signature.html`), remplacer `{% extends "base.html" %}` par `{% extends "base_light.html" %}`.
2. Vérifier que ça affiche toujours bien.
3. Tester l'app dans Chrome en simulant une tablette : ouvre les devtools (F12), passe en mode responsive, choisis une tablette Android.

---

## Étape 12 — UX tactile pour Android + Chrome

Cible : tablette Android, navigateur Chrome, écran ~10". **Le public est varié (visiteurs de tout âge, parfois en habits de travail, parfois pressés)** — tout doit être gros, lisible à 80 cm, et évident à utiliser sans explication.

- **Polices** : 24 px minimum pour le corps de texte, 32 px pour les libellés de champs, 36-40 px pour les boutons. Pas de gris clair sur blanc — bon contraste obligatoire.
- **Champs de saisie** : hauteur 60-80 px, texte tapé à 28 px minimum. Le visiteur doit pouvoir relire ce qu'il a tapé sans plisser les yeux.
- **Boutons** : minimum 80 px de haut, largeur généreuse, espacement de 20 px entre boutons pour éviter les taps accidentels.
- **Labels et messages** explicites en français simple (« Votre nom », pas « Nom »). Pas d'abréviations.
- **Consignes de sécurité** : police 26-30 px, paragraphes courts, pictogrammes visibles.
- **Pas de hover** (pas de souris sur tablette).
- **Tactile spécifique** : `inputmode="text"`, autocapitalisation des noms (`autocapitalize="words"`), désactiver l'autocorrect sur les champs nom/société (`autocorrect="off"`).
- **Signature pad** : utiliser toute la largeur disponible, hauteur ~250 px (assez grand pour signer confortablement au doigt).
- **Identification visuelle du site** : afficher clairement le nom du site sur chaque écran (ex. bandeau coloré différent par site) pour qu'on sache d'un coup d'œil quelle tablette on regarde.
- **Tester en réel** sur la tablette Android dès que possible. Tester en se tenant debout à 80 cm de l'écran (position d'un visiteur à l'accueil).

Astuce Chrome Android : mode plein écran (« Ajouter à l'écran d'accueil » → ouvre l'app comme une PWA sans la barre d'URL). C'est ce qu'on utilisera en prod, avec l'URL `/visiteur/<site_code>/` bookmarkée en plein écran sur chaque tablette.

✅ **Milestone 12** : workflow testé et confortable sur la vraie tablette, en mode kiosque, sur au moins un site.

---

## Étape 13 — Job de sortie automatique à 23h59

Lire la section « Background jobs » de [CLAUDE.md](CLAUDE.md). Regarder un job existant pour comprendre la structure : par exemple [core/jobs/daily/](core/jobs/daily/).

À faire :

1. Créer `core/jobs/daily/visiteur_auto_sortie.py` en s'inspirant des jobs existants.
2. Le job doit :
   - Récupérer tous les `Visiteur` avec `heure_sortie__isnull=True` et `heure_arrivee__date=today` (tous sites confondus).
   - Mettre `heure_sortie` à 23:59 et `sortie_auto=True`.
   - Logger ce qu'il fait (combien de visiteurs marqués sortis, **par site** dans le log).
3. Tester manuellement : créer 2-3 visiteurs sans sortie sur différents sites, lancer `python manage.py runjobs daily`, vérifier en base.

✅ **Milestone 13** : le job tourne et marque correctement les visiteurs en oubli, sur tous les sites.

---

## Étape 14 — Page « Liste évacuation » (par site)

Une page dédiée, accessible depuis la page principale de chaque tablette via un bouton très visible « LISTE ÉVACUATION ».

1. Route `path("<slug:site_code>/evacuation/", views.evacuation, name="evacuation")` (méthode GET).
2. Vue qui affiche, **pour ce site uniquement** :
   - La liste des visiteurs présents (filtrés sur le site).
   - La liste des salariés actuellement pointés dans l'usine via Kelio, **filtrés sur le site** si Kelio expose cette info.
3. Pour Kelio : importer depuis [core/kelio.py](core/kelio.py). Fonction à appeler à déterminer.

**Important** : si Kelio est en panne ou injoignable, la page doit quand même afficher la liste des visiteurs (avec un avertissement « Kelio indisponible »). Ne pas faire planter la page sur une erreur réseau.

✅ **Milestone 14** : la page `/visiteur/<site>/evacuation/` affiche visiteurs + salariés Kelio du site concerné.

---

## Étape 15 — Envoi du mail évacuation (avec emails du site)

1. Lire [core/emailing.py](core/emailing.py), en particulier la fonction `send_email`.
2. **Plus besoin de variable `.env` pour les emails** : la liste des destinataires est portée par le modèle `Site` (champ rempli dans l'admin Django).
3. Ajouter une vue `envoyer_evacuation(request, site_code)` (POST) qui :
   - Récupère le site et ses emails évacuation.
   - Construit le mail (HTML lisible avec les deux tables visiteurs/Kelio + nom du site dans le sujet).
   - L'envoie à la liste de destinataires du site.
4. Sujet du mail : `[ÉVACUATION — Sectronic Montaigu] Liste des personnes présentes — 22/05/2026 14:32`.
5. Bouton « Envoyer la liste par mail » sur la page évacuation.

Tester : créer un site de test avec ton adresse perso dans les emails évacuation, déclencher l'envoi.

✅ **Milestone 15** : le mail évacuation arrive à la bonne liste de destinataires, avec le bon site dans le sujet.

---

## Étape 16 — Premiers tests unitaires

Lire la doc [Tests Django](https://docs.djangoproject.com/fr/5.0/topics/testing/overview/) et regarder [archives_papier/tests.py](archives_papier/tests.py) comme exemple.

Écrire au moins :

1. Un test qui crée un `Site` puis un `Visiteur` lié, et vérifie qu'il est bien présent (`heure_sortie is None`).
2. Un test qui appelle la vue `sortie` et vérifie que `heure_sortie` est rempli après.
3. Un test pour le job daily : créer 2 visiteurs sans sortie sur deux sites différents, lancer le code du job, vérifier qu'ils sont marqués sortis avec `sortie_auto=True`.
4. **Un test d'isolation entre sites** : créer un visiteur sur Montaigu et un sur Nantes, charger `/visiteur/montaigu/` et vérifier qu'on ne voit que celui de Montaigu.
5. Un test pour la page évacuation **avec Kelio mocké** (regarder [esd/tests.py](esd/tests.py) pour le pattern `unittest.mock.patch`).
6. Un test qu'on ne peut pas valider un enregistrement sans signature.

Lance les tests : `python manage.py test visiteur`. Ils doivent passer.

✅ **Milestone 16** : 6+ tests verts.

---

## Étape 17 — Mode multi-visiteurs (groupe)

Quand 3 personnes de la même société arrivent ensemble pour voir la même personne, elles ne devraient pas avoir à re-taper « société » et « hôte » 3 fois.

Proposition : après la fin du workflow d'un visiteur, afficher un bouton « Ajouter un collègue » qui re-ouvre le formulaire avec société et hôte pré-remplis (chaque personne signe individuellement). Le site reste évidemment celui de la tablette.

✅ **Milestone 17** : plusieurs visiteurs d'un groupe peuvent être enregistrés rapidement.

---

## Étape 18 — Page historique pour l'admin

Une vue qui liste **toutes** les visites passées, avec filtre par date, **site**, société, hôte. Accessible uniquement aux utilisateurs connectés (décorateur `@login_required`). Sur `base.html` (avec menu), pas `base_light.html`.

Bonus : permettre de cliquer sur une ligne pour voir le détail d'une visite, y compris la signature stockée.

---

## Étape 19 — Migration sur SQL Server

Si tout est stable, on bascule les modèles `Site` et `Visiteur` de SQLite vers SQL Server (ajout de `_use_db = "sql_server"` dans `Meta`, conversion des données existantes). C'est l'occasion de découvrir le routage multi-DB.

---

## Étape 20 — Documentation et démo

1. Créer `visiteur/README.md` : à quoi sert l'app, comment elle marche, comment ajouter un nouveau site (admin Django), comment lancer les tests.
2. Préparer une démo de 10-15 min à l'équipe : montrer le workflow complet sur deux sites différents (arrivée → consignes → signature → présence → départ → évacuation → mail), et démontrer que les sites sont bien isolés.
3. Pendant la démo, expliquer **avec tes mots** ce que fait chaque morceau de code. Si tu sais l'expliquer, tu as compris.

✅ **Milestone 20 (fin)** : l'app est déployable, testée, documentée, démontrée sur plusieurs sites.

---

## Routes attendues à la fin

Toutes les routes tablette incluent le `site_code` en début de chemin.

| URL | Vue | Description |
|-----|-----|-------------|
| `/visiteur/<site>/` | `index` | Page principale tablette du site : présents + boutons « S'enregistrer » et « Liste évacuation » |
| `/visiteur/<site>/nouveau/` | `nouveau` | Formulaire d'identification (le site est imposé par l'URL) |
| `/visiteur/<site>/<id>/consignes/` | `consignes` | Lecture des consignes de sécurité du site |
| `/visiteur/<site>/<id>/signature/` | `signature` | Signature électronique |
| `/visiteur/<site>/<id>/sortie/` | `sortie` | POST de pointage de sortie |
| `/visiteur/<site>/evacuation/` | `evacuation` | Page évacuation du site (visiteurs + Kelio) |
| `/visiteur/<site>/evacuation/envoyer/` | `envoyer_evacuation` | POST déclenchant le mail évacuation |
| `/visiteur/historique/` | `historique` | Historique admin tous sites (avec filtre par site) |

---

## Considérations techniques (récap)

- **Multi-sites natif** : modèle `Site` + FK sur `Visiteur`. Chaque tablette ouvre une URL `/visiteur/<site_code>/` en mode kiosque, le site est imposé par l'URL.
- **Isolation par site** : toutes les vues tablette filtrent sur le site de l'URL ; un visiteur de Nantes n'apparaît jamais sur la tablette de Montaigu.
- **Config par site portée par le modèle** `Site` (URL des consignes, emails d'évacuation, nom affiché) — pas de variables `.env` qui devraient être listées par site.
- **Base template** : `base_light.html` (cf. [CLAUDE.md](CLAUDE.md) section « Base Templates ») — pas de menu, plein écran, parfait pour tablette en mode kiosque.
- **Pas d'authentification** sur les pages tablette : libre-service à l'accueil. La page admin Django et l'historique restent protégés.
- **Tactile** : tout doit être **gros et lisible à 80 cm de distance** (cf. étape 12). Polices ≥ 24 px corps / ≥ 32 px libellés / ≥ 36 px boutons. Champs de saisie hauts (60-80 px), boutons généreux (≥ 80 px), bon contraste, pas de hover.
- **Pas de scan de badge** en v1. Tout est saisie clavier tactile + signature au doigt.
- **Kelio** : pour la liste « salariés présents », s'appuyer sur [core/kelio.py](core/kelio.py).
- **Email évacuation** : via [core/emailing.py](core/emailing.py), destinataires lus depuis le modèle `Site`.
- **Job auto-sortie** : dans `core/jobs/daily/`, traite tous les sites.
- **Stockage initial sur SQLite** (base `default`), migration vers SQL Server à la fin.
- **Pas de mail visiteur** stocké (RGPD : minimisation).
- **Signature électronique** : librairie [signature_pad](https://github.com/szimek/signature_pad) hébergée localement (pas de CDN).

---

## Hors scope v1

Pour ne pas dériver, ces points sont **explicitement remis à plus tard** :

- Scan de carte d'identité ou QR code à l'arrivée.
- Pré-enregistrement par l'hôte avant l'arrivée du visiteur.
- Statistiques / dashboard des visites (par société, par mois, par site).
- Impression d'un badge visiteur.
- Notification du salarié visité (mail / Mattermost).
- Autocomplétion sur l'hôte visité (Kelio).
- Pré-remplissage pour visiteurs récurrents.
- Email de consignes envoyé au visiteur (les consignes sont lues et signées sur la tablette, c'est suffisant).
- Bascule automatique d'un site à un autre depuis la même tablette (un site = une tablette = une URL).

---

## Ressources utiles

- [Documentation Django (FR)](https://docs.djangoproject.com/fr/5.0/)
- [Django Girls — tutoriel débutant](https://tutorial.djangogirls.org/fr/)
- [Doc ForeignKey Django](https://docs.djangoproject.com/fr/5.0/topics/db/examples/many_to_one/)
- [signature_pad — librairie JS](https://github.com/szimek/signature_pad)
- [qrcode — librairie Python](https://pypi.org/project/qrcode/)
- [CLAUDE.md](CLAUDE.md) — conventions du projet Sectronic
- App de référence pour la structure : [archives_papier/](archives_papier/)
- App de référence pour les tests avec mocks : [esd/tests.py](esd/tests.py)
