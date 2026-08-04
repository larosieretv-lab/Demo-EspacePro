# Analyse TignesPro.net — base pour l'Espace Propriétaires La Rosière

Analyse **passive** du 4 août 2026 : uniquement des pages publiques (`/inscription`,
`/proprio_inscription`, `/sociopro_inscription`, `/salarie_inscription`, `/connexion/2`,
`/proprio_mdp_oublie.php`, `/dashboard-proprio`), en-têtes HTTP et code source livré au
navigateur. Aucun test intrusif, aucune tentative de connexion : le site ne nous appartient pas.
Tout ce qui est marqué *(inféré)* est une déduction, pas une observation directe.

---

## 1. Comment c'est codé

### Serveur
| Élément | Observation |
|---|---|
| Serveur web | `Server: IIS` (Windows Server) |
| Langage | PHP (session `PHPSESSID` renommée en `PXPVB`, endpoints `.php` partout) |
| Hébergement | IP `178.33.93.119` → OVH (France) |
| Base de données | non visible *(inféré : MySQL/MariaDB ou SQL Server)* |
| Framework | **aucun** — PHP « à la main », un fichier par page, pas de MVC apparent |
| Protocole | HTTP/1.1 uniquement (pas de HTTP/2) |
| Auteur | Vincent Brigand (`meta name="Author"`) + agence Azimuts (Valérie Seban) |

### Architecture applicative
- **Routing** : URLs propres (`/inscription`, `/connexion/2`, `/dashboard-partenariat`)
  réécrites par IIS vers des `.php`, mais l'ancien monde affleure encore
  (`/index.php`, `/switchlang.php`, `/proprio_mdp_oublie.php`).
- **Rendu** : 100 % côté serveur, HTML assemblé par includes (header / section / footer).
- **Interactions** : jQuery `$.load()` — le serveur renvoie des **fragments HTML** injectés
  dans le DOM. Pas d'API JSON, pas de séparation front/back.
  - inscription proprio → `POST proprio_ajax/proprio_ajax_inscription.php`
  - connexion → `POST /connect.php`
  - mot de passe oublié → `POST proprio_email_pwd.php`
  - inscription sociopro → `POST inscription_verif_saisie1.php`
- **Multilingue** : FR/EN via `switchlang.php` + cookie `OTTIGNESLangue` (1 an).

### Front-end
Thème HTML acheté (type *Unify/Pixup*), non compilé, dépendances datées :

| Lib | Version | Statut 2026 |
|---|---|---|
| jQuery | 2.2.3 (2016) | obsolète |
| Bootstrap | 3.x | fin de support depuis 2019 |
| jQuery Validate + form.validate | ancien | ok mais client-side only |
| DataTables 1.10.16, Select2 4.0.6-rc, Leaflet 1.8, Dropzone, bootstrap-slider | 2017-2018 | obsolète |
| tarteaucitron.js | — | gestion consentement CNIL |
| Google Analytics GA4 | `G-8HV6D7CKRE` | chargé **avant** consentement |

Trois libs sont chargées depuis des CDN externes (`cdnjs.cloudflare.com`,
`cdn.datatables.net`) **sans attribut `integrity` (SRI)** → dépendance de chaîne
d'approvisionnement non vérifiée.

### Défauts de fabrication visibles
- Du HTML (`<div class="rotate-mask">`, bandeau cookies) est **imprimé avant le `<!DOCTYPE html>`** :
  le document est structurellement invalide, et cela indique un `echo` avant l'en-tête → tout
  `header()` PHP ultérieur échouerait.
- Les blocs d'alerte « CGU en attente », « facture à payer » sont **dans le HTML de tous les
  visiteurs**, y compris déconnectés, et masqués par CSS/JS. Fuite d'information mineure,
  mais surtout : la logique métier est côté affichage.
- La page d'accueil `/` est un `<SCRIPT>document.location.href='/inscription'</SCRIPT>` de 59 octets
  au lieu d'une redirection HTTP 302.
- `/admin/conf/timthumb.php` est utilisé pour redimensionner les images : composant abandonné
  depuis ~2014, historiquement à l'origine de RCE massives. Sa seule présence est un signal.
- Pas de `robots.txt` (404).
- Le code contient des restes visibles (`class="celarfix"`, `<strong>` non ouvert, styles inline).

---

## 2. Gestion des comptes

### Trois profils, trois silos
`/inscription` est un **choix de profil**, pas un formulaire :

| Profil | Inscription | Connexion | Cookie d'identité | Mot de passe oublié |
|---|---|---|---|---|
| Socioprofessionnel | `/sociopro_inscription` | `/connexion/1` | `OTTIGNESIDClient` | `/mdp_oublie` |
| **Propriétaire** | `/proprio_inscription` | `/connexion/2` | `PROPRIOIDClient` | `/proprio_mdp_oublie.php` |
| Salarié Tignes Dév. | `/salarie_inscription` | `/connexion/3` | `OTTIGNESIDSalarie` | `/salarie_mdp_oublie.php` |

