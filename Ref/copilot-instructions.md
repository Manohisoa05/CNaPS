# Contexte du projet : Gestion des incidents SI (CNaPS)

> Ce fichier décrit le projet en entier. Lis-le avant de générer du code.
> Il peut être placé dans `.github/copilot-instructions.md` (Copilot le lit automatiquement)
> ou collé au début d'une conversation Copilot Chat.

## 1. Ce que tu dois faire

Tu m'aides à développer une application web de gestion des incidents du système d'information de la CNaPS.
L'application suit un incident du début à la fin : déclaration, qualification, affectation, traitement, résolution, validation, clôture, archivage.
Chaque action est tracée. Respecte strictement ce document. Si une règle manque ou est ambiguë, **pose-moi la question** au lieu d'inventer.

## 2. Technologies

- **Front-end** : Angular (composants, services, guards, intercepteurs HTTP, formulaires réactifs)
- **Back-end** : Java avec Spring Boot (API REST, couches Controller / Service / Repository, DTO)
- **Base de données** : Oracle
- **Langue** : noms de classes, méthodes et variables en anglais ; libellés affichés à l'utilisateur et commentaires en français

## 3. Base de données : existant et nouveau

Une base Oracle existe déjà. Je ne dois **pas** recréer ses données.

**Déjà existant, à utiliser en lecture seule (sélection dans des listes, aucun CRUD)** :
- départements, services, sections
- utilisateurs (matricule, nom, département, service, e-mail, statut)
- applications, modules, services SI
- types, catégories, sous-catégories d'incident
- équipes et techniciens

**À créer (nouvelles tables et CRUD)** :
- priorités, SLA, calendrier ouvré
- motifs d'attente, motifs de réouverture
- modèles de notifications
- rôles et permissions de l'application (liés aux utilisateurs existants)
- articles de la base de connaissances
- toutes les tables du cœur du projet (voir section 9)

Pour les données existantes : ne crée ni entité de création/modification ni écran d'administration. Crée seulement des entités en lecture et des endpoints `GET` pour alimenter les listes déroulantes.

## 4. Acteurs

| Acteur | Rôle dans l'application |
|---|---|
| Utilisateur / Demandeur | Crée et suit ses tickets, valide ou refuse la résolution |
| Service SI / Qualificateur | Qualifie, fixe la priorité et le SLA, affecte |
| Technicien / Équipe SI | Diagnostique, traite, documente, résout |
| Chef de service SI | Supervise, réaffecte, gère les escalades |
| DSI | Supervision globale, arbitrage des incidents majeurs |
| Auditeur | Consulte les historiques et journaux (lecture seule) |
| Administrateur | Rôles, permissions, paramétrage |

## 5. Rôles applicatifs (contrôle d'accès)

`ROLE_UTILISATEUR`, `ROLE_DEMANDEUR`, `ROLE_TECHNICIEN`, `ROLE_VALIDATEUR`, `ROLE_CHEF_SERVICE`, `ROLE_RESPONSABLE_SI`, `ROLE_DSI`, `ROLE_AUDITEUR`, `ROLE_ADMIN_APPLICATION`.

Règles :
- principe du moindre privilège et séparation des responsabilités
- toute action est contrôlée côté **back-end** (annotations de sécurité), pas seulement côté Angular
- la matrice précise rôle / module / action n'est pas encore validée : la rendre **configurable** (tables rôle, permission, module, action), ne pas la coder en dur

## 6. Cycle de vie d'un ticket (workflow des statuts)

```
BROUILLON → SOUMIS → QUALIFIÉ → AFFECTÉ → EN COURS ⇄ EN ATTENTE → RÉSOLU → À VALIDER → FERMÉ → ARCHIVÉ
```

Transitions autorisées (déduites du SFD, à implémenter dans un seul service de workflow) :

