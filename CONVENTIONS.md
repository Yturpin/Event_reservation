# Conventions du projet

Ce fichier fixe le format commun de nos fichiers JSON, pour que les cinq services se ressemblent et s'emboîtent.

## Les services

| Service | Identifiant | Agrégat principal | Préfixe des identifiants |
|---|---|---|---|
| Utilisateurs | `utilisateurs` | `Utilisateur` | `USR-` |
| Cas | `cas` | `Cas` | `CAS-` |
| Réservations | `reservations` | `Reservation` | `RES-` |
| Paiements | `paiements` | `Paiement` | `PAY-` |
| Avis | `avis` | `Avis` | `AVI-` |

## Vocabulaire : « cas » et « événement »

- Un **cas** est ce que l'on réserve : un concert, un match, un spectacle… Son identifiant s'appelle **`casId`**.
- Un **événement** est un message qu'un service envoie aux autres pour annoncer un fait (`PlacesReservees`, `PrixModifie`…), au sens du cours. Son identifiant s'appelle **`eventId`**, comme dans le support du prof.

Le mot « événement » ne désigne **jamais** un concert ou un match.

## Règles de nommage

| Élément | Règle | Exemples |
|---|---|---|
| Langue | Français, **sans accents** (le code viendra ensuite) | `prixPlace`, `casId` |
| Champs JSON | camelCase | `placesRestantes`, `dateReservation` |
| Commandes | Verbe à l'**infinitif**, PascalCase | `ReserverPlaces`, `AnnulerReservation` |
| Événements | Fait au **passé**, PascalCase | `PlacesReservees`, `CasAnnule` |
| Agrégats | Nom au singulier, PascalCase | `Reservation`, `Paiement` |
| Copies (réplicas) | Nom de l'agrégat d'origine + `Replica` | `CasReplica` |
| Statuts | MAJUSCULES, mots séparés par `_` | `EN_ATTENTE_PAIEMENT`, `CONFIRMEE` |
| Fichiers | Nom de l'élément + `.json` | `ReserverPlaces.json` |

## Valeurs

| Type de valeur | Format | Exemple |
|---|---|---|
| Identifiants métier | Préfixe du service + numéro | `USR-12`, `CAS-7`, `RES-301` |
| Identifiant d'un événement (`eventId`) | UUID | `"b820a46d-1e3f-48a0-9459-f723c80e1260"` |
| Dates et heures | ISO 8601 | `"2026-12-12T20:00:00"` |
| Montants | Nombre en euros | `45.0` |

**Pas de commentaires dans les fichiers JSON** : le format ne les autorise pas. Les explications vont dans les champs `description`.

## Organisation des dossiers

```text
services/       # étape 2 : un fichier par service
aggregates/     # étapes 3 et 4 : un dossier par service, avec son agrégat et ses copies
events/         # étape 4 : un fichier par événement
projections/    # étape 6 : la projection et sa requête
```

Exemple pour Réservations :

```text
services/reservations.json
aggregates/reservations/Reservation.json
aggregates/reservations/CasReplica.json
events/PlacesReservees.json
```

## Modèles

### Étape 2 : un service et ses commandes

Fichier `services/<identifiant>.json`.

```json
{
  "service": "reservations",
  "description": "Gère la réservation et l'annulation de places, et tient le compte des places restantes.",
  "commandes": [
    {
      "nom": "ReserverPlaces",
      "description": "Un participant réserve des places pour un cas.",
      "donnees": {
        "utilisateurId": "USR-12",
        "casId": "CAS-7",
        "nbPlaces": 2
      },
      "refuseeSi": [
        "il ne reste pas assez de places",
        "le cas est annulé ou déjà passé"
      ],
      "evenementsEmis": ["PlacesReservees"]
    }
  ],
  "reactions": [
    {
      "evenement": "PaiementAccepte",
      "venantDe": "paiements",
      "action": "Passer la réservation en CONFIRMEE.",
      "evenementsEmis": ["ReservationConfirmee"]
    }
  ]
}
```

- `donnees` : ce que l'appelant doit fournir, avec des valeurs d'exemple.
- `refuseeSi` : les cas où le service refuse la commande. C'est ce qui distingue une commande d'un événement.
- `evenementsEmis` : les événements publiés si la commande réussit. On peut laisser `[]` à l'étape 2 et le compléter à l'étape 4.

**Commande ou réaction ?**

- Une **commande** vient de l'extérieur, en général d'un utilisateur, et peut être refusée.
- Une **réaction** est déclenchée par un événement publié par un autre service. Le service qui réagit ne modifie que ses propres données. Celui qui a publié l'événement ne sait pas qui l'écoute.