C'est **trois applications juxtaposées** partageant un gabarit : trois tables, trois tunnels,
trois fichiers de reset. Le même e-mail peut vraisemblablement exister dans les trois *(inféré)*.
Sur `/connexion`, l'utilisateur doit **choisir lui-même son programme** dans un `<select>` —
le serveur ne déduit pas le profil depuis l'identifiant.

### Parcours propriétaire (le nôtre)
Formulaire unique, compte créé immédiatement :
`civilité` (dont « Société » → révèle `raison_sociale` + `contact_societe`), `nom`, `prénom`,
`email`, `téléphone`, `code postal`, `ville`, `mot de passe` + confirmation.
→ `POST proprio_ajax/proprio_ajax_inscription.php` (réponse HTML injectée).
*(Inféré : e-mail de confirmation puis validation manuelle par l'office de tourisme, cohérent
avec les statuts de dossier visibles dans les modales.)*

### Parcours socioprofessionnel (plus intéressant, workflow B2B)
Étape 1 sans mot de passe : `nom fiscal`, `nom/prénom du responsable`, **SIRET**, `téléphone`,
`email`, `acceptation CGA` → `inscription_verif_saisie1.php`.
Le compte n'existe qu'après vérification. Le reste du système montre un vrai **workflow d'adhésion** :
saisie d'« entités » (établissements), envoi du dossier, contrôle de complétude, validation des CGU,
puis **facturation de l'adhésion** (`/dashboard-partenariat`).

### Session & authentification
```
PXPVB=…; Max-Age=10800; path=/; secure; HttpOnly      ← session PHP, 3 h
PROPRIOIDClient / OTTIGNESIDClient / OTTIGNESIDSalarie ← identité persistante
OTTIGNESLangue=fr; Max-Age=31536000                    ← langue
```
- Session : 3 h, `Secure` + `HttpOnly` ✅, **pas de `SameSite`** ❌.
- Les cookies d'identité servent au « maintien de votre authentification d'une visite à l'autre »
  (dixit le bandeau cookies) : c'est un **remember-me maison**. Impossible de vérifier leur
  contenu sans compte — s'il s'agit d'un ID client en clair ou faiblement signé, c'est une
  usurpation triviale *(à considérer comme le risque n°1 tant qu'il n'est pas démenti)*.
- `/dashboard-proprio` non authentifié renvoie **200 avec une page vide** au lieu d'un 302 vers
  la connexion : le garde-fou est un `exit` tardif, pas un contrôle d'accès en amont.

---

## 3. Sécurité — bilan

### Ce qui est en place ✅
- HTTPS avec HSTS (`max-age=31536000`).
- Session `Secure` + `HttpOnly`, durée courte (3 h).
- `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`.
- `Cache-Control: no-store, no-cache` sur les pages authentifiées.
- Indicateur de robustesse de mot de passe (`jquery.validate.password.js`).
- Bandeau de consentement + tarteaucitron.

