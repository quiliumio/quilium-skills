# Les placeholders `{{_field.…}}` / `{{_form.…}}` — référence exhaustive

> **Avant d'écrire du HTML à la main, lis `customform-builder.md`.** Le constructeur visuel
> du backoffice ne s'ouvre que sur un formulaire décrit par la clé `builder`. Un HTML écrit
> à la main est traité comme le travail d'un intégrateur : le constructeur refuse de
> l'écraser, et **le client ne peut plus rien modifier à la souris**. N'écris le HTML
> toi-même que si le client demande un balisage que le contrat `q-form` / `q-field` ne
> permet pas — et dis-le-lui. Ce fichier reste la référence du vocabulaire, quelle que
> soit la voie.

Le HTML stocké dans le champ `formHtml` n'est **pas** compilé par Liquid ou Nunjucks —
le faire donnerait à n'importe quel éditeur du CMS l'exécution de template côté serveur.
À la place, un interpolateur dédié remplace un jeu **fermé** de motifs.

Conséquences directes, à garder en tête tout du long :

- pas de boucle, pas de condition, pas d'expression — ce qui n'est pas dans le tableau
  ci-dessous n'existe pas ;
- tout ce qui vient du visiteur est **échappé automatiquement** — à une condition, la
  règle des guillemets ci-dessous ;