Dans `reactions`, `venantDe` indique le service qui publie l'événement, et `action` ce que le service fait de lui-même en le recevant.

### Étape 3 : l'agrégat d'un service

Fichier `aggregates/<identifiant>/<Agregat>.json`.

```json
{
  "agregat": "Reservation",
  "service": "reservations",
  "description": "Une réservation de places par un participant pour un cas.",
  "identifiant": "reservationId",
  "champs": {
    "reservationId": "texte",
    "utilisateurId": "texte",
    "casId": "texte",
    "nbPlaces": "entier",
    "prixTotal": "nombre",
    "statut": "EN_ATTENTE_PAIEMENT | CONFIRMEE | ANNULEE",
    "dateReservation": "date"
  },
  "exemple": {
    "reservationId": "RES-301",
    "utilisateurId": "USR-12",
    "casId": "CAS-7",
    "nbPlaces": 2,
    "prixTotal": 90.0,
    "statut": "EN_ATTENTE_PAIEMENT",
    "dateReservation": "2026-10-26T14:00:00"
  }
}
```

- `champs` : le nom et le type de chaque champ. Pour un statut, on liste ses valeurs possibles, séparées par `|`.
- `exemple` : un document réaliste, comme il serait stocké en base.
- L'agrégat ne contient **que** les données dont le service est propriétaire.

### Étape 4 : une copie (réplica)

Fichier `aggregates/<service qui garde la copie>/<Agregat>Replica.json`.

```json
{
  "agregat": "CasReplica",
  "service": "reservations",
  "source": { "service": "cas", "agregat": "Cas" },
  "pourquoi": "Accepter ou refuser une réservation sans appeler le service Cas, même s'il est indisponible.",
  "alimentePar": ["CasCree", "PrixModifie", "CasAnnule"],
  "identifiant": "casId",
  "champs": {
    "casId": "texte",
    "titre": "texte",
    "date": "date",
    "prixPlace": "nombre",
    "capacite": "entier",
    "statut": "PREVU | ANNULE"
  },
  "exemple": {
    "casId": "CAS-7",
    "titre": "Concert de rap",
    "date": "2026-12-12T20:00:00",
    "prixPlace": 45.0,
    "capacite": 500,
    "statut": "PREVU"
  }
}
```

- `source` : le service propriétaire et l'agrégat d'origine. La copie n'est **jamais** la source officielle.
- `pourquoi` : la raison de la copie. C'est ce que le prof veut voir.
- `alimentePar` : les événements qui créent puis mettent à jour la copie. Il en faut **au moins un de création et un de mise à jour**.
- On ne copie **que les champs utiles** au service qui garde la copie.

### Étape 4 : un événement entre services

Fichier `events/<NomEvenement>.json`. Il reprend les quatre éléments demandés par le sujet : émetteur, récepteur, donnée modifiée et nouvelle valeur.

```json
{
  "eventType": "PrixModifie",
  "eventVersion": 1,
  "description": "L'organisateur a changé le prix d'une place.",
  "emetteur": "cas",
  "recepteurs": [
    { "service": "reservations", "effet": "METTRE_A_JOUR", "copie": "CasReplica" }
  ],
  "declenchePar": "ModifierPrix",
  "donneesModifiees": ["prixPlace"],
  "exemple": {
    "eventType": "PrixModifie",
    "eventVersion": 1,
    "eventId": "d4e5f6a7-1b2c-4d3e-9f80-123456789abc",
    "dateEmission": "2026-10-27T10:15:00",
    "casId": "CAS-7",
    "prixPlace": 50.0
  }
}
```

- `emetteur` : le service qui publie. Ce doit être le **propriétaire** des données modifiées.
- `recepteurs` : chaque service qui reçoit l'événement. `effet` vaut `CREER` (création de la copie), `METTRE_A_JOUR` ou `SUPPRIMER`.
- `declenchePar` : la commande qui provoque l'événement.
- `donneesModifiees` : les champs de l'agrégat qui changent.
- `exemple` : le message tel qu'il circule. Il contient toujours `eventType`, `eventVersion`, `eventId` et `dateEmission`, puis l'identifiant concerné et **les nouvelles valeurs**.

### Étape 6 : une projection

Format à fixer au moment de l'étape 6.

## Vérifications avant de rendre l'étape 4

- [ ] Chaque événement a un émetteur qui est bien le propriétaire des données.
- [ ] Chaque événement a au moins un récepteur, et chaque récepteur a la copie correspondante.
- [ ] Chaque copie a au moins un événement de création et un de mise à jour.
- [ ] Aucun service ne modifie les données d'un autre.
- [ ] Les noms suivent les règles ci-dessus.
- [ ] Chaque fichier JSON est valide.
