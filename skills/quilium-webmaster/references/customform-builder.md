# Le modèle `builder` — construire un formulaire que le client garde en main

Lis ce fichier **avant** `customform-html.md` et `customform-validation.md`. C'est la voie
par défaut pour créer ou refaire un formulaire. Les deux autres restent la référence quand
il faut sortir du modèle — et il faut savoir ce que ça coûte.

## Pourquoi passer par le modèle

Le backoffice v2 a un **constructeur visuel** : le client ajoute des champs à la souris,
coche « obligatoire », choisit une branche conditionnelle, et le CMS régénère le HTML et
les règles. Ce constructeur ne s'ouvre que sur un YAML qui porte la clé `builder`.

Un HTML écrit à la main — sans `builder` — est traité comme le travail d'un intégrateur :
le constructeur refuse de l'ouvrir pour ne pas l'écraser. **Le client ne peut alors plus
rien modifier à la souris.** Il doit repasser par toi, ou accepter de tout perdre.

Écrire le modèle plutôt que le HTML, c'est donc laisser le formulaire modifiable. C'est
aussi la seule façon d'obtenir des règles, un HTML et des emails qui se correspondent
exactement : ils sont dérivés de la même source.

## Ce qu'il faut écrire — et pourquoi tout

La dérivation `modèle → règles → HTML` vit **dans le navigateur**, pas dans le CMS. Le CMS
valide `builder`, mais ne produit rien à partir de lui. Par MCP, tu dois donc écrire un
document **complet** :

| Champ du bloc | Contenu |
|---|---|
| `validation` | `builder` **et** `rules`, `labels`, `success`, `error` |
| `formHtml` | le HTML dérivé du modèle |
| `mailAdminBody`, `mailUserBody` | les corps d'emails (facultatif) |

Ce que le CMS **refuse** au save (422, avec la liste complète des problèmes) :

- un `builder` mal formé — type inconnu, nom de champ illégal, `select` sans options…
- un `rules` absent — il reste obligatoire, c'est lui que le moteur exécute
- un champ `required: true` dans `builder` **sans règle qui l'impose** dans `rules`. Sinon
  le formulaire afficherait un astérisque et l'attribut `required` que le serveur ne
  vérifie pas — et `required` en HTML se contourne en postant directement.
- un `builder` qui demande la protection reCAPTCHA (le défaut) **sans règle `recaptcha`**
  dans `rules`. Hors SMTP personnalisé, le moteur n'envoie aucun email si reCAPTCHA n'a
  pas été validé — sans erreur. Un formulaire qui l'oublie « marche » en local et n'envoie
  rien en production. Pour y renoncer, écris `recaptcha: false` dans `builder`.

Ce que le CMS **ne vérifie pas** : `formHtml` et les corps d'emails sont du HTML libre.
Un `{{_form.id}}` manquant ou un marqueur inexistant ne lèvent rien. Suis la recette.

## La forme du modèle

```yaml
builder:
  mode: ui                      # ui | libre — `ui` = constructeur actif
  submitLabel: Envoyer          # facultatif
  trapKey: _hp                  # facultatif ; `false` pour ne pas générer de piège
  recaptcha: true               # défaut true — écris-le ; `false` pour y renoncer
  fields:
    - key: email                # ^[A-Za-z_][A-Za-z0-9_-]{0,63}$ — ni point, ni crochet, ni espace
      type: email               # text email tel url number date textarea select radio checkbox file
      label: Adresse email      # obligatoire
      required: true
      placeholder: vous@exemple.com
      hint: Nous ne la communiquons à personne.
      messages:                 # facultatif — remplace le message par défaut d'un validateur
        required: Sans email, impossible de vous répondre.
```

Par champ, selon le type :

| Clé | Types | Sens |
|---|---|---|
| `options` | `select`, `radio` (obligatoire) — `checkbox` (facultatif) | `valeur: Libellé`, dans l'ordre d'affichage |
| `multiple` | `select`, `checkbox`, `file` | plusieurs valeurs / fichiers |
| `rows` | `textarea` | hauteur, 1 à 40 |
| `accept` | `file` | extensions, ex. `.pdf,.docx` |
| `maxSize` | `file` | plafond en **octets** (5242880 = 5 Mo), max 83886080 |
| `showIf` | tous | `{field, value}` — n'apparaît que dans cette branche |
| `requiredWith` | tous | clé d'un autre champ — exigé seulement s'il est rempli |