| De | Vers | Qui | Condition |
|---|---|---|---|
| BROUILLON | SOUMIS | Demandeur | Champs obligatoires remplis, numéro généré |
| SOUMIS | QUALIFIÉ | Qualificateur / Responsable SI | Type, catégorie, impact, urgence, priorité renseignés, SLA appliqué |
| QUALIFIÉ | AFFECTÉ | Qualificateur / Responsable SI | Équipe et technicien choisis |
| AFFECTÉ | EN COURS | Technicien | Prise en charge |
| EN COURS | EN ATTENTE | Technicien | **Motif obligatoire**, date de début, SLA suspendu |
| EN ATTENTE | EN COURS | Technicien | Reprise, SLA repris |
| EN COURS | RÉSOLU | Technicien | **Cause et solution obligatoires** (actions corrective et préventive) |
| RÉSOLU | À VALIDER | Système | Validation demandée au demandeur / validateur |
| À VALIDER | FERMÉ | Demandeur / Validateur | Résolution acceptée |
| À VALIDER | EN COURS | Demandeur / Validateur | Résolution refusée, **motif obligatoire** |
| FERMÉ | RÉOUVERT | Selon habilitation | **Motif obligatoire**, historique conservé |
| FERMÉ | ARCHIVÉ | Système / Admin | Conservation historique et statistique |

Règles de workflow :
- toute transition non listée est refusée avec un message clair
- chaque transition écrit une ligne dans l'historique (ancien statut, nouveau statut, acteur, date, motif)
- la réouverture et le refus de résolution conservent tout l'historique précédent
- **aucune suppression physique** d'un ticket clôturé

## 7. Priorités et SLA

| Code | Priorité | SLA indicatif | Exemple |
|---|---|---|---|
| CRI | Critique | 4 heures | Service essentiel totalement indisponible |
| URG | Urgent | 1 jour | Impact important |
| PRI | Prioritaire | 3 jours | Dysfonctionnement majeur |
| NOR | Normal | 5 jours | Impact limité |

Règles :
- le demandeur décrit le problème et choisit l'**impact** (Faible / Moyen / Fort / Critique)
- seul le responsable habilité attribue la **priorité finale** ; le système applique ensuite le SLA correspondant
- le SLA se calcule sur le **calendrier ouvré** et est **suspendu** pendant EN ATTENTE
- la priorité et le SLA sont **historisés** à chaque changement (ancienne valeur, nouvelle valeur, motif, acteur)
- les valeurs ci-dessus sont indicatives : les stocker dans les tables SLA (modifiables), jamais en dur dans le code

### Escalade SLA

| Seuil du délai consommé | Action |
|---|---|
| 50 % | Alerte informative |
| 80 % | Alerte au responsable et à l'équipe |
| 100 % | Dépassement SLA, notification à la hiérarchie |
| Après dépassement | Escalade selon la criticité |

Implémenter avec une tâche planifiée Spring (`@Scheduled`) qui recalcule les échéances et déclenche les alertes une seule fois par seuil.

## 8. Notifications

Canaux : dans l'application et par e-mail (selon configuration).

| Événement | Destinataire |
|---|---|
| Création | Demandeur |
| Affectation | Technicien / équipe |
| Réaffectation | Responsable concerné |
| SLA 80 % | Responsable |
| SLA dépassé | Responsable / hiérarchie |
| Mise en attente | Demandeur |
| Résolution | Demandeur |
| Validation requise | Demandeur / validateur |
| Réouverture | Équipe responsable |
| Incident majeur | Cellule de crise / DSI |

Les messages viennent des **modèles de notifications** stockés en base.

## 9. Modèle de données (tables à créer)

