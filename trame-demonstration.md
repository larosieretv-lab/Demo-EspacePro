# Trame de démonstration — 10 minutes

À lire une fois avant la réunion, pas pendant. L'objectif n'est pas de montrer vingt écrans :
c'est d'en faire manipuler trois par votre interlocuteur. Ce qu'il aura fait de ses mains, il
s'en souviendra ; ce que vous aurez montré, non.

---

## Avant d'entrer dans la salle

- **Thème clair.** La maquette suit le réglage de votre système. En thème sombre, sur un
  vidéoprojecteur dans une salle éclairée, elle devient illisible. Réglez macOS ou Windows en
  clair, et vérifiez sur l'écran de la salle avant que les gens s'assoient.
- **Un seul onglet ouvert.** Fermez le reste. Une notification qui surgit pendant une
  démonstration de sécurité, c'est le seul détail dont ils se souviendront.
- **Zoom à 110 ou 125 %.** Le texte de la maquette est fin ; au fond d'une salle de réunion,
  personne ne lit du 15 pixels.
- **Ouvrez la page et laissez le bandeau de consentement s'afficher.** Ne le fermez pas
  d'avance : il fait partie de la démonstration.
- **Testez le réseau.** Un seul écran appelle l'extérieur, la vérification SIRET. S'il n'y a
  pas de wifi, ce écran bascule sur son jeu local — dites-le plutôt que de laisser croire à une
  panne.
- **Sachez le mot de passe :** `demonstration`. Vous en aurez besoin, et le chercher devant
  eux casse le rythme.

L'adresse : **https://larosieretv-lab.github.io/Demo-EspacePro/**

Prévoyez aussi l'écran **Recette** imprimé, un exemplaire par personne. La feuille de style
d'impression le sort proprement, sans bandeau ni boutons. C'est le document qu'ils garderont.

---

## La trame

| Temps | Écran | Ce que vous faites | Ce que vous dites |
|---|---|---|---|
| 0:00 | Accueil, bandeau de consentement affiché | Rien. Laissez-les le regarder | « Avant tout : rien n'est chargé tant que vous n'avez pas répondu. Refuser prend un clic, comme accepter — même bouton, même taille » |
| 0:30 | Cliquez **Tout refuser** | | « Voilà. Aucun traceur ne partira, et on ne vous reposera pas la question avant six mois » |
| 1:00 | Bandeau jaune → **Propriétaire** | Ouvrez le tableau de bord | « Voici ce que voit un propriétaire en arrivant. Ce que l'office attend de lui est en haut, classé, le bloquant d'abord » |
| 1:30 | **Mes logements** → ouvrez *Chalet Les Mélèzes* | Montrez la ligne d'assurance | « Son assurance expire le 30 septembre, avant l'ouverture. Personne n'a eu à y penser : le dossier l'a vu » |
| 2:30 | **Mes avantages** → *Éditer un billet* sur le Ruitor | Laissez le code QR s'afficher | « Cinq entrées gratuites au cinéma, cinq à la patinoire. Il édite son billet au moment d'y aller » |
| 3:30 | Bandeau jaune → **Agent de l'office** | Le code à six chiffres est demandé. Cliquez *Coller le code* | « Un agent de l'office ne passe jamais du mot de passe à ses dossiers : le second facteur est imposé, sans contournement » |
| 4:00 | **Contrôle à l'entrée** → *Coller un billet valable* → **Contrôler** | | « Entrée autorisée » |
| 4:30 | **Rejouer le même billet** → **Contrôler** | Laissez le refus s'afficher, en silence | « Déjà utilisé, à telle heure, par tel agent. Une capture d'écran transmise à un ami ne crée pas une entrée de plus : elle en retire une » |
| 5:30 | **À instruire** → *Refuser* sur une pièce | Choisissez un motif, validez | « L'agent ne peut pas refuser sans dire pourquoi » |
| 6:00 | Bandeau jaune → **Propriétaire** | Le motif est là, sous la pièce | « Le propriétaire lit le motif, pas seulement le verdict. Il remplace la pièce, et le dossier repart tout seul » |
| 7:00 | Bandeau jaune → **Traceurs** | Montrez le compteur | « Zéro requête vers un tiers. Ce compteur ne vous fait pas confiance : il lit ce que votre navigateur a réellement fait » |
| 8:00 | Bandeau jaune → **Recette** | Faites défiler | « Les treize constats relevés sur l'outil de Tignes deviennent vos critères de réception. Sept se vérifient tout de suite, six demandent le serveur — c'est écrit » |
| 9:00 | | Rendez la main | « Prenez la souris deux minutes » |