Trois règles de nommage, parce qu'un nom de champ voyage jusque dans les marqueurs du
HTML :

- **Ni point, ni crochet, ni espace.** Le motif de `builder` est plus strict que celui de
  `rules`. `address.city` est refusé.
- Pas de doublon.
- Un piège ne peut pas porter le nom d'un vrai champ.

Et deux exclusions :

- `showIf` et `requiredWith` **ne se combinent pas** sur un même champ : ils produiraient
  deux règles d'obligation concurrentes.
- `showIf` ne peut viser qu'un champ `select` ou `radio`, et `value` doit être une de ses
  options. Une condition sur un texte libre ne serait jamais fiable.

## La recette de dérivation — `builder` → `rules`

C'est exactement ce que fait le constructeur. Applique-la champ par champ, dans l'ordre :

```
required: true, sans condition   →  - validate: required
required: true + showIf          →  - validate: required_if   data: {field, value}
required: true + requiredWith    →  - validate: required_with data: {field}
type: email                      →  - validate: email                    (en plus)
type: file + accept              →  - validate: file-extensions data: {allowed: "pdf,docx"}
type: file + maxSize             →  - validate: file-size       data: {max: 5242880}
piège (trapKey, défaut _hp)      →  _hp: [- validate: honeypot]         (sans message)
recaptcha (défaut true)          →  g-recaptcha-response: [- validate: recaptcha]
```

Pour `allowed`, retire les points et les espaces de `accept` : `.pdf, .docx` → `pdf,docx`.

Chaque règle porte un `message`, sauf le piège. Sans message dans `messages:`, le défaut :

| Validateur | Message par défaut |
|---|---|
| `required`, `required_if`, `required_with` | `Le champ « <label> » est obligatoire.` |
| `email` | `Cette adresse email n'est pas valide.` |
| `file-extensions` | `Formats acceptés : <accept>.` |
| `file-size` | `Fichier trop lourd (<N> Mo maximum).` |
| `recaptcha` | `Merci de cocher la case « Je ne suis pas un robot ».` |

`labels` reprend `key: label` pour chaque champ. C'est ce que le tableau des emails et
`{{_field.<champ>.label}}` affichent.

## Exemple complet 1 — un formulaire de contact

Quatre champs. Voici les **quatre valeurs à écrire**, exactement ce que le constructeur
produirait.

### `validation`

```yaml
builder:
  mode: ui
  recaptcha: true
  fields:
    - key: nom
      type: text
      label: Nom
      required: true
    - key: email
      type: email
      label: Adresse email
      required: true
      placeholder: vous@exemple.com
    - key: message
      type: textarea
      label: Message
      required: true
      rows: 5
    - key: consent
      type: checkbox
      label: J'accepte que mes données soient utilisées pour traiter ma demande
      required: true
rules:
  nom:
    - validate: required
      message: Le champ « Nom » est obligatoire.
  email:
    - validate: required
      message: Le champ « Adresse email » est obligatoire.
    - validate: email
      message: Cette adresse email n'est pas valide.
  message:
    - validate: required
      message: Le champ « Message » est obligatoire.
  consent:
    - validate: required
      message: Le champ « J'accepte que mes données soient utilisées pour traiter ma demande » est obligatoire.
  _hp:
    - validate: honeypot
  g-recaptcha-response:
    - validate: recaptcha
      message: Merci de cocher la case « Je ne suis pas un robot ».
labels:
  nom: Nom
  email: Adresse email
  message: Message
  consent: J'accepte que mes données soient utilisées pour traiter ma demande
success: Merci, votre message a bien été envoyé.
error: Merci de corriger les champs signalés.
```

### `formHtml`

