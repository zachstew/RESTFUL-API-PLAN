# RESTful API Planning Document: Fantasy Football Draft Board

## 1. API Selection

**API chosen:** Ball Ranks API (https://ballranks.com/api/v1), listed in the
public-apis repository.

**Purpose:** Ball Ranks provides fantasy football rankings and projections. I
will use the football season rankings (PPR scoring) to build a draft board tool.

**Why I chose it:**
- I am interested in fantasy football.
- No API key is needed for basic use.
- It returns JSON with clear fields I can display and style.
- It is read-only (GET only), so my own Flask backend will add a "Draft Board"
  resource to cover Create, Update, and Delete.

**Example record I will use for testing (Ja'Marr Chase):**
```json
{
  "player": {
    "id": "00-0036900",
    "slug": "jamarr-chase",
    "name": "Ja'Marr Chase",
    "team": "CIN",
    "position": "WR"
  },
  "rank": 2,
  "positionRank": 1,
  "projectedPoints": 329
}
```

**REST principles in my design:**
- **Resource identification:** Each resource has its own URI, such as
  `/rankings/season` for the rankings and `/players/jamarr-chase/rankings` for
  one player. My own resource uses `/api/board/{id}`.
- **Statelessness:** Each request includes everything the server needs (URL,
  parameters, headers, and JSON body). The server does not remember anything
  about me between requests.
- **Standard HTTP methods:** GET reads data, POST creates, PUT/PATCH updates,
  and DELETE removes. I use each one only for its intended purpose.

## 2. Endpoint Mapping

My design follows the same REST principles: URIs identify resources, requests
are stateless, and standard HTTP methods and status codes are used. The public
API covers the Read part of CRUD. My Flask backend covers Create, Update, and
Delete with a Draft Board resource.

### Public API endpoints (Ball Ranks, GET only)

| CRUD | Verb | Endpoint | Request | Response |
|---|---|---|---|---|
| Read (list) | GET | `/rankings/season?sport=football&scoring_format=ppr&limit=100&offset=0` | Query parameters | `200 OK` with `{meta, total, rankings[]}` |
| Read (one) | GET | `/players/{player}/rankings` | Player slug in URL | `200 OK` with one player, or `404` |

**Data returned by the rankings endpoint:** `meta` (model and last updated
time), `total` (700 players), and `rankings`, an array where each item has
`player` (`id`, `slug`, `name`, `team`, `position`), `rank`, `positionRank`,
and `projectedPoints`.

### My Flask backend endpoints (contract only, not built yet)

| CRUD | Verb | Route | Request | Response |
|---|---|---|---|---|
| Read (list) | GET | `/api/rankings` | Optional `limit`, `offset` | `200 OK` with the rankings from Ball Ranks |
| Create | POST | `/api/board` | JSON body (see Section 4) | `201 Created` with the saved entry |
| Read (list) | GET | `/api/board` | None | `200 OK` with array of saved entries |
| Update | PUT / PATCH | `/api/board/<id>` | JSON body (full or partial) | `200 OK` with updated entry |
| Delete | DELETE | `/api/board/<id>` | ID in URL | `204 No Content` |

**Data accepted by my board endpoints:** a JSON body with `playerName`,
`position`, `status`, and `note` (see Section 4).

Flask will call Ball Ranks using the Python `requests` library and keep the
saved board entries on the server (for example, in a simple JSON file).

**Example for Ja'Marr Chase:** GET the rankings (he is rank 2), POST him to my
board with status "target", PATCH his status to "drafted", then DELETE him from
the board.

## 3. Data Flow Diagram (Textual)

**Flow: Ball Ranks API → Flask → browser → user**

1. **Data on the API side:** The response is a JSON object with `meta` (model
   and last updated time), `query`, `total` (700 players), and `rankings`
   (an array of player records like the Chase example in Section 1).
2. **Page load:** The page calls `loadRankings()` and shows a "Loading..."
   message.
3. **Fetch GET request:** The browser uses `fetch` with async/await and an
   `Accept: application/json` header:
   ```js
   try {
     const res = await fetch("/api/rankings?limit=100&offset=0", {
       method: "GET",
       headers: { "Accept": "application/json" }
     });
     if (!res.ok) throw new Error("HTTP " + res.status);
     const data = await res.json();
     renderPlayers(data.rankings);
   } catch (err) {
     showError("Could not load rankings. Please try again.");
   } finally {
     hideLoading();
   }
   ```
4. **Backend processing:** Flask calls Ball Ranks with `requests.get()`, checks
   the status code, flattens each record (moves the `player` fields up to the
   top level), and returns JSON to the browser. I will cache the result so I
   stay under the free daily limit. Flattened Chase record:
   `{id, slug, name: "Ja'Marr Chase", team: "CIN", position: "WR", rank: 2,
   positionRank: 1, projectedPoints: 329}`
5. **Data inside my app:** The JSON is stored in a JavaScript array in memory
   and used to build the table.
6. **Loading state:** "Loading..." shows while waiting for the response and is
   hidden in the `finally` block.
7. **Error handling:** The `try/catch` handles network failures and bad
   statuses (such as 429 when the daily limit is reached). The user sees a
   friendly message and a Retry button.
8. **Dynamic rendering:** Each record in the array becomes a table row showing
   rank, name, team, position, position rank, and projected points, with an
   "Add to Board" button. Saved board entries are shown in a second list.
9. **Conditional styling:** CSS classes are added based on data values:
   - `position` = WR/RB/QB/TE adds `pos-wr`, `pos-rb`, `pos-qb`, or `pos-te`
     (a different color for each).
   - `rank <= 12` adds `top-tier` (bold with a gold accent).
   - Board status: `target` is green, `avoid` is red, `drafted` is gray with
     a strikethrough.

   Example: Chase is a WR (`pos-wr`) and rank 2 (`top-tier`).

## 4. Form Handling

**Form: Add or Edit a Draft Board Entry**

| Field | Control | Validation |
|---|---|---|
| Player name | text input | Required, 2 to 60 characters |
| Position | dropdown (QB, RB, WR, TE) | Required, must be one of the four |
| Status | dropdown (target, avoid, drafted) | Required, default is "target" |
| Notes | text area | Optional, maximum 200 characters |

When the user clicks "Add to Board" on a ranking row, the player name and
position fill in automatically.

**Local validation:** When the user clicks Submit, JavaScript checks each field.
If something is invalid, an error message appears next to that field and no
request is sent.

**Converting form data to JSON:** JavaScript reads the form values, trims the
text, and uses `JSON.stringify` to build the payload. Example for Chase:
```json
{
  "playerName": "Ja'Marr Chase",
  "position": "WR",
  "status": "target",
  "note": "Take with an early pick"
}
```

**Backend contract:**
- `POST /api/board` with `Content-Type: application/json` returns `201 Created`
  with the saved entry.
- `PUT` or `PATCH /api/board/<id>` updates an entry and returns `200`. For
  example, `{"status": "drafted"}`.
- `DELETE /api/board/<id>` returns `204`.
- Invalid data returns `400` with an error message.

## 5. Technology Stack

- **Backend:** Python 3, Flask, and the `requests` library
- **Front end:** HTML, CSS, and JavaScript (Fetch API)
- **Testing:** Postman to test the endpoints, browser DevTools (Network tab),
  and trying the form with invalid input
- **Tools:** VS Code, Git and GitHub, curl

## 6. Implementation Steps

1. Choose the API, test it with curl, and look at the JSON it returns.
2. Set up the project folder, Flask app, and HTML, CSS, and JS files.
3. Install the tools with `pip install flask requests`.
4. Build the Flask route that gets the rankings from Ball Ranks.
5. Write the front-end `fetch` GET with loading and error messages.
6. Show the rankings in a table.
7. Add conditional styling (position colors, top tier, board status).
8. Build the form and the local validation.
9. Build the Create (POST) route and connect the form to it.
10. Add the Update (PUT/PATCH) and Delete buttons and routes.
11. Test everything in Postman and DevTools, then do a final review and
    finish the documentation.