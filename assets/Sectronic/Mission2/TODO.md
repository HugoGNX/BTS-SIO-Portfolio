# App visiteur — TODO

## Migration SQL Server

La migration vers SQL Server reste à finaliser.

Points à vérifier :

* migration des modèles Site et Visiteur
* exécution des migrations sur SQL Server
* validation des tests après migration (attention il faut les refaire)

## Enregistrement avant signature

### Problème

Le visiteur est créé en base avant d'avoir signé les consignes de sécurité.

Si l'utilisateur quitte le processus avant la signature :

* visite créée
* aucune signature enregistrée

### Cause

Le modèle est sauvegardé dès la validation du formulaire d'identification.

### Solution envisagée

Stocker temporairement les informations en session Django, puis créer le visiteur uniquement après validation de la signature.

Priorité : Moyenne

---

## Soumissions multiples (double-tap)

### Problème

Un utilisateur peut appuyer plusieurs fois rapidement sur les boutons de validation.

Conséquences :

* doublons de visiteurs
* signatures manquantes
* accumulation des messages de succès
* incohérence des listes de présence

### Cause

Absence de protection contre les soumissions multiples.

### Solutions envisagées

Front-end :

* désactiver le bouton après le premier clic
* afficher un indicateur de chargement

Back-end :

* mettre en place un mécanisme d'idempotence
* empêcher plusieurs créations identiques dans un court intervalle

Priorité : Haute

---

## Bouton "Je pars"

### Problème

Plusieurs clics rapides génèrent plusieurs notifications successives.

### Solution envisagée

Désactivation immédiate du bouton après soumission.

Priorité : Faible

---

## Filtrage des salariés par site (Kelio)

### Problème

L'API Kelio utilisée pour récupérer les salariés présents ne permet actuellement pas d'identifier avec certitude le site de présence d'un salarié.

Lors d'une évacuation, la liste des salariés affichée contient donc l'ensemble des salariés présents remontés par Kelio, sans distinction entre les différents sites Sectronic.

### Conséquences

- Les listes d'évacuation peuvent contenir des salariés appartenant à d'autres sites.
- Les responsables doivent effectuer un tri manuel lors du contrôle des présences.
- Les informations affichées ne reflètent pas précisément l'occupation réelle du site concerné.

### Solution envisagée

Étudier les possibilités offertes par Kelio pour :

- récupérer le site de présence d'un salarié ;
- exploiter une information de badgeage ou de localisation ;
- mettre en place un mapping entre les données Kelio et les sites de l'application.

Une fois cette information disponible, filtrer les salariés afin de n'afficher que ceux présents sur le site concerné par l'évacuation.

### Priorité

Moyenne