```html
<form class="q-form {{_form.class}}" method="post" action="{{_form.action}}" novalidate>
  {{_form.id}}
  {{_form.csrf}}
  {{_form.messages}}
  {{_form.errors}}

  <div class="q-field q-field--text {{_field.nom.class}}">
    <label class="q-field__label" for="qf-nom">Nom <span class="q-field__required">*</span></label>
    <input type="text" id="qf-nom" name="nom" class="q-field__control" aria-invalid="{{_field.nom.invalid}}" required value="{{_field.nom.value}}">
    <span class="q-field__error">{{_field.nom.error}}</span>
  </div>

  <div class="q-field q-field--email {{_field.email.class}}">
    <label class="q-field__label" for="qf-email">Adresse email <span class="q-field__required">*</span></label>
    <input type="email" id="qf-email" name="email" class="q-field__control" aria-invalid="{{_field.email.invalid}}" required placeholder="vous@exemple.com" value="{{_field.email.value}}">
    <span class="q-field__error">{{_field.email.error}}</span>
  </div>

  <div class="q-field q-field--textarea {{_field.message.class}}">
    <label class="q-field__label" for="qf-message">Message <span class="q-field__required">*</span></label>
    <textarea id="qf-message" name="message" class="q-field__control" aria-invalid="{{_field.message.invalid}}" required rows="5">{{_field.message.value}}</textarea>
    <span class="q-field__error">{{_field.message.error}}</span>
  </div>

  <div class="q-field q-field--checkbox {{_field.consent.class}}">
    <label class="q-field__choice"><input type="checkbox" name="consent" value="on" {{_field.consent.checked.on}} required aria-invalid="{{_field.consent.invalid}}"> J&#39;accepte que mes données soient utilisées pour traiter ma demande</label>
    <span class="q-field__error">{{_field.consent.error}}</span>
  </div>

  <div class="q-form__trap" aria-hidden="true">
    <input type="text" name="_hp" tabindex="-1" autocomplete="off" value="{{_field._hp.value}}">
  </div>

  {{_form.recaptcha}}
  <div class="q-form__actions">
    <button type="submit" class="q-form__submit">Envoyer</button>
  </div>
</form>
```

Ce HTML ne porte **aucune classe de thème** — seulement le contrat `q-form` / `q-field`.
C'est l'intégrateur du site qui le stylise. N'y mets jamais de classe Tailwind ou
Bootstrap : ce serait perdu à la prochaine régénération.

### `mailAdmin` (sujet) et `mailAdminBody`

```
subject : Nouveau message de {{nom}}
```

```html
<p>Nouvelle demande reçue depuis {{_page.url}} le {{_now}}.</p>
{{_form.fields}}
```

### `mailUser` (sujet) et `mailUserBody`

```
subject : Nous avons bien reçu votre message
```

```html
<p>Bonjour {{_field.nom.value}},</p>
<p>Nous avons bien reçu votre message le {{_now}} et nous y répondrons rapidement.</p>
<p>Voici le récapitulatif de votre envoi :</p>
{{_form.fields}}
```

Remarque le sujet et le corps : **`{{nom}}` dans le sujet, `{{_field.nom.value}}` dans le
corps.** Ce n'est pas une coquille — voir « Les emails » plus bas.

## Exemple complet 2 — un devis à trois branches

Le cas qui justifie le constructeur : une radio ouvre une branche, chaque branche a ses
champs obligatoires, un champ n'est exigé que si un autre est rempli.

### `validation`

```yaml
builder:
  mode: ui
  submitLabel: Envoyer ma demande
  recaptcha: true
  fields:
    - key: sujet
      type: radio
      label: Votre demande
      required: true
      options:
        particulier: Particulier
        entreprise: Entreprise
        recrutement: Candidature
    - key: nom
      type: text
      label: Nom
      required: true
    - key: societe
      type: text
      label: Société
      required: true
      showIf:
        field: sujet
        value: entreprise
    - key: tva
      type: text
      label: N° TVA
      required: true
      requiredWith: societe
      hint: Obligatoire dès que la société est renseignée.
    - key: poste
      type: text
      label: Poste visé
      required: true
      showIf:
        field: sujet
        value: recrutement
    - key: cv
      type: file
      label: CV
      required: true
      showIf:
        field: sujet
        value: recrutement
      accept: .pdf,.doc,.docx
      maxSize: 5242880
    - key: email
      type: email
      label: Adresse email
      required: true
    - key: message
      type: textarea
      label: Message
      required: true
      rows: 4
    - key: consent
      type: checkbox
      label: J'accepte que mes données soient utilisées pour traiter ma demande
      required: true
rules:
  sujet:
    - validate: required
      message: Le champ « Votre demande » est obligatoire.
  nom:
    - validate: required
      message: Le champ « Nom » est obligatoire.
  societe:
    - validate: required_if
      message: Le champ « Société » est obligatoire.
      data:
        field: sujet
        value: entreprise
  tva:
    - validate: required_with
      message: Le champ « N° TVA » est obligatoire.
      data:
        field: societe
  poste:
    - validate: required_if
      message: Le champ « Poste visé » est obligatoire.
      data:
        field: sujet
        value: recrutement
  cv:
    - validate: required_if
      message: Le champ « CV » est obligatoire.
      data:
        field: sujet
        value: recrutement
    - validate: file-extensions
      message: "Formats acceptés : .pdf,.doc,.docx."
      data:
        allowed: pdf,doc,docx
    - validate: file-size
      message: Fichier trop lourd (5 Mo maximum).
      data:
        max: 5242880
  email:
    - validate: required
      message: Le champ « Adresse email » est obligatoire.
    - validate: email
      message: Cette adresse email n'est pas valide.
  message:
    - validate: required
      message: Le champ « Message » est obligatoire.
  consent:
    - validate: required
      message: Le champ « J'accepte que mes données soient utilisées pour traiter ma demande » est obligatoire.
  _hp:
    - validate: honeypot
  g-recaptcha-response:
    - validate: recaptcha
      message: Merci de cocher la case « Je ne suis pas un robot ».
labels:
  sujet: Votre demande
  nom: Nom
  societe: Société
  tva: N° TVA
  poste: Poste visé
  cv: CV
  email: Adresse email
  message: Message
  consent: J'accepte que mes données soient utilisées pour traiter ma demande
success: Merci, votre demande a bien été envoyée.
error: Merci de corriger les champs signalés ci-dessous.
```

