# Plan d'implémentation — Espace Propriétaires La Rosière (v1)

Périmètre : **propriétaires uniquement**, avec leur système de comptes et le back-office
office de tourisme nécessaire pour les valider. Socio-pros et salariés sont hors périmètre v1
(voir §10). Base de l'analyse : [analyse-tignespro.md](analyse-tignespro.md).

**Durée : 9 semaines** (8 pour le socle, 1 pour le module des avantages décidé le 12 août).
Le compte à rebours part de la validation, pas de la rédaction : un feu vert avant fin août
met la mise en ligne **fin octobre**, soit six semaines avant l'ouverture de la saison, fixée
au 12 décembre 2026. C'est la fenêtre à tenir, et elle se referme d'une semaine par semaine
d'attente.

---

## 0. Décisions arrêtées — 5 août 2026

Quatre points tranchés, qui figent le périmètre v1 :

| Décision | Conséquence |
|---|---|
| **Aucun encaissement en v1** | Pas de Stripe, pas de facture émise par l'application, pas de rapprochement bancaire. L'ancien §7 sort du périmètre |
| **L'adhésion reste payante** | Le parcours va jusqu'à la validation par l'office ; l'appel de cotisation se fait hors application. Le modèle prévoit la place du paiement, sans l'implémenter |
| **Un logement peut avoir deux propriétaires** | Copropriété native dans le modèle de données dès la v1, avec invitation par e-mail. C'est structurant, pas cosmétique — voir §3 |
| **Les faiblesses de sécurité sont un livrable** | Les 13 constats de l'audit deviennent des critères de recette opposables, pas une intention — voir §8 |

**Le troc est neutre sur le calendrier** : la semaine libérée par l'encaissement finance la
copropriété et le durcissement sécurité. Les 8 semaines du socle tiennent.

### Décision complémentaire — 12 août 2026

| Décision | Conséquence |
|---|---|
| **Les propriétaires ont des entrées gratuites au cinéma et à la patinoire, et ces entrées sont comptées** | Nouveau module : droits par saison, billets à usage unique, contrôle à l'entrée. Le décompte impose un serveur qui arbitre — voir §7 bis. **+1 semaine**, la seule du plan qui ne se finance pas par un troc |

Le nombre d'entrées n'est pas figé dans le code : c'est un **paramètre de saison**, réglé par
l'office depuis son back-office. La maquette affiche cinq à titre d'exemple.

---

## 1. Hypothèses (à confirmer avant S1)

