

- `gestion_incidents/src/main/resources/application.properties`
  - Configuration Spring Boot globale.
  - Définit le port du backend sur 8082 par défaut.
  - Configure la limite des pièces jointes à 10 Mo chacune et 55 Mo par requête.
  - Indique la datasource Oracle et les variables d’environnement.
  - N’active pas le profil local par défaut : cela reste contrôlé au lancement.

- `gestion_incidents/src/main/resources/application-local.properties`
  - Fichier dédié au profil local.
  - Le matricule de test courant est `000008` (compte technicien pour tester les incidents attribués).
  - Pour tester un autre compte local, modifier `app.local-user.matricule` (par exemple `000001` ou `000002`) puis redémarrer le backend avec le profil `local`.
  - Ce mécanisme est uniquement destiné au développement et ne constitue pas une authentification.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/user/controller/CurrentUserController.java`
  - Expose l’endpoint REST `/api/current-user`.
  - N’est activé que sous le profil Spring `local`.
  - Renvoie les infos de l’utilisateur courant au frontend.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/user/service/CurrentUserService.java`
  - Récupère l’utilisateur défini dans `app.local-user.matricule` depuis Oracle (actuellement `000008`).
  - Lit ses infos dans `UTILISATEUR`.
  - Lit également les libellés de direction et service depuis le schéma `DRH`.
  - Joint aussi `UTILISATEUR_ROLE` et `ROLE` pour récupérer les rôles.
  - Construit le DTO renvoyé au frontend.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/user/dto/CurrentUserResponse.java`
  - Représente la réponse JSON de l’utilisateur courant.
  - Contient : matricule, nom, prénom, mail, direction, service, fonction, rôles.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/controller/IncidentController.java`
  - Expose les endpoints REST pour créer une demande JSON ou multipart (`POST /api/incidents`) et lister les demandes du demandeur (`GET /api/incidents`).
  - Lit le header `utilisateur` pour identifier le demandeur.
  - Renvoie uniquement les demandes qui appartiennent au matricule transmis.
  - Le multipart accepte la partie JSON `incident` et jusqu’à cinq parties fichier `attachments`.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/controller/LocalIncidentQueueController.java`
  - Expose `GET /api/incidents/queue` uniquement sous le profil `local`.
  - Retourne les incidents réellement enregistrés dans Oracle, triés du plus récent au plus ancien; la date de création provient de `INCIDENT_HISTORIQUE`.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/controller/LocalIncidentDispatchOptionsController.java`
  - Expose `GET /api/incidents/dispatch-options` uniquement sous le profil `local`.
  - Charge les responsables SI et techniciens depuis `UTILISATEUR_ROLE`/`ROLE` et les services depuis `DRH.DRH_SERVICE` (lecture uniquement).

- `gestion_incidents/src/main/resources/db/manual/003_applications_modules.sql`
  - Script manuel à exécuter une seule fois dans `GESTION_INCIDENTS`; crée les catalogues application/module, leur association, la séquence d’applications et la colonne de module sur `INCIDENT`.
  - Insère les 24 applications et les 36 modules transmis ainsi que leurs rattachements; les opérations MERGE de données peuvent être rejouées.

- `gestion_incidents/src/main/resources/db/manual/004_module_sequence.sql`
  - Script manuel à exécuter une fois après le script `003`; crée `SEQ_MODULE_SI` en commençant après le plus grand code `MOD-` déjà présent.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/controller/LocalApplicationCatalogController.java`
  - Expose sous profil `local` la lecture des applications et modules, l’ajout d’une application et l’association d’un module existant par le responsable.
  - Endpoints : `GET /api/applications`, `GET /api/applications/modules`, `GET /api/applications/{code}/modules`, `POST /api/applications`, `POST /api/applications/modules` et `PUT /api/applications/{code}/modules/{moduleCode}`.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/service/ApplicationCatalogService.java`
  - Lit/ajoute les entrées du catalogue Oracle, crée un module avec un code généré et l’associe à l’application sélectionnée, associe sans doublon les modules existants et vérifie l’association avant dispatch.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/application-catalog/application-catalog.component.ts`
  - Page « Applications disponibles » du responsable : liste le catalogue, ajoute applications et modules, associe modules existants et présente les rattachements nécessaires au dispatch.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/controller/LocalIncidentDispatchController.java`
  - Expose `PATCH /api/incidents/{numero}/dispatch`; enregistre le code application, le module, le service et le matricule technicien choisis, le statut `AFFECTE` et l’acteur dans l’historique Oracle.
  - Expose aussi `GET /api/incidents/assigned-to-me`, qui retourne les incidents liés au matricule utilisateur courant fourni par le profil local, sans accepter un matricule arbitraire du frontend.

