# Le YAML de validation — référence exhaustive

> **Pour créer ou refaire un formulaire, commence par `customform-builder.md`.** Il décrit
> le modèle `builder` — la voie qui laisse le formulaire modifiable à la souris par le
> client — et donne des exemples complets. Ce fichier-ci est la référence des règles
> elles-mêmes, que `builder` produit.

Le champ `formvalidate` contient du YAML, validé à l'enregistrement. Un document invalide
est **refusé en 422** avec la liste complète des problèmes — pas seulement le premier,
pour que tu corriges tout en une passe.

Un document déjà stocké mais devenu invalide (import, règle retirée du moteur) ne fait pas
tomber la page : la validation du bloc est neutralisée et un avertissement est journalisé.
Autrement dit **le formulaire ne valide plus rien** — d'où l'importance de lire les erreurs
plutôt que de les contourner.

## Squelette

```yaml
rules:            # obligatoire
  email:
    - validate: required
      message: "L'adresse email est obligatoire."
    - validate: email
      message: "Cette adresse email n'est pas valide."
  message:
    - validate: required
      message: "Merci d'écrire un message."
  _hp:
    - honeypot                    # forme courte : réservée aux validateurs sans message

labels:           # facultatif — libellés lisibles, utilisés par {{_form.fields}} et {{_field.<champ>.label}}
  email: "Adresse email"
  message: "Message"

meta:             # facultatif — désigne les champs sémantiques
  emailField: email
  messageField: message

success: "Merci, votre message a bien été envoyé."
error: "Merci de corriger les champs signalés."

redirect: <uuid-de-la-page-de-remerciement>    # facultatif

cleantalk:        # facultatif — uniquement `silent`
  silent: false
```

Huit clés racine, et **rien d'autre** : `rules`, `labels`, `meta`, `success`, `error`,
`redirect`, `cleantalk`, `builder`. Toute autre clé est refusée.

`builder` est le modèle du constructeur visuel — voir `customform-builder.md`. Quand il
est présent avec `mode: ui`, le CMS vérifie en plus qu'un champ `required: true` a bien une
règle `required`, `required_if` ou `required_with` dans `rules`. Sinon le formulaire
afficherait une contrainte que le serveur ne vérifie pas.

Avec `builder`, le CMS exige aussi la cohérence avec `rules` : un champ `required: true`
sans règle qui l'impose est refusé, et un `builder` sans `recaptcha: false` exige une
règle `recaptcha` (sinon, en production, aucun email ne part — en silence).

## `rules`

Un objet `nom-du-champ: [liste de règles]`. La clé doit correspondre **exactement** à
l'attribut `name=` de l'input dans le HTML — c'est l'erreur la plus fréquente et la plus
coûteuse, parce qu'une règle orpheline échoue en silence sur tous les envois.

### Un message est obligatoire

Chaque règle doit porter son `message` — **sauf** `honeypot`, `empty` et `csrf`. Sans lui,
le champ passe en erreur avec sa bordure rouge et **rien pour l'expliquer** : le visiteur
voit un formulaire qui refuse sans dire pourquoi. C'est refusé au save, pour que le
problème apparaisse à l'écriture plutôt qu'à la première soumission.

Les trois exceptions se justifient : un piège doit rester invisible, et un échec csrf
n'est pas une faute du visiteur — il n'y a rien d'utile à lui dire.

```yaml
rules:
  email:
    - validate: required
      message: "L'adresse email est obligatoire."
    - validate: email
      message: "Cette adresse email n'est pas valide."
  _hp:
    - honeypot                              # forme courte : légitime ici
```

La forme courte (`- required`) reste donc réservée aux trois validateurs sans message.

Contraintes : 100 champs maximum, 10 règles par champ, messages de 500 caractères,
document de 64 Ko. Un nom de champ dans `rules` suit `^[A-Za-z_][A-Za-z0-9_\-\.\[\]]{0,63}$`
— ce qui autorise `g-recaptcha-response` et `address.city`.

Attention : **`builder` est plus strict** — ni point, ni crochet, ni espace. Un nom généré
voyage dans les marqueurs du HTML, où un point peut atteindre un autre champ. Si tu passes
par le modèle, `address.city` est refusé ; écris `address_city`.

## Les 11 validateurs

| Validateur | `data` | Effet |
|---|---|---|
| `required` | — | valeur non vide (les espaces seuls ne comptent pas) |
| `required_if` | `field`, `value` | requis seulement si un autre champ vaut cette valeur |
| `required_with` | `field` | requis dès qu'un autre champ est rempli |
| `email` | — | format d'adresse email |
| `equals` | `value` | égalité stricte avec une valeur |
| `empty` | — | doit être vide (piège anti-spam) |
| `honeypot` | — | idem, nom explicite |
| `csrf` | — | jeton anti-CSRF, valable 10 minutes |
| `recaptcha` | — | vérifie `g-recaptcha-response` auprès de Google |
| `file-extensions` | `allowed` | liste d'extensions séparées par des virgules |
| `file-size` | `max` | taille maximale en octets (80 Mo plafond) |

