# 🎉 Food Decider — "You've Won a Meal!"

An app for you and your friend to stop arguing about where to eat. Enter your
budget, headcount, preference, and any extra notes, and it spins up meal
"prizes" — a shop, an item, and the expected amount — pulled from places and
items you've added yourself.

## How it works

- **FastAPI** backend (`main.py`) serves both the API and the frontend.
- **SQLite** (`food_decider.db`, created automatically) is your fast local
  cache — the app always reads/writes here first, so it works even with no
  internet.
- **MongoDB Atlas** is the shared source of truth. Every write is pushed to
  Mongo immediately in the background; if that fails (no internet), it's
  queued and retried automatically. On startup, the app pulls the latest
  data from Mongo so you see what your friend added on their device.
- **Suggestion engine** (`suggest.py`) scores every place+item combo:
  filters out anything over budget, boosts matches to your preference and
  notes, and *penalizes* anything you ate in the last 30 days (heavily if
  within 2 days, lightly if within 30) so you don't get the same suggestion
  every day.

## Setup

1. Install dependencies:
   ```
   pip install -r requirements.txt
   ```

2. Set up MongoDB Atlas (free tier is fine):
   - Create a cluster at https://www.mongodb.com/cloud/atlas
   - Get your connection string (Connect → Drivers → Python)
   - Copy `.env.example` to `.env` and paste it into `MONGO_URI`
   - Set `DEVICE_OWNER` to your name (e.g. "arjun") — your friend should set
     it to their name on their own device so history shows who ate what.
   - **Important:** in Atlas, whitelist both your and your friend's IP
     addresses (or `0.0.0.0/0` for simplicity, since this is a small
     personal app) under Network Access.

3. Run the app:
   ```
   uvicorn main:app --host 0.0.0.0 --port 8000 --reload
   ```

4. Open `http://localhost:8000` in your browser. Your friend runs the same
   steps on their own device/laptop, pointing at the same MongoDB cluster.

If you skip step 2, the app still works perfectly — it just stays local to
your device only (the status pill in the bottom-right will show "offline").

## Using the app

- **🎰 Claim Prize** — enter budget, people, preference, notes, hit spin.
  Tap "Claim This Prize" on whichever suggestion you actually go with; that
  logs it to history so it won't be suggested again for a while.
- **🏪 Prize Inventory** — add/edit/delete places and their menu items
  (name, price, category like veg/non-veg, and free-text tags used for
  matching your notes).
- **📜 Winning Streak** — your last 30 days of claimed meals.

## API reference (if you want to script against it)

| Method | Path                          | Purpose                       |
|--------|-------------------------------|--------------------------------|
| GET    | /api/places                   | List places                    |
| POST   | /api/places                   | Add a place                    |
| PUT    | /api/places/{id}               | Update a place                 |
| DELETE | /api/places/{id}               | Soft-delete a place             |
| GET    | /api/places/{id}/items         | Items at a place                |
| POST   | /api/places/{id}/items         | Add an item to a place          |
| PUT    | /api/items/{id}                | Update an item                  |
| DELETE | /api/items/{id}                | Soft-delete an item             |
| GET    | /api/items                    | All items (with place name)     |
| POST   | /api/suggest                  | Get meal suggestions            |
| GET    | /api/history?days=30           | Meal history                    |
| POST   | /api/history                  | Log a meal                      |
| POST   | /api/sync/push                 | Force-push unsynced local rows  |
| POST   | /api/sync/pull                 | Force-pull latest from Mongo    |
| GET    | /api/status                   | Check Mongo connection status    |

## Notes & things you may want to tweak

- The suggestion engine assumes everyone orders the *same* item
  (`price × people`). If you two often order different things, you could
  extend `/api/suggest` to combine two items instead of one — happy to add
  that if you want it.
- Deletes are "soft deletes" (a `deleted` flag) so history entries still
  point to valid names even after you remove a place/item from the menu.
- For real day-to-day use across two separate laptops, you'll each run your
  own `uvicorn` server pointed at the same Mongo cluster — this isn't
  hosted anywhere by default. If you want, I can help you deploy it (e.g.
  Render, Railway, a small VPS) so you don't need to keep a laptop running.