- `gestion_incidents/src/main/resources/db/manual/005_incident_technician_assignment.sql`
  - À exécuter une fois dans `GESTION_INCIDENTS` pour ajouter les colonnes de service et de technicien affectés à `INCIDENT`.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/repository/IncidentRepository.java`
  - Fournit la recherche des incidents par matricule du technicien attribué.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/service/IncidentService.java`
  - Vérifie si le matricule du demandeur est présent.
  - Valide les champs de la demande.
  - Enregistre l’incident en Oracle via la repository JPA.
  - Récupère les demandes du demandeur courant pour la page « Mes demandes ».
  - Valide les pièces jointes (jusqu’à 5, 10 Mo chacune, extensions autorisées), puis enregistre contenu et métadonnées dans Oracle dans la même transaction que l’incident.
  - Accepte une demande sans application; cette information sera attribuée ultérieurement par le responsable.
  - Remplit le statut initial, la demandeur, le type, la catégorie, etc.
  - Retourne le JSON de réponse avec le numéro généré.
  - Pour la file de workflow, fournit la date de création depuis `INCIDENT_HISTORIQUE.DATE_ACTION` pour l’action `CREATION`, car la table `INCIDENT` ne porte pas de colonne de date.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/repository/IncidentRepository.java`
  - Fournit la recherche des incidents par matricule du demandeur.
  - Fournit également la file complète des incidents pour l’interface workflow locale.
  - Trie la liste par numéro d’incident décroissant, sans modifier le schéma Oracle.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/entity/PieceJointeEntity.java`
  - Mappe la table Oracle `PIECE_JOINTE`, dont le BLOB `CONTENU` stocke les octets du fichier.
  - Associe nom, type MIME, taille, auteur et numéro d’incident.
  - Utilise la séquence Oracle `SEQ_PIECE_JOINTE` pour `ID_PIECE`.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/repository/PieceJointeRepository.java`
  - Persiste les métadonnées et le contenu des fichiers joints.

- `gestion_incidents/src/test/java/com/example/gestion_incidents/incident/IncidentControllerTests.java`
  - Vérifie la création d’une demande et le rejet d’une requête sans matricule.
  - Vérifie que `GET /api/incidents` ne renvoie que les demandes du demandeur spécifié.
  - Vérifie l’envoi multipart et la persistance du contenu d’une pièce jointe.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/dto/IncidentCreateRequest.java`
  - DTO de la requête JSON envoyée depuis le front pour créer un incident.
  - Contient type, catégorie, titre, description et priorité; l’application est facultative et pourra être définie ultérieurement par le responsable.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/reference/controller/IncidentReferenceController.java`
  - Expose les référentiels Oracle avec `GET /api/references/categories` et `GET /api/references/types`.
  - Le endpoint des types accepte `categoryCode` pour ne renvoyer que les types de la catégorie sélectionnée.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/reference/service/IncidentReferenceService.java`
  - Lit les catégories et les types depuis `CATEGORIE_INCIDENT` et `TYPE_INCIDENT`.
  - Utilise une requête paramétrée pour filtrer les types par catégorie.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/reference/dto/CategoryReferenceResponse.java`
  - Réponse API d’une catégorie : code, libellé et responsable.

- `gestion_incidents/src/main/java/com/example/gestion_incidents/reference/dto/TypeReferenceResponse.java`
  - Réponse API d’un type : code, libellé et code catégorie.

- `modele-front-lucy/modele-front-lucy/src/environments/environment.ts`
  - URL backend pour le front en développement.
  - Ici, pointe vers `http://localhost:8082/api`.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/services/incident-api.service.ts`
  - Service Angular qui appelle le backend.
  - `getCurrentUser()` appelle `/api/current-user`.
  - `createIncident(request, requesterMatricule, attachments)` envoie la demande en multipart avec son JSON, les fichiers et le header `utilisateur`.
  - `getMyIncidents(requesterMatricule)` récupère la liste des tickets de ce demandeur depuis `/api/incidents`.
  - `getIncidentQueue()` charge la file réelle de `/api/incidents/queue`.
  - `getIncidentDispatchOptions()` charge les responsables, techniciens et services Oracle pour la page Dispatch.
  - `getApplications()` et `getApplicationModules()` chargent le catalogue utilisé au dispatch.
  - `dispatchIncident()` persiste le choix application/module, le service et le technicien attribués dans Oracle.
  - `getCategories()` et `getTypes(categoryCode)` chargent les référentiels de catégories et types depuis Oracle.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-list/incident-list.component.ts`
  - Charge l’utilisateur courant, puis ses demandes.
  - Charge les référentiels de catégories et types pour afficher leurs libellés dans la liste au lieu des seuls codes.
  - Remplace les exemples de tickets codés en dur par les données retournées par le backend.
  - Affiche catégorie, type, titre, description, direction et service avec les données du demandeur courant.
  - Gère les compteurs, la recherche, les filtres par statut/priorité, la pagination et l’export CSV.
  - Affiche les erreurs de chargement via un toast.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/assigned-incidents/assigned-incidents.component.ts`
  - Page technicien « Incidents attribués »; charge depuis l’endpoint local uniquement les incidents attribués à l’utilisateur courant, permet la recherche et ouvre le traitement d’un ticket.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-detail/incident-detail.component.ts`
  - Pour un accès depuis « Incidents attribués », revalide le rôle technicien et vérifie que l’incident figure dans la liste assignée à l’utilisateur courant avant d’afficher le traitement.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-list/incident-list.component.html`
  - Affiche les tickets réels, l’état de chargement et le message lorsqu’aucune demande n’est disponible.
  - Présente les colonnes numéro, catégorie, type, titre, description, direction, service, priorité et statut; l’application n’est plus affichée.
  - Les colonnes de date et de département ne sont pas affichées : ces valeurs ne sont pas présentes dans la structure actuelle de `INCIDENT`.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-treatment-queue/incident-treatment-queue.component.html`
  - Affiche aussi les libellés de catégorie et de type à côté des colonnes demandeur, application et incident dans la file des nouveaux incidents.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-list/incident-list.component.scss`
  - Utilise le bleu foncé `#174a75` pour les accents, boutons, pagination active, icônes et priorité normale de la page « Mes demandes ».

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-form/incident-form.component.ts`
  - Composant de formulaire.
  - Au chargement, appelle `getCurrentUser()`.
  - Charge la liste des catégories et demande les types associés au changement de catégorie.
  - Remplit le demandeur avec les infos de l’utilisateur courant.
  - Ne demande plus l’application ni le module à l’usager; ces informations seront déterminées ultérieurement par le responsable.
  - Affiche la direction et le service du demandeur séparément; les coordonnées téléphoniques et le lieu ne sont plus demandés.
  - Gère la soumission de la demande.
  - Transmet les fichiers sélectionnés au backend au lieu de bloquer la soumission.
  - Affiche les toasts d’erreur/succès.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-form/incident-form.component.html`
  - Template du formulaire UI.
  - Contient les champs de demande et les composants visuels, sans choix d’application ou de module par le demandeur.
  - Présente un en-tête de page avec icône, titre et sous-titre, harmonisé avec celui de « Mes tickets ».
  - N’affiche plus la notification inline, uniquement les toasts.

