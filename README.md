# Gaitz — dynamic website (Next.js)

This is the real, database-backed version of the site: Next.js app, Postgres
database, dealer login with real pricing and ordering, and a WhatsApp alert
the moment someone submits the Contact form.

## What's in here

- **Public site** — Home, Products (filterable), Product detail, About, Contact — same design as the earlier static prototype, now pulling every product from the database.
- **Dealer portal** (`/portal`) — distributors log in and see dealer-only pricing, place an order (multiple products/quantities in one go), and view their order history.
- **Admin dashboard** (`/admin`) — enquiries inbox with status tracking, dealer orders with status updates, and a products list where you can edit list/dealer price and live/draft status.
- **WhatsApp alerts** — every Contact-form submission and every dealer order pings your team's WhatsApp via Twilio.
- **Login** — one login page for both roles; where you land depends on whether your account is Admin or Distributor.

## Login vs. enquiries

Login is only ever required for the dealer portal (`/portal`) and the admin dashboard (`/admin`) — nobody needs an account to browse the site or send an enquiry. Every enquiry form (the Contact page and the homepage catalogue-download banner) requires a phone number and email address, and this is enforced twice: the form won't submit without both, and the `/api/enquiry` route itself rejects any request missing a name, company, email or phone — so there's no way for an enquiry to land in the admin inbox without contact details, even if someone bypasses the form.

## 1. Get a database (2 minutes)

You need a free Postgres database — either works, pick one:

- **Vercel Postgres**: in your Vercel dashboard → Storage → Create Database → Postgres. It gives you a connection string.
- **Neon** (neon.tech): sign up, create a project, copy the connection string it shows you.

## 2. Local setup

```bash
npm install
cp .env.example .env
```

Open `.env` and paste in:
- `DATABASE_URL` — the Postgres connection string from step 1
- `NEXTAUTH_SECRET` — run `openssl rand -base64 32` and paste the output (any random 32+ character string works)
- Leave the `TWILIO_*` lines blank for now — the site works fine without them; enquiries just won't send a WhatsApp alert yet (see section 4)

Then:

```bash
npx prisma db push     # creates all the tables in your database
npx prisma db seed     # adds demo accounts + the 6 sample products
npm run dev
```

Visit `http://localhost:3000`. Demo logins (from the seed script):

| Role | Email | Password |
|---|---|---|
| Admin | `admin@gaitz.co` | `ChangeMe!Admin1` |
| Distributor | `dealer@example.com` | `ChangeMe!Dealer1` |

**Change both passwords** (or delete these accounts and create real ones) before giving anyone real access — see "Managing accounts" below.

## 3. Deploying to GitHub + Vercel

1. Push this folder to a new GitHub repo (same as before — create the repo on github.com, then either drag-and-drop upload the files or use `git init && git add . && git commit -m "Gaitz dynamic site" && git push`).
2. In Vercel: **Add New → Project**, import the repo. Vercel auto-detects Next.js — no config needed.
3. Before clicking Deploy, open **Environment Variables** and add the same ones from your `.env` file: `DATABASE_URL`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL` (set this to your real domain once you have one, e.g. `https://gaitz.co` — until then your `*.vercel.app` URL is fine), and the `TWILIO_*` ones once you've set those up.
4. Deploy. On first deploy, run the database setup once against your production database:
   ```bash
   # from your local machine, pointed at the production DATABASE_URL
   npx prisma db push
   npx prisma db seed
   ```
   (Or open a Vercel CLI shell / use Vercel's "Run Command" if you'd rather not do this from your laptop.)

Every push to `main` redeploys automatically from then on.

## 4. Connecting image storage (for real photos)

This is what powers Admin → Site content and the "Photos" tab on each product — real uploads, not placeholder boxes.

