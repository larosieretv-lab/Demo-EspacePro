# Espace propriétaires — La Rosière

Proposition pour un espace propriétaires de l'office de tourisme de La Rosière : une
maquette manipulable, le plan de réalisation qui va avec, et l'analyse de l'outil
comparable de la station voisine qui a servi de point de départ.

**Ce dépôt n'est pas le produit.** C'est le dossier d'une proposition.

## Les quatre pièces

| Fichier | Ce que c'est |
|---|---|
| [`index.html`](index.html) | La maquette. Un fichier unique, à ouvrir dans un navigateur. |
| [`plan-implementation.md`](plan-implementation.md) | Le plan de réalisation : périmètre, modèle de données, planning en 8 semaines, sécurité, exploitation de la base. |
| [`analyse-tignespro.md`](analyse-tignespro.md) | L'analyse de TignesPro, relevée de l'extérieur le 4 août 2026. Elle fournit les treize constats qui deviennent des critères de recette au §8 du plan. |
| [`trame-demonstration.md`](trame-demonstration.md) | Le déroulé de la démonstration en dix minutes : quoi cliquer, quoi dire, quoi faire faire, et quoi ne pas promettre. |

## La maquette

Elle sert à discuter d'écrans réels plutôt que de captures, et à rendre vérifiables les
promesses de sécurité au lieu de les affirmer. Douze parcours s'y manipulent — création de
compte, vérification SIRET contre l'annuaire des entreprises de l'État, déclaration d'un
logement, invitation d'un copropriétaire, instruction et refus motivé côté office, échéances
d'assurance, annuaire avec exports, verrouillage après cinq échecs de connexion, réponse
muette du mot de passe oublié, second facteur imposé aux agents, compteur de requêtes
tierces, écran de recette.

Le §12 bis du plan les détaille, ainsi que la répartition des treize constats entre ceux
qu'on peut vérifier dans la maquette et ceux qui attendent un serveur.

### Ce qu'elle ne fait pas

Aucun serveur, aucune base de données, aucun service d'envoi. Donc :

- **aucun e-mail ne part**, quel que soit le formulaire rempli ;
- aucun compte n'est créé, aucune donnée n'est transmise ni conservée ;
- tout l'état vit dans l'onglet et disparaît à sa fermeture.

Publiée sur GitHub Pages — de l'hébergement de fichiers statiques — elle n'exécute par
construction aucun code côté serveur. Les écrans concernés le disent à l'endroit où la
question se pose, plutôt que de laisser attendre un message qui n'arrivera pas.

Les données sont fictives, y compris les huit propriétaires de l'annuaire de l'office, qui
n'existent que pour donner de la matière à la recherche et à l'export.

### La parcourir

Ouvrir `index.html` suffit. Le bandeau jaune en haut permet d'ouvrir la session d'un
propriétaire ou d'un agent de l'office sans passer par un mot de passe, et donne accès aux
quatre écrans transversaux : vérification SIRET, traceurs, recette, comparaison.

Chaque écran porte son adresse (`#recette`, `#logement/LGT-002`), ce qui permet d'en
envoyer un directement. Un lien vers un espace privé passe par la connexion, puis y ramène.

Le mot de passe de démonstration est `demonstration` ; tout autre mot de passe échoue pour
de bon, et cinq échecs verrouillent le compte un quart d'heure.

## Note sur la charte

Couleurs et proportions relevées sur larosiere.net. La police est Lexend, sous licence
libre ; la charte réelle utilise TT Chocolate pour les titres. Rien n'est validé par
l'office de tourisme à ce stade.