- `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-form/incident-form.component.scss`
  - Définit le style de l’en-tête de « Demande d'incident » en cohérence avec l’en-tête de la page « Mes tickets », avec un espacement augmenté.
  - Harmonise la typographie du titre « Pièce(s) jointe(s) (facultatif) » avec les autres titres de section.

- `modele-front-lucy/modele-front-lucy/src/app/modules/general/interceptors/api.interceptor.ts`
  - Intercepteur HTTP global.
  - Ajoute les headers nécessaires aux requêtes backend.
  - Préserve le matricule déjà transmis au lieu de l’écraser par une valeur vide.
  - Gère aussi les erreurs HTTP (par ex. 401).

- Backend = récupération utilisateur + enregistrement de la demande Oracle.
- Frontend = affichage du demandeur + soumission de la demande + toasts.
- Profil local = mécanisme de test temporaire, sans authentification réelle.

## Privilèges du Responsable SI (développement local)

- Pour tester les écrans du responsable, configurez `app.local-user.matricule=000003` (rôle `ROLE_RESPONSABLE_SI`) puis redémarrez le backend avec le profil `local`. Le profil est actuellement réglé sur le compte technicien `000008`.
- Le menu du responsable affiche maintenant « Demande », « Mes demandes », « Dispatch » et « Traitement ».
- « Dispatch » ouvre la file de dispatch et son affectation à un technicien. « Traitement » est une file distincte qui permet au responsable de choisir un incident puis de le prendre en charge lui-même, sans dispatch préalable.
- Les listes Dispatch et Traitement direct utilisent `GET /api/incidents/queue`; les options d’affectation de Dispatch viennent de `GET /api/incidents/dispatch-options`. Les données de démonstration codées en dur ont été retirées.
- Le menu technicien expose « Demande », « Mes demandes » et « Incidents attribués ». Chaque ligne de cette dernière page possède le bouton « Traiter », qui ouvre l’écran de traitement après vérification du matricule courant auprès de `GET /api/incidents/assigned-to-me`.
- En développement, la barre supérieure de l’application (`modules/general/components/navbar`) vérifie les notifications toutes les 30 secondes : le responsable reçoit une notification pour chaque incident au statut `NOUVEAU`, le technicien pour chaque incident affecté à son matricule. Les nouvelles entrées affichent un toast; la cloche donne accès à la liste, au compteur non lu et à la navigation vers la file concernée. L’état lu est conservé dans le stockage local du navigateur par rôle et matricule.
- Au premier relevé d’un rôle/matricule, les incidents actifs non encore enregistrés localement déclenchent aussi un toast. Cela permet notamment de tester le workflow avec une seule session en affectant avec le responsable, puis en redémarrant le backend avec le matricule technicien. Les notifications déjà connues localement ne déclenchent pas de doublons. Elles reposent sur les files Oracle déjà existantes, n’ajoutent aucune table et ne sont activées que dans le build frontend de développement; ce mécanisme de polling n’est ni une authentification ni un service de notification de production.
- L’écran de traitement des responsables et techniciens est organisé en cinq sections visibles : Diagnostic (constat, hypothèses, vérifications/résultats), Actions (journal horodaté et temps passé), Communication (commentaire demandeur et note interne), Attente (motif, début et reprise) et Résolution (cause, solution, actions corrective et préventive). Dispatch et prise en charge ne sont plus des étapes affichées dans cet écran; la validation/fermeture par le responsable reste à implémenter ultérieurement.
- Les détails du demandeur sont chargés dans le profil local via `GET /api/users/{matricule}`, à partir des mêmes tables Oracle de référence que l’utilisateur courant.
- Les endpoints de file, d’affectation, d’utilisateur courant et de traitement ne sont actifs qu’en profil `local`. Diagnostic, actions et communications sont maintenant enregistrés ligne par ligne dans `TRAITEMENT`, dans les colonnes existantes; les champs contiennent un JSON compact qui préserve le détail, l’auteur et la date. La lecture recharge ces informations après réouverture. Le premier diagnostic fait aussi passer `INCIDENT.STATUT` de `AFFECTE` à `EN_COURS`. L’accès est limité au responsable SI ou au technicien affecté dans ce workflow local. Attente, résolution et historique exhaustif restent à persister.
- Le parcours de traitement propose maintenant une fiche de prévisualisation pour le diagnostic et la communication, ainsi qu’une prévisualisation du journal avant sauvegarde. Les actions non confirmées restent en brouillon, modifiables et supprimables; les actions déjà enregistrées peuvent être supprimées individuellement. La navigation Précédent/Suivant est disponible et empêche de passer aux étapes suivantes avant confirmation des étapes précédentes.
- La mise en attente et la reprise sont enregistrées comme événements JSON dans la colonne `TRAITEMENT.COMMENTAIRES` via le champ API `attente`, sans ajout de colonne. Le backend valide le motif, la date, l’auteur et le statut de départ, puis met à jour `INCIDENT.STATUT` (`EN_COURS` → `EN_ATTENTE` → `EN_COURS`). L’écran recharge les événements et dates depuis `TRAITEMENT`.
- Avant le premier enregistrement, exécuter manuellement `gestion_incidents/src/main/resources/db/manual/006_treatment_sequence.sql` avec le schéma `GESTION_INCIDENTS` pour créer `SEQ_TRAITEMENT` à partir du prochain identifiant disponible. Le script ne touche pas aux lignes de `TRAITEMENT`; il n’a pas été exécuté par l’application.
- L’écran de confirmation du dispatch est une étape interne ouverte depuis la file. Les pages accessibles uniquement aux autres rôles sont masquées et protégées par `IncidentPrivilegesGuard`.
- Le filtrage du menu et le guard sont des contrôles d’interface de développement, pas des contrôles de sécurité. L’activation en production est désactivée; une vraie autorisation devra être appliquée côté backend lors de l’authentification définitive.
- Au Dispatch, le responsable classe l’incident en choisissant une application et l’un de ses modules; cette classification et l’acteur sont enregistrés dans Oracle avant l’ouverture de la page de traitement.
- Le responsable dispose aussi de la page « Applications » (`/general/applications`) pour consulter le référentiel et y ajouter une application avec son domaine. Un code `APP-nnn` est généré par la séquence Oracle. L’ajout n’invente pas de module pour une nouvelle application.
- Fichiers concernés :
  - `gestion_incidents/src/main/resources/application-local.properties`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/incident/services/incident-api.service.ts`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/general/components/sidebar/sidebar.component.ts`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/incident/guard/incident-privileges.guard.ts`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/incident/general-routing.module.ts`
  - `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/controller/LocalIncidentQueueController.java`
  - `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/controller/LocalIncidentDispatchOptionsController.java`
  - `gestion_incidents/src/main/java/com/example/gestion_incidents/incident/service/IncidentDispatchOptionsService.java`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-treatment-queue/incident-treatment-queue.component.ts`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-dispatch/incident-dispatch.component.ts`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-direct-treatment/incident-direct-treatment.component.ts`
  - `modele-front-lucy/modele-front-lucy/src/app/modules/incident/pages/incident-direct-treatment/incident-direct-treatment.component.html`

## Schéma Oracle requis pour les pièces jointes

Le backend suppose que ces objets sont créés dans `GESTION_INCIDENTS`. `CONTENU` conserve les octets dans Oracle; aucune pièce jointe n’est stockée uniquement dans le navigateur.

```sql
CREATE TABLE PIECE_JOINTE (
    ID_PIECE       NUMBER         NOT NULL,
    NUMERO_INCIDENT NUMBER        NOT NULL,
    NOM            VARCHAR2(255),
    TYPE           VARCHAR2(100),
    TAILLE         NUMBER,
    AUTEUR         CHAR(6),
    CONTENU        BLOB           NOT NULL,
    CONSTRAINT PK_PIECE_JOINTE PRIMARY KEY (ID_PIECE),
    CONSTRAINT FK_PIECE_JOINTE_INCIDENT
        FOREIGN KEY (NUMERO_INCIDENT)
        REFERENCES INCIDENT (NUMERO)
);

