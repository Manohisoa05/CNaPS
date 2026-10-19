Explication de la conception, de A à Z
1. Le but du système

Ce schéma modélise une application de gestion des incidents informatiques, comme un helpdesk ou un outil de type ITIL. Un employé signale un problème sur une application. Une équipe le prend en charge, le traite, le résout, puis il est validé. On en tire ensuite des enseignements (analyse de cause racine, base de connaissances).

Les 17 tables se regroupent en 4 blocs :

Bloc	Rôle	Tables
1. Utilisateurs et droits	Qui peut faire quoi	UTILISATEUR, ROLE, PERMISSION, ROLE_PERMISSION, UTILISATEUR_ROLE
2. Référentiels	Les listes de valeurs de base	APPLICATION, TYPE_CATEGORIE, PRIORITE_SLA
3. Cœur	L'incident lui-même	INCIDENT
4. Cycle de vie et amélioration	Ce qui se passe autour de l'incident	AFFECTATION, TRAITEMENT, INCIDENT_HISTORIQUE, RESOLUTION, VALIDATION, PIECE_JOINTE, RCA, ARTICLE_KB
2. Bloc 1 : utilisateurs et droits (modèle RBAC)

RBAC signifie Role-Based Access Control, le contrôle d'accès par rôles. L'idée est de ne jamais donner de droits directement à une personne, mais de passer par des rôles.