### Ce qui manque ❌
| # | Constat | Gravité | Impact |
|---|---|---|---|
| 1 | **Aucun jeton CSRF** dans aucun formulaire (le `serializeArray()` n'envoie que les champs métier) | Élevée | Un site tiers peut faire agir un propriétaire connecté à son insu |
| 2 | Cookies d'identité persistants au contenu inconnu, sans `SameSite` | Élevée *(inféré)* | Usurpation de compte / CSRF facilité |
| 3 | **Aucun CAPTCHA ni rate-limiting visible** sur connexion, inscription et mot de passe oublié | Élevée | Bourrage d'identifiants, création de faux comptes, spam d'e-mails |
| 4 | **Pas de CSP**, pas de `Referrer-Policy`, pas de `Permissions-Policy` | Moyenne | Aucune barrière en cas de XSS |
| 5 | Réponses AJAX = **fragments HTML injectés** via `.load()` | Moyenne | Toute donnée mal échappée devient un XSS stocké |
| 6 | HTTP simple répond **200 sans redirection** vers HTTPS | Moyenne | Première visite non protégée (HSTS n'agit qu'après) |
| 7 | CDN externes **sans SRI** | Moyenne | Compromission d'un CDN = exécution de code chez tous les utilisateurs |
| 8 | Pas de **2FA**, pas de politique d'expiration, pas de journal de connexions | Moyenne | Aucune défense en profondeur sur des comptes qui portent des données bancaires/locatives |
| 9 | Dépendances de 2016-2018 (jQuery 2.2.3, Bootstrap 3) | Moyenne | CVE connues, plus aucun correctif |
| 10 | `timthumb.php` présent | À vérifier | Composant abandonné, historiquement critique |
| 11 | `maxlength=50` sur le mot de passe + **copier/coller bloqué** (`onpaste="return false"`) | Faible | Hostile aux gestionnaires de mots de passe → mots de passe plus faibles |
| 12 | GA4 chargé **avant** le consentement | Faible (juridique) | Non-conformité RGPD/CNIL |
| 13 | Validation uniquement côté client visible | À vérifier | Le serveur doit revalider — non vérifiable de l'extérieur |
| 14 | Pas de `Content-Security-Policy` ni de politique de mots de passe côté serveur documentée | Moyenne | — |

Rien de ce qui précède ne prouve une faille exploitable : ce sont des **absences de défenses**
constatées depuis l'extérieur. Un vrai audit demanderait un compte de test et l'accord écrit
de Tignes Développement.

---

## 4. Ce qu'on garde, ce qu'on jette pour La Rosière

### À reprendre (le métier est bon)
- La **segmentation par profil** dès l'entrée (propriétaire / socio-pro / salarié) : c'est clair
  pour l'utilisateur et ça correspond à l'organisation réelle d'un office de tourisme.
- Le **workflow d'adhésion** : dossier → complétude → validation par l'OT → CGU → facturation.
  C'est là qu'est la valeur, et c'est ce que le propriétaire de La Rosière voudra.
- La **vérification SIRET** en amont pour les pros.
- L'**espace propriétaire** orienté « faire occuper son logement » : classement meublé,
  déclaration en mairie, taxe de séjour, mise en relation.
- Le multilingue FR/EN dès le départ.

### À ne pas reproduire
- PHP sans framework, un fichier par page.
- `.load()` de fragments HTML au lieu d'une API.
- Trois systèmes de comptes parallèles → **un seul compte, plusieurs rôles**.
- Le remember-me maison → sessions gérées par une lib éprouvée.
- Blocage du coller sur les mots de passe.
- Dépendances CDN sans SRI.

---

## 5. Architecture cible proposée

**Stack** : Next.js 15 (App Router, TypeScript) + PostgreSQL + Prisma, hébergé en France
(Scaleway/OVH ou Vercel + Neon EU). Rendu serveur pour le SEO des pages publiques, API typée
pour l'espace privé.

**Authentification** : Auth.js/NextAuth ou Better Auth — sessions signées, rotation,
`SameSite=Lax`, `Secure`, `HttpOnly`. Argon2id pour les mots de passe. Magic link + mot de passe.
**2FA (TOTP) obligatoire pour les comptes OT/admin.**

**Modèle de comptes** — un utilisateur, des rôles :
```
User (id, email unique, passwordHash, emailVerified, mfaSecret, lastLoginAt)
 └─ Membership (userId, organizationId, role: OWNER | PRO | STAFF | OT_ADMIN)
      └─ Organization (raison sociale, SIRET, statut dossier)
           └─ Property / Entity (logements ou établissements)
                └─ Document (classement, assurance, photos)
Adhesion (organizationId, saison, statut, cguAcceptedAt, invoiceId)
AuditLog (userId, action, ip, userAgent, createdAt)
```

**Sécurité intégrée dès le départ**
- CSRF natif (Server Actions Next.js) + `SameSite=Lax`.
- Rate-limiting sur login / inscription / reset (Upstash ou middleware) + Turnstile Cloudflare.
- CSP stricte, `Referrer-Policy`, `Permissions-Policy`, HSTS avec `includeSubDomains; preload`,
  redirection 301 HTTP→HTTPS.
- Validation Zod partagée client **et** serveur.
- Autorisation centralisée (chaque requête filtrée par `organizationId`), jamais un `exit` tardif.
- Journal d'audit + notification e-mail à chaque nouvelle connexion.
- Uploads : antivirus, types autorisés, stockage S3 privé avec URLs signées.
- RGPD : consentement avant tout tracker, export et suppression de compte en libre-service,
  registre de traitement, DPA avec les sous-traitants.
- Dépendances : lockfile, Dependabot, aucun script CDN (tout bundlé).

**Fonctionnel Espace Propriétaire La Rosière**
1. Inscription + vérification e-mail, choix particulier/société.
2. Déclaration des logements (adresse, capacité, classement, n° d'enregistrement mairie).
3. Dépôt de documents (diagnostic, assurance, classement) avec statut de validation.
4. Tableau de bord : occupation, taxe de séjour, échéances.
5. Adhésion saisonnière : CGU → validation OT → facture → paiement en ligne (Stripe).
6. Espace OT (back-office) : validation des dossiers, relances, exports, statistiques.
7. Notifications e-mail transactionnelles (Brevo/Resend).

---

## 6. Prochaines étapes suggérées

1. Valider avec le client le **périmètre v1** (propriétaires seuls ? ou les 3 profils ?).
2. Obtenir les documents métier de La Rosière : CGU, barème d'adhésion, taxe de séjour.
3. Maquettes des 6 écrans clés avant toute ligne de code.
4. Chiffrage : v1 propriétaires ≈ 6-8 semaines, les 3 profils + facturation ≈ 3-4 mois *(à affiner)*.