Un validateur absent de cette liste est refusé au save. C'est délibéré : le moteur
ignorerait silencieusement une règle qu'il ne connaît pas, et le formulaire semblerait
validé alors qu'il ne l'est pas.

### Les validateurs conditionnels

Ce sont eux qui rendent possibles les formulaires à branches. Un champ masqué côté client
n'est pas soumis ; un `required` simple dessus ferait donc échouer **toutes** les autres
branches, sur un champ que le visiteur n'a jamais vu.

```yaml
rules:
  # Uniquement quand sujet = recrutement
  cv:
    - validate: required_if
      message: "Le CV est obligatoire pour une candidature."
      data:
        field: sujet
        value: recrutement
    - validate: file-extensions
      message: "Formats acceptés : PDF, DOC, DOCX."
      data: {allowed: 'pdf,doc,docx'}
    - validate: file-size
      message: "5 Mo maximum."
      data: {max: 5242880}

  # Dès que la société est renseignée
  tva:
    - validate: required_with
      message: "Merci d'indiquer le n° TVA de la société."
      data: {field: societe}
```

La comparaison se fait sur les chaînes, donc `value: 2` correspond bien à un `"2"` posté.
Un champ observé multi-valeurs correspond si l'une de ses valeurs est la bonne.

Une règle conditionnelle sans `field` est refusée au save : elle ne se déclencherait
jamais, et retirerait donc l'obligation en silence.

### Les pièges anti-spam

`honeypot` et `empty` sont équivalents : le champ doit arriver **vide**. Le champ
correspondant doit exister dans le HTML et être soumis (masqué en CSS, pas `disabled`),
sinon la règle échoue à chaque fois.

```html
<div aria-hidden="true" class="absolute -left-[9999px]">
  <input type="text" name="_hp" tabindex="-1" autocomplete="off" value="{{_field._hp.value}}">
</div>
```

Laisse leur `message` vide — c'est d'ailleurs le seul cas où le save l'accepte.
`{{_form.errors}}` masque les champs pièges en se fondant sur **le validateur**, pas sur le
message, donc le piège n'apparaît jamais au visiteur.

### reCAPTCHA

```yaml
  g-recaptcha-response:
    - validate: recaptcha
      message: "Merci de confirmer que vous n'êtes pas un robot."
```

À prévoir systématiquement dès que le formulaire envoie un email : **sans reCAPTCHA validé
et sans SMTP personnalisé, aucun mail ne part**, en silence. Ajoute aussi
`{{_form.recaptcha}}` dans le HTML, sinon le visiteur n'a rien à cocher et ne peut jamais
soumettre.

## `labels`

```yaml
labels:
  nom: "Nom"
  email: "Adresse email"
```

Sert à `{{_form.fields}}` (le tableau récapitulatif des emails) et à `{{_field.champ.label}}`. Sans
libellé, la clé brute s'affiche — un email listant `prenom` plutôt que « Prénom » fait
négligé. L'ordre des clés **impose aussi l'ordre des lignes** du tableau.

Dans le formulaire lui-même, écris les libellés directement dans les `<label>` : `labels`
ne sert pas à ça.

## `meta`

```yaml
meta:
  emailField: email
  messageField: message
```

Désigne quel champ porte l'adresse et lequel porte le corps du message. Cleantalk s'en sert
pour savoir quoi analyser : sans `meta`, ses placeholders resteraient figés sur des noms de
champs codés en dur côté configuration, et renommer un input casserait l'anti-spam sans
aucun signal.

À renseigner dès que Cleantalk est configuré, donc dès qu'il y a un accusé de réception au
visiteur.

## `success`, `error`

`success` s'affiche via `{{_form.messages}}` **après la redirection** qui suit une soumission
valide. `error` s'affiche immédiatement sur la page re-rendue en cas d'échec.

Laisser `success` vide n'est pas neutre : le visiteur voit alors son formulaire se vider
sans la moindre confirmation, ce qui donne l'impression que rien ne s'est passé.

## `redirect`

```yaml
redirect: 6f2c1e88-4a2b-4f19-9c31-0d2e4b7a9f10
```

L'identifiant de la page de remerciement. Sans lui, le visiteur revient sur la page du
formulaire avec le message de succès — ce qui convient dans la plupart des cas.