| Entité | Contenu principal |
|---|---|
| INCIDENT | Numéro unique, demandeur, application, module, type, catégorie, titre, description, impact, priorité, statut, lieu, dates |
| INCIDENT_HISTORIQUE | Acteur, date, action, ancienne valeur, nouvelle valeur, motif |
| AFFECTATION | Équipe, responsable, technicien, motif |
| TRAITEMENT | Diagnostic, actions (journal chronologique), commentaires demandeur, notes internes |
| RESOLUTION | Cause, solution, action corrective, action préventive |
| VALIDATION | Décision, commentaire, validateur, date |
| PIECE_JOINTE | Nom, type, taille, auteur |
| PRIORITE / SLA | Niveau, délai, calendrier, seuils |
| ARTICLE_KB | Symptôme, cause, diagnostic, solution, version, statut de validation |
| RCA | Cause racine, actions, responsable, échéance |
| ROLE / PERMISSION | Rôles, droits, modules, actions |
| MOTIF_ATTENTE / MOTIF_REOUVERTURE | Listes de motifs |
| MODELE_NOTIFICATION | Événement, canal, texte |

Format du numéro de ticket : exemple `INC-2026-000321` (INC, année, séquence Oracle sur 6 chiffres). Le format définitif n'est pas validé : le rendre facile à changer.

## 10. Écran « Demande d'incident »

| Champ | Obligatoire | Règle |
|---|---|---|
| Demandeur / Matricule | Oui | Repris du compte connecté, non modifiable |
| Département / Service | Oui | Sélection dans la base existante |
| Application / Service SI | Oui | Sélection dans la base existante |
| Module / Fonctionnalité | Selon l'application | Liste dépendante de l'application |
| Type d'incident | Oui | Sélection |
| Catégorie | Oui | Liste dépendante du type |
| Titre | Oui | Résumé du problème |
| Description | Oui | Description détaillée |
| Impact | Oui | Faible / Moyen / Fort / Critique |
| Pièce jointe | Non | Capture, document ou preuve |
| Lieu | Non | Site ou localisation |

Boutons : **Enregistrer en brouillon** et **Soumettre** (la soumission contrôle les champs obligatoires et génère le numéro).

## 11. Écran « Traitement incident »

Zones, dans cet ordre : en-tête (numéro, statut, priorité, SLA, dates), demandeur, contexte, affectation, diagnostic, actions (journal chronologique), communication (commentaires demandeur et notes internes), attente (motif, date de début, reprise), résolution (cause, solution, corrective, préventive), validation (décision, commentaire, validateur, date), historique (toutes les transitions et modifications).

## 12. Modules et menu

- **ACCUEIL** : tableau de bord (KPI, tickets critiques, alertes SLA, raccourcis)
- **DEMANDES** : nouvelle demande, mes tickets, tickets de mon service
- **TRAITEMENT INCIDENT** : nouveaux, à qualifier, à affecter, en cours, en attente, résolus, à valider, fermés, réouverts
- **INCIDENTS MAJEURS** : critiques, cellule de crise, RCA
- **SUIVI ET REPORTING** : tableau de bord, KPI, SLA, statistiques, rapports
- **BASE DE CONNAISSANCES** : articles, solutions, FAQ, problèmes récurrents
- **UTILISATEURS** : rôles, permissions, habilitations
- **PARAMÉTRAGE** : priorités, SLA, notifications, motifs
- **ADMINISTRATION** : audit, historique, sécurité

Le menu affiché dépend du rôle de l'utilisateur connecté.

## 13. Incidents majeurs, RCA et base de connaissances

- Un incident critique (CRI) peut activer le workflow « incident majeur » : cellule de crise, communication, retour à la normale, notification DSI.
- Une **RCA** (analyse de cause racine) peut être exigée pour les incidents critiques ou récurrents : cause racine, actions, responsable, échéance.
- Base de connaissances : identifier une solution réutilisable, rédiger (symptôme, cause, diagnostic, solution), faire valider, publier et versionner, associer l'article aux incidents similaires.

## 14. Indicateurs (KPI)

| KPI | Définition |
|---|---|
| Volume d'incidents | Nombre d'incidents sur une période |
| Taux SLA respecté | Incidents dans le SLA / incidents concernés × 100 |
| MTTA | Temps moyen avant prise en charge |
| MTTR | Temps moyen jusqu'à résolution |
| Taux de réouverture | Tickets réouverts / tickets résolus × 100 |
| Résolution au premier contact | Tickets résolus sans escalade / tickets traités × 100 |
| Incidents critiques | Nombre de tickets CRI |
| Incidents récurrents | Tickets liés à une même cause ou application |
| Backlog | Tickets ouverts à une date donnée |
| Âge moyen du backlog | Âge moyen des tickets non clôturés |

