# Aeris — backend paiement Stripe + envoi automatique à CJ Dropshipping

## Démarrage rapide

```bash
./setup.sh
```

Ce script vous demande uniquement vos 3 clés (Stripe, CJ, domaine — les
seules choses que je ne peux pas connaître à votre place), installe les
dépendances, et lance automatiquement la recherche des `vid` CJ pour vos 4
produits. Il reste ensuite 2 choses à faire à la main (affichées à la fin
du script) : coller les `vid` dans `lib/products.js`, et configurer le
webhook Stripe (section 3 ci-dessous).

## Ce que fait ce backend

1. `POST /api/create-checkout-session` — crée une session **Stripe Checkout**
   (page de paiement hébergée par Stripe) à partir du panier envoyé par le
   site, en revérifiant les prix côté serveur.
2. `POST /api/webhook` — reçoit la confirmation de Stripe une fois le
   paiement encaissé, puis crée automatiquement la commande chez
   **CJ Dropshipping** (produit + adresse du client), payée via le solde de
   votre compte CJ pour un envoi sans intervention manuelle.
3. Sert le site (`public/index.html`, copie de votre fichier avec le bouton
   "Passer la commande" branché sur Stripe au lieu du faux modal).
4. `GET /api/order-status/:sessionId` — vérifie le paiement Stripe et
   récupère le statut d'expédition réel côté CJ (page `public/suivi.html`).
5. Pages légales prêtes (à personnaliser) : `mentions-legales.html`,
   `cgv.html`, `politique-confidentialite.html`, `politique-retours.html` —
   liées dans le pied de page du site. **Elles contiennent des `[à
   compléter]`** (SIRET, adresse, e-mail...) : je ne peux pas inventer vos
   informations légales réelles, et une relecture par un professionnel du
   droit est recommandée avant mise en ligne.

## 1. Installer

```bash
npm install
cp .env.example .env
```

Remplissez `.env` :
- `STRIPE_SECRET_KEY` — Dashboard Stripe → Developers → API keys
- `STRIPE_WEBHOOK_SECRET` — voir étape 3
- `CJ_API_KEY` — MyCJ → Settings → API Key (générez-la si besoin)
- `DOMAIN` — l'URL publique de votre site une fois déployé

## 2. Renseigner les identifiants produits CJ

Ouvrez `lib/products.js`. Chaque produit a un champ `cjVid: null`.
Il faut le remplacer par le `vid` (variant ID) CJ du produit correspondant.

**Un script fait le travail pour vous** — vous devez juste le lancer avec
votre propre `CJ_API_KEY` (personne d'autre ne peut le faire à votre place,
il faut vos identifiants réels) :

```bash
node scripts/find-cj-vid.js "purificateur d'air voiture"
```

Il affiche les produits CJ correspondants avec leur(s) `vid`. Copiez le bon
`vid` dans `lib/products.js`, champ `cjVid`, pour chacun des 4 produits.

**Tant qu'un `cjVid` est `null`, la commande Stripe fonctionne normalement,
mais la commande n'est PAS envoyée à CJ automatiquement** — le webhook logue
une erreur explicite pour que vous puissiez la traiter à la main. C'est
volontaire : mieux vaut un échec visible qu'une commande silencieusement
perdue.

## 3. Configurer le webhook Stripe

En local, avec la [Stripe CLI](https://docs.stripe.com/stripe-cli) :

```bash
stripe listen --forward-to localhost:3000/api/webhook
```

Elle affiche un `whsec_...` à mettre dans `STRIPE_WEBHOOK_SECRET`.

En production, dans le Dashboard Stripe → Developers → Webhooks → Add
endpoint :
- URL : `https://votre-domaine.com/api/webhook`
- Événement à écouter : `checkout.session.completed`
- Copiez le "Signing secret" affiché dans `STRIPE_WEBHOOK_SECRET`

## 4. Lancer

```bash
npm start
```

Le site est servi sur `http://localhost:3000`.

## 5. Solde du compte CJ

L'envoi 100% automatique utilise le paiement par **solde CJ**
(`payType: 2` dans `lib/cjClient.js`). Rechargez votre solde dans
MyCJ → Wallet. Sans solde suffisant, la création de commande CJ échouera
(erreur loguée), et il faudra soit recharger, soit passer temporairement
en `payType: 1` (CJ renvoie un lien de paiement à régler manuellement,
moins automatique mais fonctionne sans solde préchargé).

## 6. Déploiement

Deux options prêtes à l'emploi sont incluses :

**Render** (le plus simple) — `render.yaml` est déjà configuré :
1. Poussez ce dossier sur un dépôt Git (GitHub/GitLab)
2. Sur [render.com](https://render.com) → New → Blueprint → sélectionnez le dépôt
3. Render détecte `render.yaml` et crée le service automatiquement
4. Renseignez les variables d'environnement marquées `sync: false` dans
   l'interface Render (elles ne sont jamais dans le fichier, pour rester secrètes)
5. Une fois déployé, copiez l'URL Render dans `DOMAIN`, et utilisez
   `https://votre-app.onrender.com/api/webhook` comme URL de webhook Stripe

**Docker / VPS** — `Dockerfile` fourni :
```bash
docker build -t aeris-backend .
docker run -p 3000:3000 --env-file .env aeris-backend
```
Fonctionne sur n'importe quel hébergeur supportant Docker (VPS avec Docker,
Fly.io, Railway, etc.).

Dans tous les cas, points d'attention :
- Variables d'environnement à définir sur la plateforme (pas de fichier
  `.env` en prod)
- `DOMAIN` doit correspondre à l'URL publique réelle
- Le webhook Stripe doit pointer vers cette URL publique, pas
  `localhost`

## Ce qui reste à votre charge

- Un compte Stripe activé (vérification d'identité/entreprise requise pour
  passer en mode "live")
- Un compte CJ Dropshipping avec du solde
- La politique de retours/remboursement réelle (le site l'annonce déjà,
  mais aucune logique de remboursement automatique n'est implémentée ici)
- La conformité légale de la boutique (CGV, mentions légales, RGPD pour les
  données clients transmises à Stripe et CJ)

