# Calicut Rides

**Calicut Rides** is a full-stack bike and scooter rental web app for **Kozhikode (Calicut), Kerala**. Visitors and locals can browse rental bikes by pickup area and type, book them by the hour or by the day, and manage their bookings. The owner runs the business from a protected panel to manage the fleet and see every booking.

**Live demo:** https://<your-username>.github.io/calicut-rides/ (browser-only version, bookings are saved on the visitor's device)

---

## Why this project
Tourists and students in Kozhikode often rent two-wheelers through phone calls and paper registers. Calicut Rides replaces that with a simple online system: clear prices, real pickup points around the city, automatic price calculation, and no double bookings.

## Features

### For renters
- Browse bikes with engine size (cc), type, description, pickup address, hourly and daily price, and refundable deposit
- Filter by pickup area (Beach Road, Mananchira, Mavoor Road, Palayam, Kallai, Medical College, Feroke) and by type (Scooter, Commuter, Cruiser, Sports)
- Book with name, Indian mobile number, driving licence number, pickup date and time, and duration from 1 hour to 7 days
- See the rental total before confirming. The day rate applies automatically for long rentals
- Look up bookings by phone number and cancel up to 1 hour before pickup
- Works on phones and desktops, with light and dark mode that follows the device setting

### For the owner
- Sign in with a secret access token
- Add bikes with full details, remove bikes, and mark them available or unavailable
- See all bookings with renter name, phone and licence number
- Mark a booking as completed when the bike is returned

---

## Architecture

```
 Browser (HTML, CSS, JavaScript)
        |  fetch() JSON over HTTP
        v
 Express server (server.js)
   - validates input
   - calculates price
   - checks bike availability
        |  SQL
        v
 SQLite database (db.js, calicut-rides.db)
   - bikes
   - bookings
```

## Front end
Located in `public/index.html`. It is a single-page app written in plain HTML, CSS and JavaScript, with no framework or build step.

- **Views:** Bikes, My bookings, and Owner, switched with accessible tab buttons
- **Booking form:** a native `<dialog>` modal with live price calculation
- **Data:** all data comes from the backend through a small `api()` helper around `fetch`
- **Design:** sea-teal and sun-yellow palette inspired by Kozhikode beach, Bricolage Grotesque and Figtree fonts, responsive grid, visible keyboard focus, automatic dark mode
- **Safety:** all text from the server is escaped before it is shown, to prevent script injection
- **Owner session:** the token is kept in `sessionStorage` and cleared when the tab closes

The `docs/index.html` file is a standalone browser-only version of the app that stores data in `localStorage`. It exists so the project can be demoed on GitHub Pages without a server.

## Back end
Located in `server.js` and `db.js`. It is a REST API built with **Node.js** and **Express**, backed by **SQLite** through `better-sqlite3`.

- **Validation:** names, Indian mobile numbers (`+91` optional, starting with 6 to 9), licence numbers, pickup time in the future, duration from 1 to 168 hours
- **Pricing:** calculated on the server, never trusted from the browser. Under 24 hours the cost is hours times the hourly rate, capped at the day rate. From 24 hours it is the number of days (rounded up) times the day rate
- **No double bookings:** an overlap check and the insert run inside one database transaction, so two people cannot book the same bike for the same time
- **Cancellation rule:** only the phone number that made the booking can cancel, and only up to 1 hour before pickup
- **Owner security:** owner routes need an `x-admin-token` header. The token is compared in constant time and owner access stays disabled if no token is set
- **Database safety:** every query uses prepared statements, so SQL injection is not possible through the API

## Database

**bikes**

| Column | Type | Notes |
|---|---|---|
| id | INTEGER | primary key |
| name | TEXT | e.g. Royal Enfield Classic 350 |
| type | TEXT | Scooter, Commuter, Cruiser, Sports |
| cc | INTEGER | engine size |
| area | TEXT | pickup area |
| address | TEXT | pickup address |
| price_hour | INTEGER | rupees per hour |
| price_day | INTEGER | rupees per day |
| deposit | INTEGER | refundable deposit in rupees |
| description | TEXT | short description |
| available | INTEGER | 1 or 0 |

**bookings**

| Column | Type | Notes |
|---|---|---|
| id | INTEGER | primary key |
| bike_id | INTEGER | references bikes, deleted with the bike |
| name, phone, licence_no | TEXT | renter details |
| start_ts | INTEGER | pickup time in milliseconds |
| hours | INTEGER | rental duration |
| total | INTEGER | price calculated by the server |
| status | TEXT | active, completed or cancelled |
| created_at | INTEGER | booking time |

## API reference

| Method | Route | Description |
|---|---|---|
| GET | `/api/bikes?area=&type=` | List bikes, optionally filtered |
| GET | `/api/areas` | List pickup areas |
| GET | `/api/bikes/:id/busy` | Upcoming booked time slots for a bike |
| POST | `/api/bookings` | Create a booking |
| GET | `/api/bookings?phone=` | Find bookings by phone number |
| POST | `/api/bookings/:id/cancel` | Cancel a booking (body: `phone`) |
| GET | `/api/admin/check` | Verify the owner token |
| GET | `/api/admin/bookings` | List all bookings |
| POST | `/api/admin/bikes` | Add a bike |
| PATCH | `/api/admin/bikes/:id` | Set a bike available or unavailable |
| DELETE | `/api/admin/bikes/:id` | Remove a bike |
| POST | `/api/admin/bookings/:id/status` | Set booking status |

Example booking request:

```json
POST /api/bookings
{
  "bike_id": 5,
  "name": "Anu",
  "phone": "9876543210",
  "licence_no": "KL11 20230012345",
  "start_ts": 1767254400000,
  "hours": 6
}
```

## Project structure

```
calicut-rides/
  public/index.html   front end served by Express
  docs/index.html     browser-only demo for GitHub Pages
  server.js           Express API and business rules
  db.js               SQLite schema and sample data
  package.json        dependencies and scripts
  .env.example        environment variable template
  README.md
  LICENSE
```

## Run locally
Requires Node.js 18 or newer.

```bash
npm install
cp .env.example .env     # set ADMIN_TOKEN to a long random string
npm start
```
Open http://localhost:3000. The database and 8 sample bikes are created on first run. To open the Owner tab, enter the `ADMIN_TOKEN` you set.

## Deploy
Any Node.js host with a persistent disk works, such as Render, Railway or a VPS.
1. Connect the GitHub repo
2. Build command: `npm install`. Start command: `npm start`
3. Set the `ADMIN_TOKEN` and `PORT` environment variables
4. Attach a persistent disk for the `.db` file, otherwise data resets on each deploy

For the static demo, turn on GitHub Pages in the repo settings and choose the `main` branch and the `/docs` folder.

## Known limitations
- Booking lookup uses only the phone number, so OTP verification is needed before real use
- No online payment yet, and the deposit is collected at pickup
- No licence photo upload or identity verification
- No rate limiting, so run it behind HTTPS and a reverse proxy in production

## Roadmap
- UPI and Razorpay payments
- OTP login for renters
- Bike photos and a gallery
- Google Maps pickup links
- SMS and WhatsApp booking confirmations
- Malayalam language option
- Revenue and utilisation reports for the owner

## Tech stack
Node.js, Express, SQLite (better-sqlite3), HTML, CSS, vanilla JavaScript.

## Author
Built by Sreyas as a portfolio project.

## License
MIT
