# Shenanigan's Snug website (preview)

Website for Shenanigan's Snug Irish Pub & Grill in Cave Creek, Arizona, with a preview of the staff dashboard.

## What's here

- `index.html` holds the whole site: Home, Menu, Order online (coming soon), Events, Reservations, Private parties, Our story, Gallery, Visit and the Staff dashboard.
- `img/` holds the logo and photos (the photos are from Mountain View Pub).
- `vendor/` holds jsPDF (makes the printable PDF calendar) and pdf.js (turns uploaded PDF flyers into images).
- `AGENTS.md` is a detailed guide to the code for AI coding agents and developers.

## Deploying

There is no build step. On Vercel choose Framework Preset "Other" and leave the build command empty. Every push to `main` redeploys the site.

To try it locally, run `python3 -m http.server 8000` in this folder and open `http://localhost:8000`.

## Staff dashboard

Open it from the "Staff sign-in" link in the footer. It has three tabs:

- **Overview:** today's orders and requests, the announcement bar, and a sold-out list.
- **Calendar:** add, edit and delete events, rename or recolor event types, and upload event flyers (photo or PDF) that flip to show their details on the Events page.
- **Menu:** add, edit and delete dishes with prices, sizes, descriptions and photos, and mark dishes sold out.

## Not live yet

- **Staff changes are saved only in the browser where they were made.** Menu, calendar, event type and flyer changes stay after a reload on that device, but other visitors don't see them. The announcement bar and the Overview counts reset on reload.
- **The staff dashboard has no login.** Anyone who opens it can make changes, but only in their own browser.
- **Orders, bookings, party requests and messages are not sent anywhere.** Online ordering shows a "coming soon" page; the working order page is hidden at `#order-preview`.
- **Hours, events, address and phone are examples** until the owners confirm them.

The next phase is a real backend so changes show for everyone: a Clerk staff login, a database, photo storage and Stripe payments. See `AGENTS.md` for details.
