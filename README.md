# SKB BHV — echte Stripe-betalingen

Dit project koppelt de SKB BHV-website aan **echte** Stripe-betalingen (iDEAL, creditcard,
Bancontact), via drie kleine serverless functies. De website zelf (`public/index.html`) is
dezelfde site als je demo-versie, met één verschil: de betaalstap stuurt de klant nu écht
naar Stripe Checkout in plaats van een nagebootste betaling te simuleren.

## Hoe het werkt

1. Klant vult zijn gegevens in en klikt op **"Betaal veilig via Stripe"**.
2. De browser roept `POST /api/create-checkout-session` aan → deze functie maakt een
   Stripe Checkout Session aan met de juiste pakket-prijs, korting, BHV-pasje en
   locatietoeslag, en stuurt de klant door naar de beveiligde Stripe-betaalpagina.
3. Na betalen stuurt Stripe de klant terug naar
   `https://jouw-domein/?payment=success&session_id=...`
4. De site roept dan `GET /api/verify-session` aan om **server-side** te bevestigen dat er
   echt betaald is (nooit de browser blindelings vertrouwen), en bouwt daarna het
   deelnemersdashboard op.
5. `api/webhook.js` ontvangt daarnaast een server-naar-server melding van Stripe zelf zodra
   een betaling is voltooid — handig voor het opslaan van bestellingen in een database of
   het versturen van een bevestigingsmail, ook als de klant zijn browser dichtklikt vóórdat
   stap 3/4 plaatsvindt.

## Stap 1 — Stripe-account

1. Maak een gratis account op [stripe.com](https://stripe.com) (of log in als je er al een hebt).
2. Zet in het Dashboard rechtsboven **"Test mode"** aan zolang je nog aan het testen bent.
3. Ga naar **Developers → API keys** en kopieer de **Secret key** (begint met `sk_test_...`).
4. Ga naar **Settings → Payment methods** en zet **iDEAL** en **Bancontact** aan (creditcard
   staat meestal al standaard aan).

## Stap 2 — Project deployen op Vercel

De makkelijkste manier om deze backend + website live te zetten is via
[Vercel](https://vercel.com) (gratis voor dit soort kleine projecten).

1. Zet deze hele map (`skb-bhv-stripe/`) in een git-repository (GitHub/GitLab/Bitbucket).
2. Ga naar [vercel.com/new](https://vercel.com/new) en importeer die repository.
3. Vercel herkent automatisch de `api/`-map als serverless functions en `public/` als de
   statische site — je hoeft verder niets te configureren.
4. Voeg vóór de eerste deploy de environment variables toe (zie Stap 3).
5. Klik op **Deploy**. Na een paar seconden krijg je een URL zoals
   `https://skb-bhv-jouwnaam.vercel.app`.

> Liever een andere host (Netlify, Cloudflare Pages, een eigen server)? De `api/`-functies
> zijn standaard Node.js-functies met een `(req, res)`-signatuur; die werken met kleine
> aanpassingen ook op de meeste andere platforms.

## Stap 3 — Environment variables instellen

Zowel lokaal (in een `.env`-bestand, gebaseerd op `.env.example`) als in Vercel
(**Project Settings → Environment Variables**) heb je nodig:

| Variabele | Waarde | Waar te vinden |
|---|---|---|
| `STRIPE_SECRET_KEY` | `sk_test_...` (later `sk_live_...`) | Stripe Dashboard → Developers → API keys |
| `STRIPE_WEBHOOK_SECRET` | `whsec_...` | Zie Stap 4 hieronder |
| `SITE_URL` | bijv. `https://skb-bhv-jouwnaam.vercel.app` | Je eigen Vercel-URL, **zonder** trailing slash |

Na het toevoegen van environment variables in Vercel: doe een nieuwe deploy (Vercel past ze
pas toe vanaf de eerstvolgende build).

## Stap 4 — Webhook registreren (aanbevolen)

1. Ga in het Stripe Dashboard naar **Developers → Webhooks → Add endpoint**.
2. Endpoint-URL: `https://jouw-domein.vercel.app/api/webhook`
3. Selecteer in elk geval het event **`checkout.session.completed`**
   (en optioneel `checkout.session.expired`).
4. Na het aanmaken toont Stripe een **Signing secret** (`whsec_...`) — zet die als
   `STRIPE_WEBHOOK_SECRET` in je environment variables (Stap 3) en deploy opnieuw.

Wil je eerst lokaal testen zonder alles te deployen? Gebruik de
[Stripe CLI](https://stripe.com/docs/stripe-cli):
```
stripe listen --forward-to localhost:3000/api/webhook
```
Dit print een tijdelijke `whsec_...` die je lokaal kunt gebruiken.

## Stap 5 — Testen

In test-mode accepteert Stripe nooit echte kaarten. Gebruik testkaart `4242 4242 4242 4242`,
een willekeurige toekomstige vervaldatum en een willekeurige CVC. Voor iDEAL kun je in
test-mode een gesimuleerde bank kiezen die de betaling laat slagen of mislukken.

Loop het hele proces door: pakket kiezen → gegevens invullen → "Betaal veilig via Stripe" →
testbetaling afronden → controleren of je automatisch teruggestuurd wordt naar het dashboard
met de juiste deelnemersplekken klaarstaand.

## Stap 6 — Live gaan

1. Zet in het Stripe Dashboard **"Test mode"** uit.
2. Kopieer je **live** Secret key (`sk_live_...`) en vervang `STRIPE_SECRET_KEY` daarmee.
3. Herhaal Stap 4 (webhook) voor de live-omgeving — test- en live-webhooks zijn apart.
4. Doe een nieuwe deploy.

## Bestandenoverzicht

```
skb-bhv-stripe/
├── api/
│   ├── create-checkout-session.js   # Maakt de Stripe Checkout Session aan
│   ├── verify-session.js            # Bevestigt server-side dat er betaald is
│   └── webhook.js                   # Ontvangt betaalbevestigingen rechtstreeks van Stripe
├── public/
│   └── index.html                   # De volledige SKB BHV-website (met echte Stripe-koppeling)
├── .env.example                     # Voorbeeld van de benodigde environment variables
├── package.json
├── vercel.json
└── README.md                        # Dit bestand
```

## Nog openstaand / aan te vullen

- **E-mailbevestiging**: `api/webhook.js` heeft een `TODO` waar je een bevestigingsmail naar
  de klant én naar info@skbbhv.nl kunt versturen (bijv. via [Resend](https://resend.com) of
  [Postmark](https://postmarkapp.com)).
- **Bestellingen opslaan**: momenteel wordt een bestelling alleen bewaard in de browser van
  de klant (via de ingebouwde `window.storage`). Voor een echt bedrijfsproces wil je
  bestellingen ook server-side opslaan (bijv. in een Postgres-database via
  [Vercel Postgres](https://vercel.com/storage/postgres) of [Supabase](https://supabase.com)) —
  `api/webhook.js` is de logische plek om dat toe te voegen.
- **Facturen**: de factuurpagina op de site genereert nu zelf een factuur in de browser. Wil
  je liever dat Stripe automatisch een factuur aanmaakt en mailt? Zet dan
  `invoice_creation: { enabled: true }` aan in `api/create-checkout-session.js` (staat er
  als commentaar klaar).