CREATE SEQUENCE SEQ_PIECE_JOINTE
    START WITH 1
    INCREMENT BY 1
    NOCACHE
    NOCYCLE;
```

Le formulaire accepte PNG, JPG/JPEG, PDF, DOC et XLS, avec une limite de 10 Mo par fichier et 5 fichiers par demande. Les métadonnées et les octets des fichiers sont enregistrés dans la même transaction que la demande.

## Données de test des services SI

- Le script `gestion_incidents/src/main/resources/db/manual/002_seed_workflow_test_users.sql` prépare quatre services DRH sous la direction `08A1` : `SECUR`, `RESEA`, `DEV` et `BIDBA`.
- Il ajoute deux utilisateurs fictifs par service (matricules `000004` à `000011`) dans `INTRANET.ANNUAIRE_ELECTRONIQUE` et `GESTION_INCIDENTS.UTILISATEUR`, avec la fonction `TECH` et le rôle `ROLE_TECHNICIEN`.
- Les huit courriels utilisent le domaine réservé `.example.test`.
- Le script est réexécutable (MERGE) et contient trois sections à exécuter séparément, connecté successivement à `DRH`, `INTRANET` et `GESTION_INCIDENTS`. Il ne contient ni mot de passe ni instruction de GRANT.
- La section DRH crée la direction `08A1` et la fonction `TECH` seulement si elles sont absentes, puis crée/actualise les quatre services. Les comptes existants `000001` à `000003` ne sont pas modifiés. Le rôle applicatif `ROLE_TECHNICIEN` doit déjà exister.
- Les techniciens fictifs peuvent être sélectionnés pendant le dispatch si les scripts de données et de colonnes d’affectation ont été appliqués. Diagnostic, actions et communication sont enregistrés dans `TRAITEMENT` en profil local; l’attente, la résolution et la validation demandent encore leur persistance dédiée.