1. In your Vercel project → **Storage** tab → **Create Database** → **Blob**. Connect it to this project.
2. Vercel automatically adds `BLOB_READ_WRITE_TOKEN` to your production environment variables — nothing to copy for the deployed site.
3. For **local dev**, pull that token down: `npx vercel link` (once, to connect this folder to the Vercel project), then `npx vercel env pull .env.local`. That writes `BLOB_READ_WRITE_TOKEN` into `.env.local`, which Next.js reads automatically alongside `.env`.
4. Log in as admin and go to **Admin → Site content** to upload the homepage hero, plant photos, and series tiles, or **Admin → Products → [a product] → Photos** to add product images. The public site picks them up immediately — no redeploy needed.

Until this is connected, uploads show a clear "image storage isn't connected yet" message instead of failing silently, and the public site keeps showing the grey placeholder boxes.

## 5. Connecting WhatsApp (Twilio)

1. Create a free account at [twilio.com](https://www.twilio.com).
2. In the Console, activate the **WhatsApp Sandbox** (Messaging → Try it out → Send a WhatsApp message). It gives you a sandbox number and a join code — send that code from your team's WhatsApp to the sandbox number once, to opt in.
3. Copy your **Account SID** and **Auth Token** from the Console dashboard.
4. Set these in `.env` (locally) and in Vercel's Environment Variables (production):
   ```
   TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   TWILIO_AUTH_TOKEN=your_auth_token
   TWILIO_WHATSAPP_FROM=whatsapp:+14155238886   # the sandbox number Twilio gives you
   TEAM_WHATSAPP_TO=whatsapp:+91XXXXXXXXXX      # the number that should receive alerts
   ```
5. Redeploy (or restart `npm run dev` locally). Submit the Contact form — your team's WhatsApp should get a message within seconds.

The sandbox is free and fine for testing, but every recipient has to send that join code once, and Twilio occasionally reminds sandbox users to re-join. For production, apply for a real **WhatsApp Business Sender** through Twilio (a short approval process, a few days) — no code changes needed on this end, you just swap `TWILIO_WHATSAPP_FROM` to the approved number.

## 6. Managing accounts

There's no self-serve "sign up" page on purpose — distributor accounts are something you create, not something anyone can request. For now, the fastest way to add or change one is directly in the database (Prisma Studio gives you a simple UI for this):

```bash
npx prisma studio
```

This opens a browser-based table editor. Open the `User` table to add a distributor (set `role` to `DISTRIBUTOR`, fill in `companyName`) or change a password. Passwords are stored hashed — to set one, generate a hash first:

```bash
node -e "console.log(require('bcryptjs').hashSync('the-new-password', 10))"
```
and paste the result into `passwordHash`.

(A proper "invite a distributor" admin screen is a natural next build step — see Roadmap.)

## Before you go live

- **Replace the demo products** — `prisma/seed-data.js` has 6 sample SKUs with made-up specs and prices. Swap these for your real Product Master data, or add products directly via `npx prisma studio` / the admin Products tab.
- **Upload real photos** — connect image storage (section 4 above), then use Admin → Site content and Admin → Products → Photos. Anything not uploaded yet still shows a grey placeholder box, so it's always obvious what's left to add.
- **Capability numbers, founding year, manufacturing location** — left blank on the About/Home pages because the source material had conflicts (2003 vs 2021, Kanpur vs Agra). Fill these in once confirmed.
- **Contact details** — phone, email and address are still placeholders in `app/contact/page.js`, `app/about/page.js`, and `components/SiteFooter.js`.
- **Change the two demo passwords**, or delete those accounts.

## Roadmap — natural next steps

Since you mentioned wanting to keep building this out over the coming weeks, in roughly the order I'd tackle them:

1. **"Invite a distributor" flow** in admin, instead of editing the database directly.
2. **Order confirmation emails/WhatsApp to the distributor**, not just the internal alert.
3. **Full WhatsApp two-way** (customers messaging in, not just alerts out) — this is a bigger step up (Meta Cloud API, webhook handling, conversation state) and worth scoping separately once the alert flow above is proven out.
4. **Real production WhatsApp sender** (out of the Twilio sandbox).
5. **Drag-to-reorder product photos** and a proper "add new product" form in admin (right now new SKUs go in via `prisma/seed-data.js` or Prisma Studio).

Happy to build any of these next — just say which one.