Observe la dérivation : `societe` est `required` + `showIf` → `required_if`. `tva` est
`required` + `requiredWith` → `required_with`. `cv` cumule `required_if`,
`file-extensions` et `file-size`.

### Ce que le HTML ajoute pour les branches

Un champ `showIf` reçoit `data-when="sujet=entreprise"` sur son conteneur, et le
formulaire se termine par un script qui **masque et désactive** la branche non choisie.
Désactiver n'est pas cosmétique : un champ masqué mais actif serait posté, et apparaîtrait
dans l'email.

Un champ `file` pose `enctype="multipart/form-data"` sur le `<form>`. Sans lui, le
navigateur n'envoie que le nom du fichier.

Le reste du HTML suit exactement la structure de l'exemple 1. Pour un `radio` :

```html
<div class="q-field q-field--radio {{_field.sujet.class}}">
  <fieldset class="q-fieldset" aria-invalid="{{_field.sujet.invalid}}">
    <legend class="q-field__label">Votre demande <span class="q-field__required">*</span></legend>
    <label class="q-field__choice"><input type="radio" name="sujet" value="particulier" {{_field.sujet.checked.particulier}}> Particulier</label>
    <label class="q-field__choice"><input type="radio" name="sujet" value="entreprise" {{_field.sujet.checked.entreprise}}> Entreprise</label>
    <label class="q-field__choice"><input type="radio" name="sujet" value="recrutement" {{_field.sujet.checked.recrutement}}> Candidature</label>
  </fieldset>
  <span class="q-field__error">{{_field.sujet.error}}</span>
</div>
```

Pour une branche :

```html
<div class="q-field q-field--text {{_field.societe.class}}" data-when="sujet=entreprise">
  …
</div>
```

Et le script, à placer **dans** le `<form>`, avant `</form>` :

```html
<script>
(function () {
  var script = document.currentScript
  var form = script && script.closest ? script.closest('form') : null
  if (!form) return
  var blocks = form.querySelectorAll('[data-when]')
  if (!blocks.length) return
  function valueOf (name) {
    var nodes = form.querySelectorAll('[name="' + name + '"]')
    for (var i = 0; i < nodes.length; i++) {
      var node = nodes[i]
      if (node.type === 'radio' || node.type === 'checkbox') {
        if (node.checked) return node.value
      } else {
        return node.value
      }
    }
    return ''
  }
  function refresh () {
    for (var i = 0; i < blocks.length; i++) {
      var block = blocks[i]
      var when = block.getAttribute('data-when') || ''
      var at = when.indexOf('=')
      if (at < 0) continue
      var on = valueOf(when.slice(0, at)) === when.slice(at + 1)
      block.hidden = !on
      var inputs = block.querySelectorAll('input, select, textarea')
      for (var j = 0; j < inputs.length; j++) inputs[j].disabled = !on
    }
  }
  form.addEventListener('change', refresh)
  form.addEventListener('input', refresh)
  refresh()
})()
</script>
```

## Le contrat HTML, en une page

Le générateur produit toujours cette structure. Reproduis-la : le client qui ouvre le
constructeur la régénérera à l'identique, sans rupture visible.