UTILISATEUR : identifié par son matricule (CHAR(6), identifiant métier de l'employé), avec sa direction, son service, sa fonction et son mail. Les codes direction, service et fonction viennent probablement d'un référentiel RH externe, absent de ce schéma.
ROLE : un profil, par exemple Demandeur, Technicien, Responsable ou Administrateur.
PERMISSION : une autorisation précise, définie par un module et une action (par exemple INCIDENT + CREATE, ou RCA + READ).
UTILISATEUR_ROLE et ROLE_PERMISSION : des tables d'association.

Le principe de cette chaîne est : Utilisateur → Rôles → Permissions. Un utilisateur peut avoir plusieurs rôles, et un rôle regroupe plusieurs permissions. Ce sont deux relations plusieurs-à-plusieurs, qu'on ne peut pas stocker dans une seule colonne. On crée donc une table intermédiaire dont la clé primaire est composite (les deux colonnes ensemble), ce qui empêche aussi les doublons.

L'intérêt est que pour changer les droits d'un profil, on modifie un seul rôle au lieu de toucher chaque utilisateur.

3. Bloc 2 : les référentiels

Ce sont des tables de paramétrage, rarement modifiées :

APPLICATION : le code, le libellé, le propriétaire (matricule du responsable) et la criticité. Une application critique justifie une réaction plus rapide.
TYPE_CATEGORIE : associe un type d'incident (par exemple Panne ou Demande) à une catégorie (par exemple Réseau ou Messagerie).
PRIORITE_SLA : définit chaque niveau de priorité (Critique, Haute, etc.) avec son délai de résolution, son calendrier (heures ouvrées ou 24h/24) et un seuil d'alerte. SLA veut dire Service Level Agreement, l'engagement de délai pris envers les utilisateurs.
4. Bloc 3 : INCIDENT, la table centrale

Tout le reste s'accroche à elle. Chaque incident a :

un NUMERO (identifiant unique) ;
un DEMANDEUR (le matricule de celui qui signale) ;
l'APPLICATION concernée ;
un TYPE et une CATEGORIE ;
un TITRE (obligatoire) et une DESCRIPTION ;
une PRIORITE et un STATUT. Le statut indique l'étape du workflow, par exemple Nouveau, Affecté, En cours, Résolu ou Clôturé. Ces valeurs ne sont pas définies dans le schéma, ce sont des exemples.
5. Bloc 4 : le cycle de vie

Chaque table répond à une question précise, et toutes sont reliées à INCIDENT par une clé étrangère NUMERO_INCIDENT. La relation est 1 incident pour N lignes :

Table	Question à laquelle elle répond	Contenu
AFFECTATION	Qui s'en occupe ?	Équipe, responsable, technicien, motif (utile en cas de réaffectation)
TRAITEMENT	Qu'a-t-on fait ?	Diagnostic, actions menées, commentaires
RESOLUTION	Pourquoi et comment est-ce réglé ?	Cause, solution, action corrective, action préventive
VALIDATION	Le résultat est-il accepté ?	Décision, commentaire, acteur, date
INCIDENT_HISTORIQUE	Qu'est-ce qui a changé, quand, par qui ?	Acteur, date, action, ancienne et nouvelle valeur
PIECE_JOINTE	Quelles preuves sont jointes ?	Nom, type, taille, auteur (captures d'écran, logs)
RCA	Quelle est la cause profonde, et comment éviter la récidive ?	Cause racine, actions, responsable, échéance

Pourquoi des tables séparées plutôt que tout dans INCIDENT ? Un incident peut être réaffecté plusieurs fois et avoir plusieurs traitements. Mettre tout cela dans une seule table donnerait des dizaines de colonnes souvent vides et ferait perdre l'historique.

INCIDENT_HISTORIQUE joue le rôle de journal d'audit. Il enregistre chaque modification (par exemple statut « Nouveau » → « En cours ») avec son auteur et son horodatage automatique. C'est ce qui garantit la traçabilité.

RCA (Root Cause Analysis) ne concerne que les incidents importants ou récurrents. On cherche la vraie cause et on suit les actions avec un responsable et une échéance.

ARTICLE_KB est la base de connaissances. Chaque article décrit un symptôme, sa cause et sa solution. La prochaine fois qu'un problème similaire survient, le technicien trouve la solution sans repartir de zéro. Cette table est indépendante : elle n'a aucune clé étrangère.

6. Le flux complet

Création → Affectation → Traitement → Résolution → Validation → (RCA) → Base de connaissances

À chaque étape, le statut de l'incident change et une ligne est ajoutée à l'historique.

Exemple concret :

Un employé (matricule 000123) déclare « Application RH inaccessible », priorité Haute. C'est une ligne dans INCIDENT.
Le responsable l'affecte à l'équipe Support N2 et au technicien 000456. C'est une ligne dans AFFECTATION.
Le technicien note son diagnostic et ses actions dans TRAITEMENT, et joint une capture d'écran dans PIECE_JOINTE.
Il renseigne la cause (certificat expiré) et la solution dans RESOLUTION.
Le demandeur confirme que tout fonctionne : une ligne dans VALIDATION, décision « Accepté ».
Si l'incident est jugé important, une RCA est ouverte. Une fiche ARTICLE_KB est créée pour la suite.
7. Limites du schéma (à connaître pour répondre aux questions)
Peu de clés étrangères sur INCIDENT. Le demandeur, l'application, le type, la catégorie et la priorité sont stockés en simple texte, sans lien déclaré vers les référentiels. Rien n'empêche donc une valeur incohérente. C'est pour cela que j'avais préparé les relations implicites en commentaire dans le fichier DBML.
Aucune date sur INCIDENT. Il n'y a pas de date de création ni de clôture, alors que le calcul du SLA en a besoin. On peut les retrouver via INCIDENT_HISTORIQUE, mais c'est indirect.
PRIORITE_SLA.DELAI n'a pas d'unité (minutes, heures ?).
PIECE_JOINTE ne stocke que les métadonnées, pas le fichier ni son chemin.
TRAITEMENT et RESOLUTION autorisent plusieurs lignes par incident. Si vous voulez une seule résolution par incident, il faut ajouter une contrainte UNIQUE sur NUMERO_INCIDENT.
ARTICLE_KB n'est relié à aucun incident ni aucune résolution.
Les clés NUMBER n'ont pas de séquence ni d'identity défini : l'auto-incrémentation reste à prévoir côté Oracle.

Présenter ces limites avec des pistes d'évolution montre que vous maîtrisez la conception.

Plan de présentation suggéré : besoin → vue d'ensemble en 4 blocs → diagramme → scénario concret → limites et évolutions.

Si vous voulez, je peux en faire un support PowerPoint prêt à présenter.

---

# Complément : front Angular de gestion des incidents

## 1. Objectif du front

Le front permet au demandeur de créer et suivre ses incidents, puis à l'équipe SI de les dispatcher, traiter, résoudre et faire valider. La sidebar distingue les pages **Demande**, **Mes demandes**, **Nouveaux**, **Dispatch**, **Suivi des incidents** et **Documentation**.

## 2. Pages et routes

| Étape | Page | Route | Rôle principal |
|---|---|---|---|
| Déclaration | Demande d'incident | `/general/incident-form` | Le demandeur décrit l'incident, choisit les informations générales et peut joindre des fichiers. |
| Suivi demandeur | Mes tickets | `/general/incidents` | Le demandeur recherche, filtre et consulte ses tickets. |
| Sélection | Nouveaux | `/general/incident-treatment` | Le responsable choisit un incident nouveau. Cette page ne fait aucune affectation. |
| Affectation | Dispatch | `/general/incident-dispatch` | Le Chef de Service ou le Directeur choisit le responsable, l'équipe et le technicien. |
| Traitement | Détail du ticket | `/general/incident-detail/:id` | Le technicien analyse, traite et résout; le responsable valide. |
| Reporting | Suivi des incidents | `/general/incident-dash` | Page de suivi et d'indicateurs existante. |

## 3. Workflow retenu

1. Le demandeur soumet une demande d'incident. Les pièces jointes sont facultatives; les formats affichés sont PNG, JPG, PDF, DOC et XLS, avec une limite de 10 Mo par fichier.
2. Dans **Nouveaux**, le responsable sélectionne un incident. Aucun technicien n'est encore assigné.
3. Dans **Dispatch**, le Chef de Service ou le Directeur sélectionne l'équipe et le technicien. La confirmation transmet l'incident affecté à la page de traitement.
4. Le technicien clique **Prendre en charge**. Le statut passe à **En cours**.
5. Le technicien enregistre l'analyse, puis le traitement. Ces actions sont distinctes pour rendre leur ordre visible.
6. Pour enregistrer la résolution, le technicien renseigne la cause, la solution, l'action corrective et l'action préventive. Le ticket passe alors à **À valider**.
7. Le responsable enregistre sa décision : **Acceptée** ferme le ticket; **Refusée** renvoie l'incident dans **Dispatch** pour une nouvelle affectation.

La mise en attente requiert un motif et propose les dates de début et de reprise. Les changements de statut et les actions sont ajoutés à l'historique affiché dans le détail du ticket.

> **Règle à confirmer dans le SFD :** la consigne métier retenue ici renvoie une résolution refusée à **Dispatch**. Le SFD initial indiquait un retour à **En cours**. La règle définitive doit être validée avant le branchement backend.

## 4. Fichiers principaux

| Fichier | Responsabilité |
|---|---|
| [sidebar.component.ts](modele-front-lucy/modele-front-lucy/src/app/modules/general/components/sidebar/sidebar.component.ts) | Liens de navigation de la sidebar. |
| [general-routing.module.ts](modele-front-lucy/modele-front-lucy/src/app/modules/incident/general-routing.module.ts) | Association entre les routes et les composants. |
| [incident-form.component.html](modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-form/incident-form.component.html) | Formulaire de demande, priorité/SLA indicatif, pièces jointes et actions. |
| [incident-list.component.html](modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-list/incident-list.component.html) | Liste **Mes tickets**, indicateurs, filtres et tableau. |
| [incident-treatment-queue.component.html](modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-treatment-queue/incident-treatment-queue.component.html) | Liste **Nouveaux**, réservée à la sélection d'un incident. |
| [incident-dispatch.component.html](modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-dispatch/incident-dispatch.component.html) | Affectation du responsable, de l'équipe et du technicien. |
| [incident-detail.component.html](modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-detail/incident-detail.component.html) | Prise en charge, analyse, traitement, résolution, validation et historique. |

Les fichiers `.ts` portent les données et les transitions locales; les fichiers `.scss` définissent la présentation responsive.

## 5. État actuel et limites

- Les incidents, utilisateurs, responsables, équipes et techniciens sont des données de démonstration front.
- Les changements de statut et l'historique ne sont pas persistés dans une base de données.
- Les rôles affichés ne constituent pas un contrôle d'accès; l'autorisation devra être vérifiée par le backend.
- Les données de navigation entre **Nouveaux**, **Dispatch** et le détail sont transmises par le routeur Angular; elles ne remplacent pas un stockage backend.
- La page **Mes tickets** utilise également des données locales pour démontrer la recherche, les filtres, la pagination et l'export CSV.

## 6. Vérifications effectuées

- Le front Angular a recompilé avec succès via le serveur de développement.
- Le parcours dispatch → prise en charge → analyse → traitement → résolution → validation acceptée → fermeture a été testé.
- Le parcours de validation refusée a été testé et renvoie le ticket dans **Dispatch** avec le motif et les informations préremplies.