Accepte aussi la forme liste (`redirect: ['<uuid>']`). Récupère l'identifiant avec
`get-navigations-with-pages`, ne l'invente pas.

## `cleantalk`

```yaml
cleantalk:
  silent: false
```

**`silent` est la seule clé acceptée ici.** `key`, `email` et `message` sont refusés avec
un message qui rappelle où chacun se configure : la clé d'API vit dans le YAML du type de
contenu — c'est un secret, et c'est un identifiant par site, pas par formulaire — et les
champs analysés se déduisent de `meta`.

`silent: true` (le défaut) laisse passer les envois détectés comme spam en apparence, mais
les actions suivantes sont neutralisées : le visiteur voit un succès, l'email ne part pas.
`silent: false` affiche une erreur.

## Exemple complet — formulaire à trois branches

```yaml
rules:
  sujet:
    - validate: required
      message: "Merci de préciser votre demande."

  societe:
    - validate: required_if
      message: "Le nom de la société est obligatoire."
      data: {field: sujet, value: entreprise}
  tva:
    - validate: required_with
      message: "Merci d'indiquer le n° TVA de la société."
      data: {field: societe}

  poste:
    - validate: required_if
      message: "Merci de préciser le poste visé."
      data: {field: sujet, value: recrutement}
  cv:
    - validate: required_if
      message: "Le CV est obligatoire pour une candidature."
      data: {field: sujet, value: recrutement}
    - validate: file-extensions
      message: "Formats acceptés : PDF, DOC, DOCX."
      data: {allowed: 'pdf,doc,docx'}
    - validate: file-size
      message: "5 Mo maximum."
      data: {max: 5242880}

  nom:
    - validate: required
      message: "Le nom est obligatoire."
  email:
    - validate: required
      message: "Merci d'indiquer votre adresse email."
    - validate: email
      message: "Cette adresse email n'est pas valide."
  message:
    - validate: required
      message: "Merci d'écrire un message."
  consent:
    - validate: required
      message: "Merci d'accepter le traitement de vos données."
  _hp:
    - honeypot
  g-recaptcha-response:
    - validate: recaptcha
      message: "Merci de confirmer que vous n'êtes pas un robot."

labels:
  sujet: "Demande"
  nom: "Nom"
  societe: "Société"
  tva: "N° TVA"
  poste: "Poste visé"
  email: "Adresse email"
  message: "Message"

meta:
  emailField: email
  messageField: message

success: "Merci, votre demande a bien été envoyée."
error: "Merci de corriger les champs signalés."
```

## Sites multilingues

N'écris **qu'un seul** bloc `validation`, en langue par défaut. Ne le duplique
jamais par langue : les règles (`required_if`, les limites de fichier, les
conditions) n'ont pas de langue, et les recopier oblige à les corriger dans
chaque locale.

La traduction se fait au rendu : le template passe chaque message par le filtre
`t`, donc par le **dictionnaire** du site. Le texte que tu écris dans `message`
devient la clé du dictionnaire.

```yaml
rules:
  email:
    - validate: required
      message: "Merci d'indiquer votre adresse email."   # <- clé du dictionnaire
```

Puis, dans le Dictionnaire du site, une entrée par langue :

| Clé | en | de |
|---|---|---|
| `Merci d'indiquer votre adresse email.` | Please enter your email address. | Bitte geben Sie Ihre E-Mail-Adresse ein. |

Trois conséquences :

- **Un site monolingue n'a rien à faire.** Sans entrée au dictionnaire, le
  message s'affiche tel quel.
- **Le texte est la clé.** Corriger une faute de frappe dans `message` casse ses
  traductions — il faut alors mettre à jour l'entrée du dictionnaire aussi.
- **Vaut aussi pour `success` et `error`**, qui passent par le même filtre.

Le HTML, lui, se traduit par le mécanisme habituel du contenu (une version par
langue) : il contient des libellés, des légendes et le texte du bouton.

## Diagnostiquer

**Un champ n'est jamais validé** → sa clé ne correspond pas à son `name=`. Compare les deux
listes.

**Un champ devient rouge sans message** → sa règle n'a pas de `message`. Le résumé
`{{_form.errors}}` retombe alors sur le libellé du champ, mais le visiteur n'apprend pas ce
qu'on attend de lui.

**Toutes les branches échouent sur un champ conditionnel** → il est déclaré `required` au
lieu de `required_if`.

**Aucune règle ne s'applique, aucune erreur ne s'affiche** → le YAML stocké est invalide et
la validation a été neutralisée. Ré-enregistre le bloc : le 422 te dira quoi corriger.

**Le message de succès ne s'affiche pas** → `success` est vide, ou `{{_form.messages}}` est
absent du HTML.
