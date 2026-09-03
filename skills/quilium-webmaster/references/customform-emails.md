# Les emails — référence exhaustive

Un `customform` envoie ses emails via des actions `sendmail`. Le mécanisme est celui de
tous les formulaires Quilium ; ce qui change, c'est que **le corps du mail est lui aussi
un champ de contenu**, avec les mêmes placeholders que le formulaire.

## Trois couches, trois responsables

| Couche | Où | Qui la modifie | Fréquence |
|---|---|---|---|
| Layout — branding, en-tête, pied | `mails/_layout.liquid` dans le thème | développeur | une fois par site |
| Enveloppe — charge le corps | `mails/customform-admin.liquid` | développeur | une fois, ne rebouge plus |
| **Corps** | champ `embed` du bloc | **toi / le client** | à volonté |

L'enveloppe fait quatre lignes et ne change plus jamais :

```liquid
{% layout 'mails/_layout' %}
{% block content %}{{ _content.custom.mailAdminBody | quilium_form: _content }}{% endblock %}
```

Si elle n'existe pas encore, voir `setup.md`.

## Ce qui se règle par instance

| Réglage | Où |
|---|---|
| Destinataire (`to`) | champ `to` de l'action `sendmail`, sur le bloc |
| Objet (`subject`) | idem |
| Adresse de réponse (`replyTo`) | idem |
| Corps | champ `embed` (`mailAdminBody`, `mailUserBody`) |
| **Envoyer ou non** | vider `to` — l'action est alors sautée |

Via MCP :

```
update-content
  custom: '{"mailAdmin": {"to": "contact@site.lu",
                          "subject": "Nouveau message de {{prenom}} {{nom}}",
                          "replyTo": "{{email}}"}}'
```

`mailAdmin` est le **nom du champ `sendmail`**, à lire dans le content-type.

### Accolades serrées

Dans `to`, `subject` et `replyTo`, les champs postés s'interpolent avec des accolades
**sans espaces** : `{{prenom}}`, jamais `{{ prenom }}`. Ces trois valeurs passent par un
mécanisme différent des placeholders `q:`, plus strict. Une accolade espacée reste
affichée telle quelle dans l'objet du mail.

L'expéditeur (`From`) n'est pas modifiable par instance : il vaut l'expéditeur Quilium du
site, sauf SMTP personnalisé déclaré dans le YAML du type de contenu.

## Écrire un corps de mail

Le corps utilise **le même vocabulaire que le formulaire** — rien de nouveau à apprendre.

Le plus robuste, et le défaut recommandé :

```html
<p>Nouvelle demande reçue depuis {{q:page}} le {{q:date}}.</p>
{{q:fields}}
```

`{{q:fields}}` produit un tableau de tous les champs soumis. Son intérêt : le mail reste
exhaustif **quand un champ est ajouté au formulaire**, sans que personne ne touche au
corps ni au template.

```html
<table class="q-fields">
  <tr><th align="left">Adresse email</th><td>jean@example.lu</td></tr>
  <tr><th align="left">Message</th><td>Bonjour…</td></tr>
</table>
```

Il exclut d'office la plomberie (`form`, `_csrf`, `g-recaptcha-response`, `update_*`) et
les champs pièges déclarés `honeypot` ou `empty`. Les champs vides sont omis pour que le
mail reste lisible.

Un champ nommé `id` **n'est pas** exclu : c'est plus souvent une référence métier (numéro
de commande, numéro de dossier) qu'une clé technique, et le perdre du mail de notification
sans trace coûte plus cher que de l'afficher.

Les libellés viennent de `labels:` dans le YAML de validation. Sans eux, les clés brutes
s'affichent (`prenom` au lieu de « Prénom ») — c'est le premier reproche qu'on fait à un
mail généré.

L'ordre des lignes suit l'ordre de `labels:`, puis l'ordre de soumission pour le reste.

Pour un accusé de réception, une rédaction sur mesure passe mieux :

```html
<p>Bonjour {{q:value:prenom}},</p>
<p>Nous avons bien reçu votre message et vous répondrons rapidement.</p>
{{q:fields}}
```

Les placeholders propres au formulaire (`{{q:csrf}}`, `{{q:error:…}}`, `{{q:messages}}`)
rendent une chaîne vide dans un mail — inoffensifs, mais inutiles.

## Les deux gardes qui bloquent un envoi

Ce sont elles qui expliquent la quasi-totalité des « le mail ne part pas ». Aucune des
deux ne produit d'erreur visible : l'envoi est simplement sauté.

### 1. Sans reCAPTCHA, aucun mail

Hors SMTP personnalisé, `sendmail` exige que reCAPTCHA ait été validé. Sans lui, la
fonction sort en silence.

**À faire systématiquement** : la règle `g-recaptcha-response: [recaptcha]` dans le YAML,
et `{{q:recaptcha}}` dans le HTML.

Le contournement local (`NODE_ENV=local`) fait passer l'envoi en développement — donc un
test local ne prouve rien sur ce point.

### 2. Un destinataire dynamique exige Cleantalk

L'accusé de réception au visiteur a forcément `to: {{all.email}}`. Or un `to` contenant une
accolade est refusé si Cleantalk n'a pas tourné — c'est une protection anti-relais ouvert :
sans analyse anti-spam, on n'autorise pas un formulaire public à faire envoyer un mail à
une adresse arbitraire.

Pour l'activer :

1. la clé d'API dans le YAML du type de contenu (`config.cleantalk.key`) — un secret, un
   par site ;
2. `meta.emailField` et `meta.messageField` dans le YAML de contenu — Cleantalk exige un
   message à analyser, et c'est `meta` qui le désigne ;
3. facultatif : `cleantalk: {silent: false}` pour que le spam bloque au lieu d'être
   silencieusement écarté.

Sans `meta`, Cleantalk ne s'exécute pas du tout, et l'accusé de réception ne part donc
jamais en production. Là encore, le contournement local masque le problème.

## Diagnostiquer

**Aucun mail ne part** → pas de règle `recaptcha`, ou `to` vide.

**Le mail admin part, pas celui du visiteur** → Cleantalk n'est pas configuré (clé
manquante, ou `meta` absent).

**Le tableau `{{q:fields}}` est vide** → le moteur qui rend les mails n'est pas à jour ; ou
le corps a été écrit dans le mauvais champ.

**Les libellés affichent les clés brutes** → `labels:` manque dans le YAML de validation.

**L'objet du mail contient `{{ prenom }}` littéralement** → accolades espacées ; il en faut
des serrées.

**Rien n'arrive dans le mailcatcher attendu** → plusieurs mailcatchers coexistent souvent
en développement. Vérifie sur quel port SMTP pointe le moteur avant de conclure à un
non-envoi.