Axes d'analyse : département, service, application, équipe, technicien, catégorie, priorité, statut, période.
Exports Excel / PDF / CSV, **uniquement selon les habilitations**.
Calculer les KPI par requêtes Oracle à partir de l'historique des transitions (les dates exactes sont indispensables).

## 15. Audit et sécurité

À journaliser (acteur, date, ancienne et nouvelle valeur) : création, statut, priorité, affectation, résolution, validation, clôture, réouverture, paramétrage.
L'auditeur consulte ces journaux en lecture seule. Contrôler les exports. Protéger les pièces jointes et les données personnelles.

## 16. Règles de développement

1. Un seul **service de workflow** gère les changements de statut. Aucun autre code ne modifie directement le statut.
2. Contrôles d'accès et validations toujours **côté back-end**, en plus du front.
3. Toutes les valeurs métier (SLA, seuils, priorités, motifs, modèles de messages) viennent de la base, pas du code.
4. Chaque changement sensible écrit dans l'historique ou le journal d'audit.
5. API REST claire, DTO pour les échanges, pas d'entité JPA exposée directement.
6. Formulaires Angular réactifs avec les mêmes règles que le back-end (champs obligatoires, listes dépendantes).
7. Code simple et lisible, commenté en français, avec des noms explicites.
8. Ajoute des tests pour le workflow, le calcul du SLA et les droits d'accès.

## 17. Tests de recette à satisfaire

| ID | Test | Résultat attendu |
|---|---|---|
| T01 | Créer un incident | Numéro unique et statut initial correct |
| T02 | Champ obligatoire absent | Soumission refusée |
| T03 | Qualifier | Priorité et SLA calculés et enregistrés |
| T04 | Affecter | Équipe et technicien enregistrés |
| T05 | Mettre en attente | Motif obligatoire |
| T06 | Résoudre | Cause et solution enregistrées |
| T07 | Refuser la résolution | Retour EN COURS avec motif |
| T08 | Valider / fermer | Ticket FERMÉ et historique complet |
| T09 | Réouvrir | Ticket réouvert avec motif |
| T10 | Dépassement SLA | Alerte et escalade |
| T11 | Incident critique | Workflow majeur activable |
| T12 | Contrôle d'accès | Accès conforme au rôle |
| T13 | Audit | Changements sensibles historisés |
| T14 | Reporting | KPI cohérents |

Critères d'acceptation : identifiant unique pour chaque ticket, champs obligatoires contrôlés, transitions conformes au workflow, rôles qui bloquent les actions non autorisées, priorité et SLA historisés, alertes SLA fonctionnelles, résolution et validation traçables, clôture qui conserve l'historique, KPI cohérents avec les tickets, exports conformes aux habilitations.

## 18. Hors périmètre

Ne pas développer : intégration IAM / annuaire, intégration messagerie avancée, inventaire / CMDB, catalogue de services, gestion des problèmes, gestion des changements, détection automatique d'incidents, tableaux de bord Oracle BI, recherche intelligente dans la base de connaissances.

## 19. Points à confirmer (ne pas deviner)

Le SFD ne tranche pas ces points. Demande-moi avant de les coder, ou rends-les configurables :
- format définitif du numéro de ticket
- priorités et SLA définitifs, calendrier ouvré, règles de suspension du SLA
- matrice rôles / permissions
- règles de validation des incidents critiques
- durée de conservation et règles d'archivage
- canaux de notification retenus
- **Incohérence du SFD** : le cycle de vie parle de `SOUMIS`, mais le test T01 attend `NOUVEAU`, et le test T09 ainsi que le menu parlent de `RÉOUVERT` alors que ce statut n'est pas dans le cycle de vie. Je dois trancher ces noms de statuts avant de coder le workflow.