- un placeholder inconnu rend une chaîne **vide** (il ne s'affiche pas tel quel), donc une
  faute de frappe se traduit par un trou silencieux — relis tes noms de champs ;
- les `{{ ... }}` qui ne commencent **pas** par `_field`, `_form`, `_page`, `_now` ou
  `_request` sont laissés intacts : ton JS inline peut contenir des accolades sans risque.

## Toujours guillemeter les attributs

L'échappement couvre les cinq entités qui rendent une valeur sûre dans du contenu **et
dans un attribut guillemeté** — simple ou double, au choix. Il ne peut pas couvrir un
attribut **non** guillemeté, et ce n'est pas un manque à combler : le moteur ne voit
qu'une chaîne, il ne sait pas si un placeholder atterrit dans un attribut ou dans du
texte. Échapper l'espace et le `=` par précaution rendrait `{{_field.email.error}}` dans un
`<p>` comme `Cette&#32;adresse&#32;email…`.

D'où une règle d'écriture, pas une option :

```html
<!-- OUI -->
<input name="email" value="{{_field.email.value}}">
<input name="email" value='{{_field.email.value}}'>

<!-- NON — une valeur postée peut ouvrir un attribut à elle -->
<input name="email" value={{_field.email.value}}>
```

Sans guillemets, une valeur comme `a onmouseover=alert(1)` devient un attribut à part
entière. Guillemète **tous** tes attributs, sans exception : c'est de toute façon la
convention HTML, et ici c'est ce qui tient la promesse d'échappement.

## Les classes viennent du site, pas d'ici

Les classes des exemples ci-dessous (`border-gray-300`, `border-red-500`…) sont des
**marqueurs neutres**. Elles ne sont là que pour montrer *où* les classes vont — remplace-les
systématiquement par celles du site que tu traites.

Trois sources, par ordre de fiabilité :

1. **un élément de formulaire déjà présent dans le thème** (`elements/contactForm.liquid`,
   `elements/newsletterCta.liquid`…) — la plus fiable, parce qu'elle donne les classes
   d'input, de label, de bouton **et les états d'erreur réellement utilisés sur ce site** ;
2. **le layout** (`layout.liquid`, `layout/base.liquid`) — palette Tailwind déclarée en
   ligne, ou lien vers le CSS compilé ;
3. **un `brand/design-system.html`** quand le projet en a un.

Si aucune n'existe, demande plutôt que d'inventer : un formulaire en classes Bootstrap sur
un site Tailwind se voit au premier coup d'œil.

Deux pièges liés :

- **Tailwind compilé purge ce qu'il ne scanne pas.** Les classes que tu écris dans le
  contenu ne sont pas dans les fichiers scannés : si le site compile Tailwind au lieu de
  charger le CDN, elles disparaissent. Vérifie comment le layout charge son CSS, et rabats-toi
  sur des classes déjà présentes dans le thème, ou sur un `<style>` dans le champ.
- **Les états d'erreur font partie du design system**, au même titre que le reste. Un
  formulaire n'est pas fini quand il s'affiche, mais quand un champ en erreur **se voit**.

## Table de référence

Les 17 placeholders, exhaustivement.

### Structure du formulaire

| Placeholder | Rend |
|---|---|
| `{{_form.action}}` | l'URL courante (chemin + querystring) — à mettre dans `action=""` |
| `{{_form.id}}` | `<input type="hidden" name="form" value="…">` |
| `{{_form.csrf}}` | `<input type="hidden" name="_csrf" value="…">`, vide si aucune règle `csrf` |
| `{{_form.recaptcha}}` | le widget reCAPTCHA + son script, vide si aucune règle `recaptcha` |

**`{{_form.id}}` est obligatoire, écris-le toujours** en première ligne du `<form>`, suivi
de `{{_form.csrf}}` :

```html
<form method="post" action="{{_form.action}}">
  {{_form.id}}
  {{_form.csrf}}
```

Rien ne l'injecte à ta place. Sans lui, la soumission n'identifie aucun bloc : **aucune
validation ne se déclenche**, aucune erreur ne s'affiche, et selon la page les actions
d'autres blocs peuvent partir à la place. L'échec est totalement silencieux — le formulaire
a l'air de fonctionner et ne valide rien.

### Valeurs saisies

| Placeholder | Rend |
|---|---|
| `{{_field.champ.value}}` | la valeur postée, échappée — pour `value=""` ou le contenu d'un `<textarea>` |
| `{{_field.champ.checked.valeur}}` | `checked` si le champ vaut cette valeur, sinon vide |
| `{{_field.champ.checked}}` | `checked` si le champ a été posté, quelle que soit sa valeur |
| `{{_field.champ.selected.valeur}}` | `selected`, même logique — pour les `<option>` |

Un champ multi-valeurs (groupe de cases, `select multiple`) est reconnu :
`{{_field.options.checked.pro}}` coche si `pro` fait partie des valeurs envoyées.

**La forme sans valeur est faite pour la case à cocher unique.** Une case sans attribut
`value` est postée par le navigateur avec la valeur `on`, un nom que tu n'as écrit nulle
part et que tu ne peux donc pas comparer :

```html
<input type="checkbox" name="consentement" {{_field.consentement.checked}}>
```

Avec une valeur explicite, garde la forme complète — elle compare littéralement.

La recherche se fait sur la **clé postée littérale**. Un champ nommé `address.city` se lit
`{{_field.address.city.value}}` — il n'y a pas de navigation dans un objet.

### Erreurs

| Placeholder | Rend |
|---|---|
| `{{_field.champ.error}}` | le premier message d'erreur du champ, vide s'il n'y en a pas |
| `{{_field.champ.class}}` | `q-field--error` si le champ est en erreur, sinon vide |
| `{{_field.champ.invalid}}` | `true` ou `false` — pour `aria-invalid` |
| `{{_form.errors}}` | la liste de **toutes** les erreurs du formulaire |
| `{{_form.class}}` | `q-form--error` si le formulaire a au moins une erreur |
| `{{_form.messages}}` | le message global de succès ou d'échec |

`{{_field.champ.class}}` et `{{_form.class}}` **ne prennent aucun argument** : ils posent
les classes du contrat Quilium, `q-field--error` et `q-form--error`. C'est l'intégrateur
du site qui les stylise dans sa feuille de style — n'écris jamais de classes de thème dans
le HTML du formulaire.

`{{_form.errors}}` produit un balisage à classes fixes, que tu styles en CSS :

```html
<ul class="q-form__errors" role="alert">
  <li class="q-form__error" data-field="email">Cette adresse email n'est pas valide.</li>
</ul>
```

Les champs pièges (`honeypot`, `empty`) en sont exclus — inutile d'exposer un piège
anti-spam au visiteur. L'exclusion porte sur **le validateur**, pas sur le message vide :
un vrai champ dont la règle n'a pas de message reste listé, avec son libellé à défaut de
phrase. C'est volontaire — un champ en erreur qui disparaît du résumé est précisément
l'impasse que ce résumé existe pour éviter.

`{{_form.class}}` applique **le même critère**. Une soumission qui ne rate que le piège —
un bot, ou un gestionnaire de mots de passe qui remplit le champ caché — ne marque donc
pas le formulaire en erreur : sinon le visiteur verrait la classe d'erreur avec un résumé
vide, et rien à corriger.

`{{_form.messages}}` produit :

```html
<div class="q-form-messages">
  <p class="q-form-message q-form-message--success">Merci, votre message a bien été envoyé.</p>
</div>
```

La classe finale vaut `--success` ou `--error` selon l'issue. Après une soumission
valide le visiteur est redirigé, et c'est au chargement suivant que le message de succès
apparaît — d'où l'intérêt de laisser `{{_form.messages}}` dans le formulaire lui-même.

### Libellés et contexte

| Placeholder | Rend |
|---|---|
| `{{_field.champ.label}}` | le libellé déclaré dans `labels:`, sinon la clé du champ |
| `{{_form.fields}}` | un tableau de tous les champs soumis — surtout utile dans un email |
| `{{_page.url}}` | l'URL absolue de la page |
| `{{_now}}` | la date et l'heure de soumission (`JJ/MM/AAAA HH:MM`) |

`{{_form.fields}}` et `{{_field.….label}}` servent essentiellement aux corps d'emails ; dans le
formulaire tu écris les libellés directement dans le markup. Voir `emails.md`.

## Ce que l'interpolateur fait pour toi

Il ajuste la balise `<form>` quand c'est nécessaire, sans jamais écraser ce que tu as
écrit :

- `method="post"` s'il n'y en a pas ;
- `enctype="multipart/form-data"` si le formulaire contient un `<input type="file">` —
  sans quoi seul le nom du fichier serait envoyé ;
- les hidden `form` et `_csrf` s'ils manquent.

Cela s'applique à **chaque** balise `<form>` du champ, pas seulement à la première : un
second formulaire sans `name="form"` déclencherait les actions de tous les blocs de la
page, et échouerait sur la règle `csrf`.

Sur un markup sans balise `<form>` (le corps d'un email), rien de tout cela ne
s'applique.

## Affichage conditionnel

Il n'y a pas de condition côté serveur : l'affichage conditionnel se fait **en CSS ou en
JavaScript**, et le champ `embed` accepte l'un comme l'autre.

Le motif éprouvé — chaque bloc porte sa condition en attribut, le script ne connaît aucun
champ en dur :

```html
<div data-when="sujet=entreprise">
  …les champs de la branche entreprise…
</div>

<script>
(function () {
  var form = document.querySelector('[data-qform]');
  if (!form) return;
  var blocks = form.querySelectorAll('[data-when]');
  function refresh() {
    blocks.forEach(function (block) {
      var parts = block.getAttribute('data-when').split('=');
      var field = form.querySelector('[name="' + parts[0] + '"]:checked')
               || form.querySelector('[name="' + parts[0] + '"]');
      var on = field && field.value === parts[1];
      block.hidden = !on;
      block.querySelectorAll('input, select, textarea')
           .forEach(function (input) { input.disabled = !on; });
    });
  }
  form.addEventListener('change', refresh);
  refresh();
})();
</script>
```

Quatre détails qui font que ça tient :

**`disabled` en plus de `hidden`.** Un champ seulement masqué reste soumis. Désactivé, il
n'est pas envoyé du tout — ce qui est exactement ce qu'attend `required_if`, et ce qui
évite de polluer l'email avec des champs vides.

**Les blocs sont visibles par défaut**, masqués par le premier `refresh()`. Sans
JavaScript, tout s'affiche et le formulaire reste utilisable. La validation serveur, elle,
ne dépend jamais du JavaScript.

**`{{_field.….checked}}` recoche la branche choisie** après une erreur, donc `refresh()` rouvre
le bon bloc tout seul et le visiteur ne perd pas son contexte.

**La validation serveur doit suivre** : un champ de branche déclaré simplement `required`
fera échouer toutes les autres branches. C'est `required_if` qu'il faut — voir
`validation-yaml.md`.

Ce motif ne gère qu'une **égalité simple** sur un champ. Pour un arbre plus riche, écris
le JavaScript qu'il faut : rien ne l'interdit.

## Le squelette d'un champ

Cinq éléments, tous nécessaires. Répète-le pour **chaque** champ, sans en sauter un seul.

```html
<div>
  <label for="cf-email" class="<classes de label du thème>">Email *</label>
  <input type="email" id="cf-email" name="email"
         value="{{_field.email.value}}"
         aria-invalid="{{_field.email.invalid}}" aria-describedby="cf-email-err"
         class="<classes d'input du thème> {{_field.email.class}}">
  <span id="cf-email-err" role="alert" class="<classe de message d'erreur du thème>">{{_field.email.error}}</span>
</div>
```

`value` pour ne pas perdre la saisie · `aria-invalid` et `aria-describedby` pour
l'accessibilité · la classe conditionnelle **sur l'input** · le message.

Trois erreurs récurrentes, toutes vues en conditions réelles :

- **`{{_field.champ.class}}` posé sur le `<div>` parent** au lieu de l'input : le conteneur change
  d'état, le champ reste visuellement normal. Seule exception légitime : une **case à
  cocher**, où une bordure rouge n'aurait pas de sens — l'état va sur son conteneur.
- **n'en mettre que sur certains champs** : on soigne les champs principaux et on oublie
  les champs conditionnels, qui sont justement ceux qui échoueront.
- **`{{_field.champ.invalid}}` confondu avec `{{_field.champ.class}}`** : le premier rend
  `true`/`false`, le second un nom de classe. Ne mets pas `.class` dans un `aria-invalid`.

## Une limite à connaître

Les entités HTML ne sont pas décodées dans un bloc `<script>`. Une valeur y est **sûre**
(elle ne peut pas fermer la balise) mais ne se relit pas telle quelle. Les placeholders
vont dans le markup, pas dans du JavaScript inline.