| # | Hypothèse | Si faux |
|---|---|---|
| H1 | 1 développeur à temps plein | à 2-3 j/semaine → 4 mois |
| H2 | Un interlocuteur OT disponible, recette sous 48 h | +2 à 3 semaines |
| H3 | ~~Paiement en ligne optionnel~~ → **tranché : hors périmètre v1** | — |
| H4 | Les propriétaires existants sont dans un fichier exploitable (Excel/CSV/export) | reprise à rechiffrer |
| H5 | CGU et mentions légales fournies par le client en S1 | mise en ligne décalée |
| H6 | Pas d'interconnexion avec un PMS/channel manager en v1 | à chiffrer séparément |
| H7 | Deux propriétaires maximum par logement, sans quote-part à gérer | au-delà, arbitrages à revoir |
| H8 | Les entrées gratuites sont un quota par saison et par propriétaire, identique pour tous | un barème variable (selon le nombre de lits, l'ancienneté) se code, mais il faut le connaître en S1 |
| H9 | Les points de contrôle (cinéma, patinoire) ont un accès Internet, même intermittent | sans réseau du tout, le contrôle en ligne est impossible : voir §7 bis, le mode dégradé devient obligatoire |
| H10 | Un billet consommé n'est pas remboursable, et une entrée non utilisée est perdue en fin de saison | si l'office veut des reports ou des annulations après scan, à préciser |

---

## 2. Stack

| Couche | Choix | Pourquoi |
|---|---|---|
| Framework | Next.js 15 (App Router, TypeScript) | SSR pour le SEO des pages publiques, Server Actions = CSRF natif |
| Base | PostgreSQL 16 | relationnel, contraintes fortes, hébergeable en France |
| ORM | Prisma | migrations versionnées, typage bout en bout |
| Auth | Better Auth (ou Auth.js) | sessions serveur, 2FA TOTP, pas de JWT en cookie |
| Hash | Argon2id | standard OWASP 2026 |
| Validation | Zod | schémas partagés client + serveur |
| UI | Tailwind + shadcn/ui | pas de CDN, tout bundlé |
| Fichiers | S3 compatible (Scaleway Object Storage, Paris) | URLs signées, jamais de bucket public |
| E-mails | Resend ou Brevo | transactionnel, domaine authentifié SPF/DKIM/DMARC |
| Hébergement | Scaleway Serverless / OVH, région France | RGPD, souveraineté, argument client fort |
| CI | GitHub Actions | lint, types, tests, migrations, déploiement |

Aucun script chargé depuis un CDN tiers. Aucune dépendance non verrouillée.

---

## 3. Modèle de données

Un seul `User`, des rôles — l'inverse des trois silos de TignesPro.

```prisma
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  passwordHash  String?
  emailVerified DateTime?
  firstName     String
  lastName      String
  phone         String?
  locale        String    @default("fr")
  mfaSecret     String?   // obligatoire pour OT_ADMIN
  lastLoginAt   DateTime?
  createdAt     DateTime  @default(now())
  memberships   Membership[]
  sessions      Session[]
}

model Membership {
  id             String       @id @default(cuid())
  userId         String
  organizationId String
  role           Role         // OWNER | OT_STAFF | OT_ADMIN
  user           User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  organization   Organization @relation(fields: [organizationId], references: [id])
  @@unique([userId, organizationId])
}

model Organization {
  id            String   @id @default(cuid())
  type          OrgType  // INDIVIDUAL | COMPANY
  legalName     String?  // raison sociale si COMPANY
  siret         String?  @unique
  addressLine   String
  postalCode    String
  city          String
  country       String   @default("FR")
  createdAt     DateTime @default(now())
  memberships   Membership[]
  properties    PropertyOwner[]
  adhesions     Adhesion[]
}

// Un logement n'appartient plus à une organisation, mais à une ou deux.
// C'est le changement structurant de la décision du 5 août.
model PropertyOwner {
  propertyId     String
  organizationId String
  role           OwnerRole    // PRIMARY | CO_OWNER
  invitedAt      DateTime?    // copropriétaire invité par e-mail
  acceptedAt     DateTime?    // tant que null, il ne voit rien
  property       Property     @relation(fields: [propertyId], references: [id], onDelete: Cascade)
  organization   Organization @relation(fields: [organizationId], references: [id])
  @@id([propertyId, organizationId])
  @@index([organizationId])
}

model Property {
  id                 String        @id @default(cuid())
  owners             PropertyOwner[]
  name               String
  addressLine        String
  postalCode         String
  city               String
  registrationNumber String?       // n° d'enregistrement mairie
  classification     Int?          // 0-5 étoiles meublé de tourisme
  capacity           Int
  surfaceM2          Int?
  status             PropertyStatus // DRAFT | SUBMITTED | VALIDATED | REJECTED
  reviewNote         String?        // motif de rejet, visible des propriétaires
  documents          Document[]
}
// `Session` est fourni par la lib d'auth (Better Auth / Auth.js) — non détaillé ici.

model Document {
  id         String       @id @default(cuid())
  propertyId String
  type       DocumentType // CLASSIFICATION | INSURANCE | DIAGNOSTIC | PHOTO | OTHER
  fileKey    String       // clé S3, jamais une URL publique
  fileName   String
  mimeType   String
  sizeBytes  Int
  status     DocStatus    // PENDING | VALIDATED | REJECTED
  expiresAt  DateTime?    // assurance, diagnostic
  uploadedAt DateTime     @default(now())
  property   Property     @relation(fields: [propertyId], references: [id], onDelete: Cascade)
}

model Adhesion {
  id             String        @id @default(cuid())
  organizationId String
  season         String        // "2026-2027"
  status         AdhesionStatus // DRAFT | SUBMITTED | VALIDATED | REJECTED
  cguVersion     String
  cguAcceptedAt  DateTime?
  organization   Organization  @relation(fields: [organizationId], references: [id])
  // Encaissement hors périmètre v1. Les colonnes ci-dessous sont prévues mais
  // ni alimentées ni exposées : elles évitent une migration douloureuse en phase 2.
  amountCents    Int?
  invoiceRef     String?
  paidAt         DateTime?
  @@unique([organizationId, season])
}

// ------------------------------------------------------------------ avantages
// Le quota est porté par la saison, pas par le code : l'office le règle sans déploiement.
model Advantage {
  id          String   @id @default(cuid())
  season      String   // "2026-2027"
  code        String   // "RUITOR" | "PATINOIRE"
  label       String   // "Espace Culturel Le Ruitor"
  description String
  quota       Int      // entrées gratuites par propriétaire et par saison
  activeFrom  DateTime
  activeTo    DateTime
  tickets     Ticket[]
  @@unique([season, code])
}

model Ticket {
  id             String       @id @default(cuid())
  advantageId    String
  organizationId String       // à qui appartient l'entrée consommée
  issuedByUserId String       // lequel des deux copropriétaires l'a édité
  token          String       @unique   // 20 caractères, tirés d'un CSPRNG, sans signification
  status         TicketStatus // ISSUED | CONSUMED | CANCELLED
  issuedAt       DateTime     @default(now())
  consumedAt     DateTime?
  consumedByUserId String?    // l'agent qui a scanné
  consumedAtPoint  String?    // "RUITOR" | "PATINOIRE"
  advantage      Advantage    @relation(fields: [advantageId], references: [id])
  @@index([organizationId, advantageId, status])
}

// Toute présentation d'un code, acceptée ou non. Sert au comptage, aux litiges,
// et à repérer une salve de codes inventés.
model ScanAttempt {
  id            String   @id @default(cuid())
  token         String
  point         String
  verdict       String   // OK | ALREADY_USED | UNKNOWN | OUT_OF_SEASON | CANCELLED
  scannedByUserId String?
  createdAt     DateTime @default(now())
  @@index([token, createdAt])
}

model AuditLog {
  id         String   @id @default(cuid())
  userId     String?
  action     String   // LOGIN, LOGIN_FAILED, PROPERTY_SUBMITTED, DOC_VALIDATED…
  targetType String?
  targetId   String?
  ip         String?
  userAgent  String?
  createdAt  DateTime @default(now())
  @@index([userId, createdAt])
}
```

Règle d'autorisation unique : **toute requête est filtrée par les `organizationId` accessibles
via `Membership`**. Un helper `getScopedOrgIds(session)` est le seul point d'entrée — jamais de
requête Prisma sans ce filtre dans le code propriétaire. Pour les logements, le filtre passe
par `PropertyOwner` avec `acceptedAt != null` : une invitation non acceptée ne donne accès à rien.

### 3.1 Copropriété — les règles à trancher, et celles que je propose

Deux propriétaires sur un logement, ce n'est pas une ligne de plus dans une table. Ça ouvre
une série de questions dont aucune n'a de réponse évidente. Voici mes recommandations, à valider
par l'office :

| Question | Proposition | Pourquoi |
|---|---|---|
| Qui peut modifier le logement ? | **Les deux** | Le cas courant est un couple. Bloquer l'un des deux, c'est un appel au secrétariat par semaine |
| Qui peut envoyer le dossier ? | **Les deux** | Idem. L'action est tracée au nom de celui qui l'a faite |
| Qui reçoit les notifications ? | **Les deux, systématiquement** | Sinon l'un valide, l'autre l'apprend en janvier |
| Qui peut inviter un copropriétaire ? | Le `PRIMARY` uniquement | Sinon on peut se faire rattacher n'importe qui |
| Qui peut retirer un copropriétaire ? | Le `PRIMARY`, avec trace au journal | Cas de séparation, de vente |
| Que voit un invité qui n'a pas accepté ? | **Rien** | `acceptedAt` null = aucun accès |
| Et si le `PRIMARY` supprime son compte ? | Le `CO_OWNER` est promu `PRIMARY` | Sinon le logement devient orphelin |

**Le parcours d'invitation** est le vrai travail ajouté : le propriétaire saisit l'e-mail du
second, celui-ci reçoit un lien à usage unique valable 30 jours, crée son compte ou se connecte,
et est rattaché. S'il n'accepte jamais, le logement reste au premier — rien ne se bloque.

Ce que je n'implémente pas en v1, faute de besoin exprimé : les quote-parts, les mandats de
gestion, et plus de deux propriétaires. Le modèle les accueille, l'interface ne les expose pas.

---

## 4. Cartographie des écrans

### Public
- `/` — présentation de l'espace propriétaire, avantages, CTA
- `/inscription` — création de compte
- `/connexion` — un seul formulaire, **pas de choix de profil** (le rôle est déduit du compte)
- `/mot-de-passe-oublie` et `/reinitialiser/[token]`
- `/cgu`, `/mentions-legales`, `/confidentialite`

### Espace propriétaire (`/espace`)
- `/espace` — tableau de bord : statut du dossier, échéances, actions requises
- `/espace/profil` — coordonnées, mot de passe, langue, suppression de compte (RGPD)
- `/espace/logements` — liste + statut de chaque bien
- `/espace/logements/nouveau` et `/espace/logements/[id]` — formulaire complet
- `/espace/logements/[id]/documents` — dépôt, statut, motif de rejet
- `/espace/logements/[id]/proprietaires` — copropriétaire : inviter, retirer, voir le statut
- `/invitation/[token]` — page d'acceptation d'une invitation de copropriété
- `/espace/adhesion` — CGU de la saison, envoi du dossier, suivi (sans paiement en v1)
- `/espace/avantages` — droits de la saison, édition d'un billet, code QR, annulation

### Back-office OT (`/admin`, rôles `OT_STAFF` / `OT_ADMIN`)
- `/admin` — file d'attente des dossiers à traiter
- `/admin/proprietaires` — recherche, filtres, fiche détaillée
- `/admin/logements/[id]` — validation / rejet motivé document par document
- `/admin/adhesions` — suivi par saison, relances
- `/admin/exports` — CSV propriétaires / logements / adhésions
- `/admin/journal` — journal d'audit (`OT_ADMIN` uniquement)
- `/admin/controle` — contrôle à l'entrée : scan, verdict, historique du poste
- `/admin/avantages` — barème de la saison : équipements, quotas, dates d'ouverture

---

## 5. Planning semaine par semaine

### S1 — Cadrage et socle
- Atelier client : périmètre figé, récupération CGU / barème / fichier des propriétaires existants
- Maquettes des 6 écrans clés (dashboard, logement, documents, adhésion, file OT, fiche OT)
- Repo, Next.js + TypeScript + Tailwind, Postgres, Prisma, schéma initial migré
- CI GitHub Actions (lint, typecheck, tests, migration), environnement de recette en ligne
- **Livrable : maquettes validées + environnement de recette accessible au client**

### S2 — Authentification
- Inscription particulier / société, vérification d'e-mail obligatoire
- Connexion, déconnexion, sessions serveur (`Secure`, `HttpOnly`, `SameSite=Lax`, rotation)
- Mot de passe oublié avec jeton à usage unique, expiration 1 h, **réponse identique que l'e-mail existe ou non**
- Argon2id, politique de mot de passe côté serveur (12 caractères min, vérification contre HIBP k-anonymity)
- **Le coller est autorisé** sur tous les champs mot de passe
- **Livrable : parcours de compte complet et testé**

### S3 — Sécurité et rôles
- Rôles et `Membership`, helper d'autorisation centralisé, middleware de protection des routes
- 2FA TOTP pour `OT_ADMIN` et `OT_STAFF`
- Rate-limiting : connexion (5/15 min/IP + par compte), inscription, reset
- Turnstile Cloudflare sur inscription et reset
- En-têtes : CSP stricte, HSTS `includeSubDomains; preload`, `Referrer-Policy`, `Permissions-Policy`, redirection 301 HTTP→HTTPS
- `AuditLog` branché sur les événements sensibles + e-mail « nouvelle connexion »
- **Livrable : checklist sécurité §8 cochée et vérifiable**

### S4 — Logements
- CRUD logements avec validation Zod partagée
- Champs métier : classement meublé, n° d'enregistrement mairie, capacité, surface
- Machine à états `DRAFT → SUBMITTED → VALIDATED / REJECTED` avec motif de rejet
- Tableau de bord propriétaire : ce qu'il reste à faire, en clair

### S5 — Documents
- Upload S3 par URL pré-signée, types et taille contrôlés **côté serveur**
- Scan antivirus (ClamAV ou service managé) avant mise à disposition
- Validation document par document côté OT, dates d'expiration (assurance, diagnostic)
- Aucune URL publique : accès par URL signée courte durée, vérifiée contre le `Membership`

### S6 — Back-office OT et copropriété
- File d'attente, recherche, filtres, fiche propriétaire consolidée
- Validation / rejet motivé, notification automatique **aux deux** propriétaires
- Exports CSV
- Adhésion : CGU versionnées, acceptation horodatée, suivi par saison — **sans encaissement**
- **Copropriété** : invitation par e-mail, acceptation, retrait, promotion du `CO_OWNER`,
  règles du §3.1 et leurs tests (semaine financée par l'abandon du module de paiement)

### S7 — Finitions
- E-mails transactionnels (10 gabarits) FR/EN, SPF/DKIM/DMARC configurés
- Internationalisation FR/EN complète
- RGPD : bandeau de consentement **avant** tout tracker, export de ses données, suppression de compte
- Responsive et accessibilité (navigation clavier, contrastes, labels)
- Tests end-to-end Playwright sur les 5 parcours critiques

### S7 bis — Avantages, billets et contrôle *(semaine ajoutée le 12 août)*
- Barème de saison paramétrable par l'office : équipements, quotas, période de validité
- Édition d'un billet, décompte du droit, annulation d'un billet non consommé
- Génération du code QR **côté serveur**, rendu en SVG dans la page et dans l'e-mail
- Écran de contrôle : scan, consommation atomique, verdicts distincts, journalisation de
  **toutes** les présentations, y compris les refus
- Mode dégradé hors ligne (voir §7 bis) et sa synchronisation
- **Livrable : un billet édité, scanné, refusé au second passage, et le compteur qui suit**

### S8 — Recette et mise en production
- Reprise des données propriétaires existants (script de migration + e-mail d'activation)
- Recette client, correctifs
- Domaine, certificats, sauvegardes automatiques quotidiennes + test de restauration
- Documentation d'exploitation + formation de l'équipe OT (2 h)
- **Mise en ligne**

---

## 6. Tests

- **Unitaires** (Vitest) : autorisation, machines à états, validation Zod
- **Intégration** : chaque route API/Server Action avec un utilisateur non autorisé → doit échouer
- **E2E** (Playwright) : inscription → logement → documents → envoi → validation OT → notification
- **Sécurité** : `npm audit` + Dependabot en CI, revue OWASP ASVS niveau 2 sur les points §8

Test non négociable en CI : **un propriétaire A ne doit jamais accéder aux données de B.**
Un cas de test par route.

---

## 7. Encaissement — reporté en phase 2 *(décision du 5 août)*

L'adhésion reste payante, mais l'application ne l'encaisse pas en v1. Le parcours s'arrête à
la validation par l'office, qui appelle la cotisation par ses moyens habituels.

Ce qui reste malgré tout fait en v1, parce que ça ne coûte presque rien et que ça évite une
reprise douloureuse :

- les colonnes `amountCents`, `invoiceRef`, `paidAt` existent en base, non alimentées ;
- le statut `VALIDATED` de l'adhésion est le point d'accroche naturel du futur paiement ;
- l'écran d'adhésion affiche « cotisation appelée par l'office », pas un bouton mort.

Ce qui est explicitement exclu : Stripe, facture PDF, numérotation séquentielle, TVA,
relances automatiques, rapprochement bancaire. **Chiffrage phase 2 : 3 à 4 semaines**
selon la complexité du barème — c'est la numérotation légale et les avoirs qui coûtent,
pas le paiement lui-même.

---

## 7 bis. Avantages, billets et contrôle à l'entrée *(décision du 12 août)*

Chaque propriétaire adhérent dispose d'un nombre d'entrées gratuites par saison, au cinéma
de l'Espace Culturel Le Ruitor et à la patinoire de La Rosière. **Ces entrées sont comptées.**
C'est cette phrase qui commande toute l'architecture du module : dès qu'on compte, il faut un
arbitre, et l'arbitre ne peut pas être le navigateur du propriétaire.

### 7 bis.1 Le principe : le billet ne prouve rien, il désigne

Un billet est un **jeton opaque** de vingt caractères, tiré d'un générateur cryptographique.
Il ne contient ni le nom du porteur, ni le solde restant, ni une signature vérifiable hors
ligne. Il ne dit rien : il désigne une ligne en base, et c'est le serveur qui répond.

Ce choix est ce qui rend la fraude sans objet :

| Tentative | Ce qui se passe |
|---|---|
| Fabriquer un code QR | Le jeton n'existe pas en base : refus, et la tentative est journalisée |
| Modifier le compteur dans le navigateur | L'écran affiche ce qu'il veut ; l'API refuse au-delà du quota |
| Photographier le billet d'un ami | Le premier scan consomme le billet, le second est refusé — la copie ne crée rien |
| Rejouer un billet déjà scanné | Refus, avec la date et le poste du premier passage |
| Scanner deux fois dans la même seconde | Un seul passe : la consommation est une transaction avec verrou |

### 7 bis.2 La consommation, en une transaction

Le point qui fait la différence entre « compté » et « à peu près compté » :

```sql
-- verrou de ligne : la seconde requête attend, puis constate que le billet est consommé
BEGIN;
SELECT status FROM "Ticket" WHERE token = $1 FOR UPDATE;
UPDATE "Ticket" SET status = 'CONSUMED', "consumedAt" = now(), ...
  WHERE token = $1 AND status = 'ISSUED';
COMMIT;
```

L'`UPDATE` conditionné sur `status = 'ISSUED'` est la garantie : si zéro ligne est modifiée,
le billet était déjà consommé, et le verdict est un refus. Même chose à l'édition, où le
quota se vérifie **dans la transaction** qui crée le billet, jamais avant.

### 7 bis.3 Le matériel de contrôle

L'écran `/admin/controle` est une page web ordinaire, ouverte sur le poste d'accueil. Deux
manières de saisir, sans application à installer :

- **Douchette USB** en mode clavier — elle tape le code dans le champ et valide. C'est le plus
  fiable, le moins cher, et ça ne dépend d'aucune permission de navigateur. **Recommandé.**
- **Appareil photo du téléphone**, via l'API de détection de codes-barres intégrée aux
  navigateurs récents. Pratique en secours ou pour un contrôle mobile.

Le code QR affiché par le propriétaire est généré **côté serveur** et rendu en SVG : la page
n'embarque pas de bibliothèque de génération, et le même code part dans l'e-mail de
confirmation.

### 7 bis.4 Le réseau tombe. Et alors ?

C'est la seule question de ce module qui n'a pas de bonne réponse, seulement un arbitrage à
rendre par l'office :

| Option | Ce qu'on y gagne | Ce qu'on y perd |
|---|---|---|
| **Refus hors ligne** | Aucun billet ne passe deux fois, jamais | Une coupure de réseau bloque l'entrée : file d'attente, propriétaires mécontents |
| **Acceptation optimiste** | L'entrée fonctionne toujours | Pendant la coupure, un billet peut passer deux fois ; la synchronisation le détecte après coup, trop tard |

**Ma recommandation : l'acceptation optimiste, avec une trace.** Le poste garde en mémoire les
jetons consommés hors ligne et les rejoue dès le retour du réseau. Les doublons apparaissent
alors au journal, et l'office décide s'il veut en faire quelque chose. Refuser l'entrée à un
propriétaire à cause d'une coupure de réseau coûte plus cher à l'office qu'une place de cinéma
offerte deux fois — et un cinéma de station n'a pas de siège numéroté à protéger.

Cette recommandation suppose H9 : un réseau qui revient. Si un point de contrôle n'a jamais
de réseau, le mode dégradé cesse d'être un secours et devient le fonctionnement normal, avec
un jeu de jetons pré-synchronisé chaque matin. À dire en S1, pas en décembre.

### 7 bis.5 Ce que le module ne fait pas en v1

Pas de transfert de billet à un tiers, pas de billet nominatif avec photo, pas de contrôle
d'identité à l'entrée. Un billet appartient à l'organisation du propriétaire ; si la famille
se le passe, le quota s'épuise pareil, ce qui suffit à protéger l'office. Le jour où l'usage
montre un abus, le nom du porteur s'affiche sur le billet en une journée de travail.

---

## 8. Sécurité — livrable à part entière *(décision du 5 août)*

Ce tableau n'est pas une liste d'intentions : **c'est le référentiel de recette**. Chaque ligne
est vérifiable par le client ou par un tiers, et conditionne la réception du lot.

| Constat TignesPro | Traitement v1 | Comment le vérifier |
|---|---|---|
| Aucun CSRF | Server Actions Next.js + `SameSite=Lax` | Rejouer un POST depuis un autre domaine : doit échouer |
| Aucun rate-limiting / CAPTCHA | 5 essais / 15 min par IP **et** par compte, + Turnstile | 6 connexions ratées d'affilée : la 6ᵉ est refusée |
| Cookies d'identité maison | Session serveur, jeton haché en base, rotation, révocation | « Se déconnecter » puis rejouer le cookie : refusé |
| Pas de CSP | CSP stricte avec nonce | En-têtes de réponse, ou observatory.mozilla.org |
| HTML injecté via `.load()` | React, échappement par défaut, aucun `dangerouslySetInnerHTML` | Nom de logement `<script>alert(1)</script>` : affiché tel quel |
| HTTP répond 200 | Redirection 301 + HSTS preload | `curl -I http://…` doit renvoyer 301 |
| CDN sans SRI | Zéro CDN, tout bundlé | Onglet réseau : aucun domaine tiers |
| Pas de 2FA | TOTP obligatoire pour les comptes OT | Créer un compte agent : 2FA imposée à la première connexion |
| Dépendances de 2016 | Lockfile + Dependabot + CI qui casse sur vulnérabilité haute | Journal de la CI |
| Coller bloqué sur le mot de passe | Autorisé | Coller dans le champ : ça fonctionne |
| GA4 avant consentement | Aucun traceur avant accord explicite ; refuser et retirer coûtent un clic, comme accepter | Onglet réseau avant de cliquer sur le bandeau : aucune requête. Puis retirer son accord depuis la page « Traceurs » : le script ne revient pas |
| Contrôle d'accès tardif (200 vide) | Middleware + filtre par organisation ; 401 et 403 distingués | Propriétaire A appelant l'URL d'un bien de B : 403, pas une page vide |
| Validation client seule | Zod partagé, revalidation serveur obligatoire | POST direct hors formulaire avec données invalides : rejeté |

### 8.1 Ce qui va au-delà de l'audit

Trois points qui ne corrigent pas TignesPro mais qui relèvent du même niveau d'exigence :

- **Cloisonnement entre propriétaires** — un test automatisé par route, en CI : le propriétaire A
  ne doit jamais lire une donnée de B. C'est le test qui ne doit jamais devenir rouge.
- **Journal d'audit** — connexion, échec de connexion, accès refusé, dépôt, validation, refus,
  invitation, retrait de copropriétaire. Conservé après suppression de compte, anonymisé.
- **Revue OWASP ASVS niveau 2** sur les points ci-dessus avant la mise en ligne.

---

### 8.2 Critères propres aux billets

| Ce qu'on garantit | Comment le vérifier |
|---|---|
| Un billet ne passe qu'une fois | Scanner deux fois le même code : le second est refusé, avec la date du premier |
| Deux scans simultanés ne passent pas tous les deux | Envoyer deux requêtes concurrentes sur le même jeton : une seule réussit (verrou en base) |
| Le quota ne se contourne pas depuis le navigateur | Appeler l'API d'édition en boucle hors interface : refus au-delà du quota de la saison |
| Le code QR ne porte aucun droit | Décoder un billet : on n'y trouve qu'un jeton opaque, ni identité, ni solde, ni signature réutilisable |
| Un jeton n'est pas devinable | 20 caractères d'un générateur cryptographique : deviner un billet valable est hors de portée d'une attaque par essais, et le rate-limiting du poste de contrôle ferme la porte |
| Toute présentation est tracée | Scanner un code inventé : la tentative apparaît au journal avec l'heure et le poste |

## 9. Ce que le client doit fournir, et quand

| Quand | Quoi | Bloque |
|---|---|---|
| S1 | CGU, mentions légales, politique de confidentialité | S6, S7 |
| S1 | Barème d'adhésion saison 2026-2027 | S6, module paiement |
| S1 | Fichier des propriétaires existants | S8 |
| S1 | Charte graphique, logo, visuels | S1 (maquettes) |
| S2 | Accès domaine / DNS | S7 (e-mails), S8 |
| S6 | Liste des documents obligatoires par logement | S5, S6 |
| S6 | Comptes des agents OT à créer | S8 |
| S1 | **Barème des entrées gratuites** : combien par équipement, par saison | S7 bis |
| S6 | **Points de contrôle** : qui scanne, sur quel matériel, avec quel réseau | S7 bis |

---

## 10. Hors périmètre v1 (phase 2 à chiffrer)

**Encaissement de l'adhésion** (§7, 3 à 4 semaines), espace socio-professionnel (vérification
SIRET réelle, multi-établissements, adhésion facturée), espace salariés, interconnexion
PMS/channel manager, calcul et déclaration de la taxe de séjour, statistiques d'occupation,
application mobile.

Et, à ne pas laisser entrer par la porte de derrière : **la centrale de réservation**. Rien de
tel n'existe chez TignesPro — leurs CGU précisent qu'ils ne sont pas partie aux contrats conclus
entre membres et professionnels, et aucun flux de séjour ne transite par leur plateforme. Si
La Rosière en veut une, c'est un autre métier : disponibilités, mandats, encaissement pour compte
de tiers, reversements. À chiffrer séparément, jamais à glisser dans ce périmètre.

---

## 11. Risques

| Risque | Probabilité | Parade |
|---|---|---|
| Retard de fourniture des CGU / barème | Élevée | Adhésion découplée : le reste livre sans elle |
| Recette client lente | Moyenne | Environnement de recette dès S1, validation en continu |
| Données existantes inexploitables | Moyenne | Vue du fichier en S1, sinon inscription manuelle des propriétaires |
| Périmètre qui s'élargit en cours de route | Élevée | Périmètre figé en S1, tout ajout part en phase 2 |
| Fenêtre saison d'hiver manquée | Faible si S1 tient | Mise en ligne visée fin octobre, six semaines de marge avant le 12 décembre |
| Réseau absent au point de contrôle | Moyenne | Mode dégradé décidé en S1, pas découvert le soir de l'ouverture — voir §7 bis |
| Barème des entrées non arrêté | Moyenne | Le module se livre avec un quota paramétrable : l'office peut le fixer après la mise en ligne, sans déploiement |

---

## 12. Étape suivante

**Aujourd'hui**, le dossier est complet : cette analyse, ce plan, et une maquette manipulable
en ligne qui rend vérifiables les promesses du §8. Rien n'attend plus de mon côté.

**La prochaine décision appartient à l'office** : valider le périmètre et le calendrier. Tant
qu'elle n'est pas prise, construire le socle revient à parier sur un périmètre non confirmé.

**Au feu vert**, S1 démarre : dépôt, Next.js, Postgres, schéma Prisma, intégration continue et
environnement de recette en ligne — plus les trois ouvertures de comptes du §12 ter, qui
conditionnent tout le reste et prennent une demi-journée si les accès DNS sont disponibles.

---

## 12 bis. La maquette qui accompagne ce plan

`index.html` est une **maquette de façade** : un fichier unique, à ouvrir dans un navigateur.
Elle sert à discuter d'écrans réels plutôt que de captures, et à rendre vérifiables les
promesses de sécurité du §8 au lieu de les affirmer.

**Ce qu'elle démontre, et qu'on peut manipuler :**

| Parcours | Ce qu'on peut y faire |
|---|---|
| Créer un compte | Particulier ou société, jauge de robustesse, contrôle de clé SIRET, collage autorisé |
| Vérification SIRET | Appel réel à l'annuaire des entreprises de l'État, avec repli local si l'API tombe |
| Déclarer un logement | Le dossier se crée en brouillon, avec ses pièces manquantes et son avancement |
| Copropriété | Invitation d'un second propriétaire, et l'écran que reçoit l'invité avant d'accepter |
| Instruction côté office | Validation et **refus motivé** d'une pièce, motif lu par le propriétaire |
| Échéances | Une assurance qui n'atteint pas l'ouverture de la saison remonte des deux côtés |
| Annuaire | Recherche sans accent, filtres par adhésion, fiche consolidée, exports CSV réels |
| Connexion | Cinq échecs puis verrouillage de quinze minutes, décompte à l'écran |
| Mot de passe oublié | Réponse identique que l'adresse existe ou non, testable en un clic |
| Second facteur | Code à six chiffres imposé à l'agent, sans contournement |
| Traceurs | Compteur des requêtes réellement parties du navigateur, lu dans l'API de performance |
| Avantages | Édition d'un billet, code QR généré dans la page, décompte du droit, annulation |
| Contrôle à l'entrée | Scan d'un billet : autorisé, puis refusé au second passage, refusé si le code est inventé |
| Recette | Les treize constats du §8, dont sept avec un bouton qui ouvre la preuve |

**Ce qu'elle ne fait pas, et ne peut pas faire.** Aucun serveur, aucune base, aucun service
d'envoi : **aucun e-mail ne part**, aucun compte n'est créé, rien n'est conservé après la
fermeture de l'onglet. Publiée sur GitHub Pages — de l'hébergement de fichiers statiques —
elle n'exécute par construction aucun code côté serveur. Les six constats restants du §8
(CSRF, CSP, redirection HTTP, dépendances, cookie rejoué, revalidation serveur) se vérifient
sur l'environnement de recette de la S1, pas avant.

Les données y sont fictives, y compris les huit propriétaires de l'annuaire, qui n'existent
que pour donner de la matière à la recherche et à l'export.

---

## 12 ter. De la maquette au site en ligne

La maquette est une façade : un fichier, aucun serveur. Voici ce qu'il faut monter pour que
la même chose fonctionne pour de vrai, et qui fournit quoi.

### Ce qu'il faut ouvrir, et qui le fait

| Quoi | Qui | Coût indicatif | Bloque |
|---|---|---|---|
| **Nom de domaine** (ex. `espace.larosiere.net`, sous-domaine du domaine existant) | l'office | inclus s'il s'agit d'un sous-domaine | tout le reste |
| **Accès DNS** pour poser les enregistrements | l'office | — | e-mails, certificat |
| **Hébergement applicatif** Scaleway Serverless Containers, Paris | vous ou l'office | ~10-20 €/mois | déploiement |
| **PostgreSQL managé**, Paris | idem | ~15-25 €/mois | tout |
| **Stockage objet** S3, Paris, bucket privé | idem | ~1-5 €/mois selon volume | dépôt des pièces |
| **Service d'envoi** Resend ou Brevo, domaine authentifié | idem | 0 à 20 €/mois | vérification d'adresse, notifications |
| **Supervision** (Sentry ou équivalent, hébergement UE) | vous | 0 à 25 €/mois | rien, mais on vole à vue sans |

**Ordre de grandeur : 30 à 70 € par mois**, tout compris, pour quelques milliers de
propriétaires. Un compte doit être ouvert par l'office, ou par vous à son nom : c'est lui qui
doit rester propriétaire des accès le jour où il change de prestataire. **Ne pas héberger la
production sur un compte personnel de développeur** — c'est la dépendance qui coince trois ans
plus tard.

### Le domaine et les e-mails

Trois enregistrements DNS à poser avant la première invitation :

- **SPF** — autorise le service d'envoi à écrire au nom du domaine ;
- **DKIM** — signe chaque message, pour qu'il ne soit pas réécrit en route ;
- **DMARC** — dit aux serveurs destinataires quoi faire d'un message non conforme, et où
  envoyer les rapports.

Sans ces trois-là, les e-mails d'activation partent en indésirables, et la reprise du fichier
des propriétaires (§13.8) échoue silencieusement : deux cents propriétaires qui ne reçoivent
rien, et un secrétariat qui reçoit deux cents appels.

### Le déploiement

- **Une branche, un environnement.** `main` déploie en recette à chaque fusion ; une étiquette
  de version déploie en production, sur décision humaine.
- **La CI refuse de déployer** si le lint, les types, les tests ou les migrations échouent.
- **Migrations jouées automatiquement** au déploiement, jamais à la main (§13.4).
- **Retour arrière** : la version précédente reste déployable en un clic. Une migration
  destructive se fait en deux temps pour que ce retour reste possible.

### Ce qu'il faut surveiller

| Quoi | Pourquoi |
|---|---|
| Erreurs applicatives | Une erreur silencieuse en production est une donnée perdue |
| Taux d'échec des e-mails | Un domaine qui se met à rebondir se répare en heures, pas en semaines |
| Temps de réponse de l'API de contrôle | Un scan qui prend trois secondes fait une file à l'entrée du cinéma |
| Sauvegardes | Une sauvegarde qui n'a pas tourné se voit le jour où on en a besoin, sauf si on la surveille |
| Certificat TLS | Renouvellement automatique, avec alerte à J-15 s'il ne s'est pas renouvelé |

### Après la mise en ligne

Ce plan couvre la livraison, pas la vie de l'application. À prévoir séparément, et à dire
maintenant plutôt qu'à découvrir en janvier :

- **Correctifs de sécurité des dépendances** : quelques heures par mois, à budgéter ;
- **Une astreinte pendant la saison**, ou au minimum un engagement de délai de réponse — un
  espace propriétaires en panne le 20 décembre n'attend pas le lundi ;
- **La saison suivante** : ouvrir 2027-2028, reconduire les barèmes, archiver l'ancienne.
  Une demi-journée par an si le modèle est bien fait, ce qu'il est : la saison est une donnée,
  pas une constante dans le code.

---

## 13. Base de données — exploitation

### 13.1 Dimensionnement réel

Il faut le dire clairement : **le volume est trivial.** Quelques milliers de propriétaires,
quelques milliers de logements, un historique d'adhésions par saison. On parle de
**moins d'1 Go de base** en régime établi. Les fichiers (photos, diagnostics, assurances) ne
sont **pas** en base : seules les clés S3 le sont — quelques dizaines de Go côté stockage objet.

Conséquence : la contrainte n'est pas la performance, c'est la **disponibilité, la sauvegarde
et la localisation des données**. On dimensionne petit et on paie la fiabilité, pas la puissance.

### 13.2 Choix : PostgreSQL managé, hébergé en France

Pas de base auto-hébergée sur un VPS. Un office de tourisme n'a personne pour appliquer les
correctifs de sécurité Postgres un dimanche soir.

| Option | Localisation | Ordre de grandeur/mois | Remarque |
|---|---|---|---|
| **Scaleway Managed PostgreSQL** | Paris | ~15-25 € | Recommandé : français, sauvegardes + PITR inclus, même fournisseur que le stockage objet |
| Clever Cloud | Paris | ~15-30 € | Français, très bon support, bonne option si le client connaît déjà |
| OVH Cloud Databases | Gravelines/Roubaix | ~20-30 € | Si le client est déjà chez OVH (comme Tignes) |
| Neon | Francfort (UE) | ~0-20 € | Excellent en DX, mais Allemagne — moins vendeur pour une collectivité |

*Montants indicatifs à revalider au moment du devis.* Recommandation : **Scaleway Paris**, un seul
fournisseur pour la base, le stockage et l'hébergement — une facture, un DPA, un interlocuteur.

### 13.3 Trois environnements, trois bases distinctes

| Env | Base | Données | Accès |
|---|---|---|---|
| Local | Postgres en Docker | jeu de test généré (seed) | développeur |
| Recette | instance managée dédiée | données **fictives** | client + dev |
| Production | instance managée dédiée | données réelles | application seule |

Règle absolue : **aucune donnée de production en recette.** Si un jeu réaliste est nécessaire
pour la recette, on part d'un dump anonymisé (noms, e-mails, téléphones remplacés).

```yaml
# docker-compose.yml — environnement local
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: larosiere
      POSTGRES_PASSWORD: dev_only
      POSTGRES_DB: larosiere_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
volumes: { pgdata: }
```

### 13.4 Migrations

Le schéma vit dans le repo, jamais dans une console d'administration.

```bash
# en développement — crée le fichier de migration
npx prisma migrate dev --name ajout_classement_logement

# en recette et en production — joué par la CI, jamais à la main
npx prisma migrate deploy
```

- Chaque migration est un fichier SQL versionné dans Git, relu en revue de code.
- `prisma db push` est **interdit** hors du poste de développement : il modifie le schéma sans
  laisser de trace et peut détruire des colonnes.
- Toute migration destructive (suppression de colonne, changement de type) se fait en deux temps :
  déploiement additif, migration des données, puis suppression au déploiement suivant.
- La CI joue les migrations sur une base jetable avant tout déploiement : si ça casse, ça casse en CI.

### 13.5 Sauvegardes

- **Quotidiennes automatiques**, rétention 30 jours (inclus chez Scaleway/Clever Cloud).
- **PITR** (restauration à un instant précis) sur 7 jours — couvre le « on a supprimé les mauvais dossiers ce matin ».
- **Un export hebdomadaire déposé sur un stockage séparé** (bucket différent, autres identifiants) :
  une sauvegarde chez le même fournisseur que la base ne protège pas d'une compromission du compte.
- **Test de restauration effectué en S8, puis une fois par an.** Une sauvegarde jamais restaurée
  n'est pas une sauvegarde. Ce test fait partie des livrables.

### 13.6 Sécurité de la base

- **Aucune exposition publique** : réseau privé entre l'application et la base, ou liste blanche d'IP.
- **TLS obligatoire** pour toute connexion (`sslmode=require`).
- **Utilisateur applicatif restreint** : `SELECT/INSERT/UPDATE/DELETE` sur le schéma applicatif,
  ni superutilisateur, ni droit de créer des bases. Un second utilisateur, séparé, pour les migrations.
- **Identifiants dans le gestionnaire de secrets** de l'hébergeur, jamais dans le repo.
  `.env` en `.gitignore`, `.env.example` sans valeur réelle.
- **Chiffrement au repos** activé (standard chez les fournisseurs managés).
- **Aucune donnée bancaire en base** : Stripe conserve les moyens de paiement, on ne stocke
  qu'un identifiant de transaction.
- Les mots de passe sont des empreintes Argon2id, jamais réversibles.

### 13.7 RGPD

- Hébergement **France**, sous-traitant unique, **DPA signé** avec le fournisseur.
- Durées de conservation à inscrire dans le registre de traitement :
  compte inactif purgé après 3 ans sans connexion (avec e-mail d'avertissement à 2 ans et 11 mois),
  documents conservés le temps de l'adhésion + durée légale.
- **Suppression de compte** en libre-service : les données personnelles sont effacées, mais
  `AuditLog` est **anonymisé** (`userId` mis à `NULL`, conservation de l'action et de la date)
  — on garde la traçabilité sans garder l'identité.
- **Export de ses données** en JSON, en libre-service depuis `/espace/profil`.

### 13.8 Reprise des données propriétaires existantes (S8)

Script d'import dédié, avec trois garde-fous :

1. **Mode simulation obligatoire d'abord** (`--dry-run`) : produit un rapport (lignes valides,
   doublons d'e-mail, champs manquants) que le client valide **avant** tout écrit en base.
2. **Idempotent** : relançable sans créer de doublons (clé sur l'e-mail normalisé).
3. **Aucun mot de passe importé.** Les comptes sont créés sans mot de passe, et chaque
   propriétaire reçoit un e-mail d'activation avec un lien à usage unique valable 30 jours.
   C'est aussi l'occasion de nettoyer la base : les e-mails morts ressortent tout de suite.

L'envoi des e-mails d'activation se fait **par lots** (200/jour) pour préserver la réputation
du domaine expéditeur et permettre à l'OT d'absorber les appels des propriétaires perdus.
