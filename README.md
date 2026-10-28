# Meetup Tab

**Order what you want. Pay for what you had.** · *Pide lo que quieras. Paga lo que consumiste.*

A one-file web app for [Riviera Maya Tech Builders](https://rivieramaya.tech) meetups held at venues that host for free (restaurants, bars, beach clubs), where each attendee pays for their own food and drinks. The venue runs **one bill for the table**. Each person keeps **their own running tab** on their phone, and the organizer checks the two against each other.

**Live:** https://jhon-ran.github.io/meetup-tab/

## Why
Venues in Mexico usually don't want to open 4–10 separate checks for one group. This app lets the venue run a single table bill and still:

1. lets **everyone know exactly what they owe**, including shared items and tip,
2. makes sure the **venue gets paid correctly**, because the organizer compares everyone's tabs with the venue's bill, and
3. gives **RMTB spend data per venue**, to show future venues what a meetup brings in.

No money passes through the app. People pay the venue directly, and the app only records who has paid.

## What it does

### For attendees (scan the QR)
- **Check in** with a name and an optional "What are you building?".
- **Browse the venue's menu**, with search, category chips, sizes and choices. Item names show in English or Spanish, and in English mode the printed Spanish name appears underneath so you can point at it for the waiter.
- **Add items to your tab**, with quantity and a note. You can **split an item** equally with other attendees, or add a **custom item** that isn't on the menu.
- **My tab** shows your running subtotal, a tip picker (0 / 10 / 15 / 20 %, default 10 %) and the total.
- **Show to waiter:** a large screen in Spanish listing what you owe.
- **I paid:** mark your tab as paid (cash, card or transfer).
- You can remove your own unpaid items, or switch to another person on a shared phone.
- The whole app switches between **EN and ES**.

### For the organizer (`#/admin`, PIN-protected)
- **Meetups:** create a meetup for a date and venue, and get a **QR code** (plus a big QR view) and a link to share.
- **Live dashboard:** totals for ordered, paid, outstanding and tips.
- **People:** see each attendee's tab, add **walk-ins without a phone**, order for anyone, mark people as paid, undo payments, and delete items.
- **Check against the venue's bill:** enter the venue's total and the app shows whether it balances, how much on the bill **nobody has claimed**, or how much the tabs are **over the bill** (likely duplicates).
- **Venue report:** spend by category, top items and check-ins by hour. You can **copy a summary in ES or EN** for the venue manager or **download a CSV**.
- **Close or reopen** a meetup. Once closed, attendees can still mark items as paid but can't add new ones.
- **Venues:** a menu editor (categories, items, sizes, choices, notes) and JSON import/export. For a new venue, **Copy AI prompt** gives you a prompt to paste into Claude with photos of the menu; paste the JSON it returns back in.
- **Insights:** spend, attendees and average per person across meetups, grouped by venue.

The Día & Noche Beach Club (Puerto Morelos) menu is pre-loaded the first time the organizer page opens.

## Specs

| | |
|---|---|
| **App** | One static `index.html`: vanilla JS, ES modules, no build step |
| **Backend** | Firebase Firestore (live sync) + Firebase Anonymous Auth, Spark (free) plan |
| **Libraries (CDN)** | Firebase JS SDK 10.12.2 (gstatic), qrcodejs 1.0.0 (cdnjs), Google Fonts: Bungee (display) + Archivo (text) |
| **Hosting** | GitHub Pages (any static host works) |
| **Routing** | Hash router: `#/` home · `#/e/{eventId}` attendee · `#/admin` organizer · `#/admin/e/{eventId}` event control · `#/admin/venue/{venueId}` menu editor |
| **Layout** | Mobile-first, max width 560 px, light theme only (it's read outdoors in sunlight) |
| **Look** | Caribbean turquoise, rosa mexicano, sun yellow and navy; papel picado header, sign-painted headings, and "My tab" styled as a comanda (order ticket) |
| **Currency** | MXN. Menu prices already include IVA, so no tax is added. Tip is a percentage of each person's own subtotal |
| **Demo mode** | If `CONFIG.firebase.apiKey` starts with `PASTE`, the app stores data in `localStorage` on one device and shows a banner |

### Data model (Firestore)
```
venues/{venueId}                 name, location, menu, createdAt, updatedAt
events/{eventId}                 title, date, venueId, venueName, venueLocation, menu (snapshot),
                                 status (open|closed), tipDefault, venueBill, summary, createdAt, closedAt
events/{id}/attendees/{attId}    name, building, joinedAt, by (self|organizer)
events/{id}/orders/{orderId}     item + variant/option names (es/en), category, unitPrice, qty, note,
                                 shares [attIds], paid {attId: timestamp|null}, createdBy, byOrganizer, createdAt
events/{id}/payments/{pid}       attendeeId, name, subtotal, tipPct, tip, total, method, at, orderIds, by
```
Each person's share of an order is `unitPrice × qty ÷ shares.length`.

Menu JSON shape:
```json
{"categories":[{"es":"Cerveza","en":"Beer","items":[
  {"es":"Sol","en":"Sol","price":65},
  {"es":"Michelada","en":"Michelada","variants":[{"es":"½ litro","en":"½ liter","price":105},{"es":"1 litro","en":"1 liter","price":135}]},
  {"es":"Alitas (8 piezas)","en":"Wings (8 pcs)","price":190,"options":["BBQ","Buffalo"]}
]}]}
```
Use `variants` when sizes have different prices, and `options` for choices at the same price.

## Setup
1. **Firebase:** create a project, then a Firestore database (production mode, `nam5`), and enable **Authentication → Anonymous**.
2. **Firestore rules:**
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /{document=**} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```
3. **Configure:** register a web app, paste its config into `CONFIG.firebase` at the top of the `<script type="module">` in `index.html`, and set `CONFIG.organizerPin`.
4. **Host:** push `index.html` to GitHub Pages (or any static host), then add the host's domain under **Authentication → Settings → Authorized domains**.

## Meetup-night run-sheet
1. Open `/#/admin`, enter the PIN, then **+ New meetup**, pick the date and venue, and **Create**.
2. Show the **Big QR** on the table, or print it.
3. Add walk-ins with **+ Person** and order for them.
4. When the venue brings the table's bill, enter its total in **Check against the venue's bill**.
5. **Copy summary (ES)** and send it to the venue manager, then **Close meetup**.

## Known limits (proof of concept)
- The organizer PIN is checked in the browser and can be read in the page source. It keeps casual users out, but it isn't real security.
- Anyone with the link can tap an existing name and act as that person. That's fine for a trusted community.
- The Firestore rules let any signed-in anonymous user read and write everything.
- Tabs are self-reported. The bill check is the safety net.
- There's no "delete meetup" button yet. To delete one, use the Firestore console.

---
Riviera Maya Tech Builders · Built in Paradise 🌴