```
<form class="q-form {{_form.class}}" method="post" action="{{_form.action}}" novalidate [enctype="multipart/form-data"]>
  {{_form.id}}            ← TOUJOURS. Sans lui, aucune validation ne tourne, sans erreur.
  {{_form.csrf}}
  {{_form.messages}}
  {{_form.errors}}

  <div class="q-field q-field--<type> {{_field.<key>.class}}" [data-when="<champ>=<valeur>"]>
    <label class="q-field__label" for="qf-<key>">…</label>       ← sauf case unique et groupes
    <… name="<key>" class="q-field__control" aria-invalid="{{_field.<key>.invalid}}" …>
    <span class="q-field__error">{{_field.<key>.error}}</span>
  </div>

  <div class="q-form__trap" aria-hidden="true"><input name="_hp" …></div>
  {{_form.recaptcha}}
  <div class="q-form__actions"><button type="submit" class="q-form__submit">…</button></div>
</form>
```

Par type de contrôle :

| Type | Contrôle | Valeur collante |
|---|---|---|
| text, email, tel, url, number, date | `<input type="…" value="{{_field.k.value}}">` | attribut `value` |
| textarea | `<textarea>{{_field.k.value}}</textarea>` | contenu |
| select | `<option value="v" {{_field.k.selected.v}}>` | `selected` par option |
| radio | `<fieldset>` + `<legend>` + `<input type="radio" {{_field.k.checked.v}}>` | `checked` par option |
| checkbox seule | `<label class="q-field__choice"><input value="on" {{_field.k.checked.on}}>` | pas de label au-dessus |
| checkbox groupée | `<fieldset>` + `<input name="k[]" {{_field.k.checked.v}}>` | `checked` par option |
| file | `<input type="file" accept="…">` — **jamais** de `value` | aucune |

Un champ `file` n'a pas de valeur collante : le navigateur interdit d'y écrire un chemin.

## Les emails

Le sujet et le corps ne parlent **pas le même langage**. C'est le piège le plus fréquent.

| Endroit | Un champ s'écrit | Pourquoi |
|---|---|---|
| `to`, `replyTo`, `subject` | `{{nom}}` — accolades serrées | mapping direct du moteur |
| corps (`mailAdminBody`, `mailUserBody`) | `{{_field.nom.value}}` ou `{{_request.body.nom}}` | interpolé par le thème |

**Un `{{nom}}` nu dans un corps reste affiché tel quel dans l'email reçu.** Liquid ne
ré-analyse pas une chaîne qu'il vient de produire.

Dans un corps, tout le vocabulaire du formulaire est disponible :

| Marqueur | Rend |
|---|---|
| `{{_form.fields}}` | le tableau de toutes les réponses, libellés inclus, pièges et plomberie exclus |
| `{{_field.<champ>.value}}` | une réponse |
| `{{_request.body.<champ>}}` | la même, forme brute |
| `{{_field.<champ>.label}}` | le libellé du champ |
| `{{_page.url}}` | la page d'où vient l'envoi |
| `{{_now}}` | la date et l'heure de soumission |

Un champ inexistant rend du vide, sans erreur. Vérifie chaque clé citée contre `builder`.

Le sujet ne peut citer qu'un champ qui **existe** : `Nouveau message de {{prenom}}` sur un
formulaire sans champ `prenom` part avec un trou, et personne ne le voit avant de lire
l'email.

## reCAPTCHA

Le squelette du formulaire contient toujours `{{_form.recaptcha}}`, juste avant le
bouton : le thème y rend le widget dès qu'une règle `recaptcha` existe, et rien sinon.
Les clés reCAPTCHA sont une configuration du moteur, pas du site — tu n'as rien à poser.
Ce qui dépend de toi : la règle dans `rules`, dérivée de `recaptcha: true`. Sans elle,
en production, l'email ne part pas et personne n'est prévenu.

## Quand sortir du modèle

Écris le HTML à la main **seulement** si le client veut un balisage que le contrat ne
permet pas — une mise en page sur mesure, un composant tiers. Alors :

- n'écris **pas** de `builder`, ou mets `mode: libre` ;
- suis `customform-html.md` pour le vocabulaire, et garde `{{_form.id}}` ;
- **dis au client** que son formulaire ne sera plus modifiable dans le constructeur.

Le silence sur ce point est ce qui coûte le plus cher ensuite.