---

## Les trois moments où vous leur donnez la souris

Ce sont eux qui vendent. Le reste est du décor.

**1. Le verrouillage.** Écran de connexion, invitez-les à taper n'importe quel mot de passe,
plusieurs fois. Le compteur descend sous leurs yeux, puis le compte se verrouille avec un
décompte de quinze minutes. Dites : *« essayez de recharger la page »* — le verrou tient. Et
ajoutez : *« le bon mot de passe est refusé lui aussi, c'est le principe »*.

**2. Le nom de logement hostile.** Déclarez un logement devant eux et nommez-le
`<script>alert(1)</script>`. Rien ne se passe : le nom s'affiche en toutes lettres. C'est la
faille la plus courante du web, rendue visible en dix secondes. Si personne dans la salle ne
comprend le geste, ne le faites pas — ça tombe à plat devant un public non technique.

**3. Le billet rejoué.** Laissez-les scanner le billet une deuxième fois eux-mêmes. Le refus
est plus convaincant quand c'est leur main qui l'a déclenché.

---

## Les questions qui viendront

**« Combien ça coûte à faire tourner ? »**
30 à 70 € par mois, hébergement en France compris. Le détail est au §12 ter du plan.

**« Nos données partent où ? »**
En France, chez un hébergeur français, un seul sous-traitant, un contrat de sous-traitance
signé. C'est un argument que peu de prestataires peuvent tenir, et c'est le vôtre.

**« On peut avoir les socio-pros aussi ? »**
Oui, en phase 2, à chiffrer séparément. Ne l'ouvrez pas maintenant : c'est le périmètre qui
double en réunion, et le calendrier qui casse. Le plan le dit au §10, montrez-le et passez.

**« Et si vous n'êtes plus là dans deux ans ? »**
Le code est dans un dépôt qui leur appartient, les migrations sont versionnées, la
documentation d'exploitation fait partie des livrables. Un autre développeur reprend sans
archéologie. C'est la question qui les inquiète le plus et qu'ils poseront le moins
directement.

**« Combien d'entrées gratuites ? »**
Ce que vous voulez : c'est un réglage de saison, modifiable depuis leur back-office sans
déploiement. La maquette en montre cinq à titre d'exemple.

**« C'est quand qu'on peut l'avoir ? »**
Neuf semaines à partir de votre feu vert. Un accord avant fin août met la mise en ligne fin
octobre, six semaines avant l'ouverture de la saison. Chaque semaine d'hésitation décale
d'une semaine.

---

## Ce qu'il ne faut pas dire

- **Ne promettez pas que ça marche déjà.** C'est une maquette : aucun serveur, aucun e-mail
  n'est envoyé, rien n'est conservé. Le bandeau jaune le dit ; dites-le aussi, une fois, au
  début. Un client qui découvre seul que le mail n'arrive pas perd confiance dans tout le
  reste.
- **Ne dites pas que la sécurité est prouvée.** Sept constats sur treize sont vérifiables dans
  la maquette. Les six autres demandent un serveur et se vérifient en recette. C'est écrit sur
  l'écran Recette : appuyez-vous dessus, c'est plus crédible qu'une affirmation.
- **N'attaquez pas TignesPro.** L'analyse dit « défenses absentes », pas « failles
  exploitées », et aucun test intrusif n'a été mené. Si le sujet vient, tenez cette ligne :
  vous avez regardé de l'extérieur ce qui est visible de l'extérieur. Ils connaissent peut-être
  les gens de Tignes.
- **N'inventez pas de chiffre.** Si on vous demande le prix de la phase 2 ou le coût d'un
  module, dites que c'est à chiffrer et notez la question. Un chiffre lâché en réunion devient
  un engagement.

---

## Ce que vous demandez en repartant

Ne sortez pas sans avoir posé ces quatre demandes. Elles conditionnent le démarrage, et les
obtenir tôt vaut mieux que les réclamer en semaine 3.

1. **Une validation du périmètre** — propriétaires seuls, sans encaissement, avec le module
   des avantages.
2. **Les CGU, les mentions légales et la politique de confidentialité**, ou le nom de la
   personne qui les rédige.
3. **Le barème** : combien d'entrées gratuites par équipement et par saison.
4. **Un contact technique pour le domaine et les DNS** — c'est ce qui bloque les e-mails, donc
   tout le reste.

Et fixez la date du point suivant avant de quitter la salle. Un dossier qui repart sans
prochaine date ne repart pas.
