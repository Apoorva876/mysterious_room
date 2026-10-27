
# AI Mystery Room — Project Specification

**Document:** `SPEC.md`  
**Version:** 1.0  
**Status:** Implementation blueprint  
**Project type:** Full-stack AI-powered mystery investigation game

---

## 1. Project Overview

### 1.1 Summary

AI Mystery Room is a web application where a player creates a mystery from a short premise and investigates an AI-generated case. The game contains a story, suspects, discoverable clues, optional hints, and a hidden solution. The player must accuse one suspect. The backend evaluates the accusation, calculates the score, and reveals the culprit and explanation.

### 1.2 Problem Statement

Mystery games can be repetitive or limited to prewritten cases. AI Mystery Room uses Gemini to create varied cases from player-provided premises and offers an interactive investigation experience while preserving a fair, logically consistent solution.

### 1.3 Goals

- Generate playable mysteries at Easy, Medium, and Hard difficulty.
- Let players inspect suspects, discover clues, and request hints.
- Keep the solution private until the player submits an accusation.
- Calculate scores on the server, not in the browser.
- Persist case state in MongoDB.
- Deliver a responsive, accessible interface that works in production.

### 1.4 Target Users

- People who enjoy detective stories, puzzles, and escape-room-style games.
- Students and casual players seeking short interactive games.
- Users who want to explore mysteries based on their own story premises.

### 1.5 Out of Scope for Version 1

- Multiplayer or real-time collaborative investigations.
- User accounts, social features, leaderboards, and payments.
- User-created public case sharing.
- Persistent cross-device profiles.
- Voice acting, animations requiring a game engine, or generated images.

The app should support anonymous play. Do not collect unnecessary personal information.

---

## 2. Technology Stack

### Frontend
- React
- Vite
- Tailwind CSS
- React Router
- Axios

### Backend
- Node.js
- Express
- Mongoose
- `@google/genai` for Gemini
- Zod for request and generated-data validation
- `cors`, `dotenv`
- `express-rate-limit` for basic rate limiting

### Database and Hosting
- MongoDB Atlas
- Frontend: Vercel
- Backend: a Node.js-compatible host such as Render

Use compatible, currently supported package versions. Do not hard-code API keys, database credentials, or production URLs in source code.

---

## 3. Core User Experience and Flow

1. The player opens the home page.
2. The player chooses **Create Mystery**.
3. The player enters a premise and chooses Easy, Medium, or Hard.
4. The frontend sends the premise and difficulty to the backend.
5. The backend validates the request, asks Gemini to generate a mystery, validates the response, and stores the case.
6. The player enters the Investigation Room.
7. The player reads the case story and examines the suspect cards.
8. The player discovers clues one at a time. Each discovered clue becomes visible and contributes to the score.
9. The player can request hints, subject to the difficulty's hint limit. Each used hint reduces the score.
10. The player selects a suspect and submits an accusation.
11. The backend locks the case, evaluates the accusation, calculates the final score, and returns the result.
12. The Results page shows whether the accusation was correct, the culprit, the solution explanation, and the score.
13. The player can start a new mystery.

### UX Rules
- Show clear loading feedback while a case is generated.
- Disable duplicate submissions while an operation is in progress.
- Preserve case progress after a page refresh by retrieving the case from the backend.
- Provide clear empty, loading, success, and error states.
- Confirm before submitting an accusation because it ends the investigation.
- After a case is completed, do not allow further clue discoveries or hint requests.
- If a case ID is invalid or expired/deleted, show a friendly message and offer to return home.

---

## 4. Pages and UI Requirements

### 4.1 Home (`/`)
Purpose: Explain the game and invite the player to begin.

Required elements:
- App title: **AI Mystery Room**
- Short description of the experience.
- Primary **Create a Mystery** button.
- Brief steps explaining how to play.
- Difficulty overview.
- Responsive layout for mobile, tablet, and desktop.

### 4.2 Create Mystery (`/create`)
Required elements:
- Text area for the premise.
- Premise guidance/example.
- Difficulty selector: Easy, Medium, Hard.
- Character count and input validation.
- Generate button.
- Loading state while Gemini generates the case.
- Recoverable error message and retry option.

Validation:
- Trim whitespace.
- Require a premise of 10–500 characters.
- Reject empty or excessively long input.
- Accept plain text only; do not render premise text as HTML.

### 4.3 Investigation Room (`/case/:id`)
Required elements:
- Case title, story, and difficulty badge.
- Progress indicator for clues discovered.
- Suspect cards showing only public suspect information.
- Clue area showing discovered clues and controls to discover available clues.
- Hint panel and remaining-hints indicator.
- Current score estimate, if displayed, clearly based on server-returned state.
- **Make an Accusation** button.
- Loading, empty, and error states.

Do not include the culprit ID, solution explanation, hidden clue text, or internal scoring answer in frontend bootstrap data, HTML, logs, or API responses before the accusation.

### 4.4 Accusation (`/case/:id/accuse`)
Required elements:
- Suspect selection using accessible radio controls or selectable cards.
- Confirmation dialog summarizing the selected suspect.
- Submit button.
- Validation when no suspect is selected.
- Prevent duplicate submissions.
- If the case is already completed, redirect to Results.

### 4.5 Results (`/case/:id/result`)
Required elements:
- Verdict: correct or incorrect accusation.
- Final score.
- Selected suspect.
- Correct culprit.
- Explanation of how the clues support the solution.
- Optional recap of discovered clues and used hints.
- **Play Again** button returning to Create Mystery or Home.

The result endpoint may reveal the solution only after the case has been completed.

### 4.6 Shared UI
- Header/navigation with a home link.
- Consistent buttons, cards, badges, dialogs, and form styles.
- Responsive design with keyboard-accessible controls.
- Visible focus states, sufficient contrast, and meaningful labels.
- Avoid relying on color alone to communicate correct/incorrect outcomes.

---

## 5. Difficulty Configuration

Use this single configuration in the backend as the source of truth.

| Difficulty | Suspects | Total clues | Available hints | Correct-accusation base score |
|---|---:|---:|---:|---:|
| Easy | 3 | 5 | 2 | 100 |
| Medium | 4 | 7 | 3 | 150 |
| Hard | 5 | 9 | 3 | 200 |

The generator must produce exactly the configured number of suspects, clues, and hints for the chosen difficulty. Reject or regenerate invalid output rather than silently changing the difficulty.

---

## 6. AI Generation Requirements

### 6.1 Gemini Responsibilities

Gemini generates:
- A case title and opening story.
- A fixed number of suspects with names, descriptions, motives, and alibis.
- A fixed number of clues.
- Hints appropriate to the difficulty.
- The culprit identifier.
- A solution explanation and clue-to-solution reasoning.

Gemini is a content generator, not the authority for scoring, permissions, or game state. Those rules belong to the backend.

### 6.2 Required Internal Data Shape

The backend should validate a normalized object equivalent to:

```json
{
  "title": "The Clockmaker's Last Hour",
  "story": "A concise, self-contained mystery setup.",
  "suspects": [
    {
      "id": "suspect_1",
      "name": "Mira Vale",
      "description": "The victim's business partner.",
      "motive": "A dispute over a missing ledger.",
      "alibi": "Claims to have been in the library."
    }
  ],
  "clues": [
    {
      "id": "clue_1",
      "title": "The Stopped Clock",
      "text": "The clock stopped later than the witness claimed.",
      "relevance": "Challenges one suspect's timeline."
    }
  ],
  "hints": [
    "Compare each alibi with the timeline.",
    "Pay attention to the clock and the witness statement."
  ],
  "solution": {
    "culpritId": "suspect_1",
    "explanation": "The timeline and physical evidence contradict the culprit's alibi.",
    "clueReasoning": [
      {
        "clueId": "clue_1",
        "reason": "The clock's time contradicts the stated timeline."
      }
    ]
  }
}
```

This is an illustrative schema. Real output must contain the exact configured counts. Do not copy the sample culprit or solution into generated cases.

### 6.3 Generation Prompt Requirements

The backend prompt must instruct Gemini to:
- Return valid JSON only, following the specified schema.
- Use the exact number of suspects, clues, and hints for the requested difficulty.
- Give every suspect a unique ID and a plausible motive/alibi.
- Use unique IDs for clues and ensure every clue references a valid suspect or meaningfully contributes to the timeline.
- Create one unambiguous culprit.
- Ensure the solution is logically deducible from the story and discoverable clues.
- Avoid contradictions between the story, suspect statements, clues, hints, and solution.
- Avoid making the culprit obvious from a name, description, or formatting.
- Make hints progressive and helpful without directly naming the culprit too early.
- Use family-friendly, fictional content and avoid graphic detail.
- Never include instructions to execute code or reveal secrets.

### 6.4 Validation and Retry

Use Zod (or an equivalent strict schema) to validate all generated content before saving:
- Required strings are non-empty and within configured length limits.
- IDs are unique.
- Suspect, clue, and hint counts match the selected difficulty.
- `solution.culpritId` matches an existing suspect.
- All solution clue references match existing clues.
- All arrays and nested objects follow the schema.
- Content does not contain unexpected fields that could be accidentally exposed.

If validation fails, retry generation a limited number of times (for example, up to two retries). If all attempts fail, return a safe `502` error and do not save the invalid case. Log diagnostic details on the server without logging API keys or exposing the hidden solution to the client.

Schema validation cannot prove that a mystery is logically consistent. Add a separate review/check step in the generation prompt and test representative generated cases manually. If a case fails logical checks, regenerate it instead of publishing it.

---

## 7. Security and Hidden-Solution Design

**Critical requirement:** The browser must not receive the culprit ID, solution explanation, hidden clue text, or other answer-bearing fields before the accusation has been submitted.

Implementation rules:
1. Keep the Gemini API key and MongoDB URI in backend environment variables only.
2. Store the solution in the database, but exclude it from normal case responses using explicit response DTOs or projections.
3. Never serialize and return the full Mongoose document to the client.
4. Return only public suspect fields. Motives and alibis are part of the public game content; the solution is not.
5. Store clues as hidden server-side data until discovered. The client may receive clue IDs/titles or availability information, but not undiscovered clue text or answer-bearing reasoning.
6. The discover endpoint returns the text of only the requested clue and records discovery server-side.
7. The hint endpoint returns only the next permitted hint and records hint usage server-side.
8. The accusation endpoint evaluates the suspect ID on the server and atomically marks the case complete.
9. The result endpoint returns solution details only if the case is complete.
10. Never trust a score, discovered-clue count, hint count, culprit ID, or completion status supplied by the browser.
11. Use server-side input validation, request-size limits, rate limiting, and safe error messages.
12. Configure CORS to allow the known frontend origin(s), not every origin in production.
13. Do not put secrets in Vite `VITE_*` variables; frontend environment variables are public.
14. Do not log API keys, database credentials, or full hidden solution objects in production logs.
15. Consider that an anonymous case ID is not strong authentication. Use unguessable IDs and rate limiting; add authentication only if the product later needs stronger ownership guarantees.

### Response DTO Rule
Create explicit serializers such as `toPublicCaseDTO(caseDoc)` and `toResultDTO(caseDoc)`. Do not use `res.json(caseDoc)` for raw database documents.

---

## 8. Database Design

Use a Mongoose model named `Case`.

Suggested document:

```js
{
  title: String,
  story: String,
  premise: String,
  difficulty: "easy" | "medium" | "hard",

  suspects: [{
    id: String,
    name: String,
    description: String,
    motive: String,
    alibi: String
  }],

  clues: [{
    id: String,
    title: String,
    text: String,
    relevance: String
  }],

  hints: [String],

  solution: {
    culpritId: String,
    explanation: String,
    clueReasoning: [{
      clueId: String,
      reason: String
    }]
  },

  discoveredClueIds: [String],
  usedHintCount: Number,
  accusation: {
    suspectId: String,
    isCorrect: Boolean,
    submittedAt: Date
  },

  score: Number,
  status: "active" | "completed",
  createdAt: Date,
  updatedAt: Date,
  completedAt: Date
}
```

### Database Rules
- Use schema validation, enums, timestamps, and sensible string/array limits.
- Initialize `discoveredClueIds` as an empty array, `usedHintCount` and `score` as `0`, and `status` as `active`.
- Use a random, unguessable public case ID. Do not expose MongoDB internal identifiers if not needed.
- Never store a client-provided score or answer as authoritative state.
- Keep solution fields in the same protected server-side record or a separate private subdocument; either approach must ensure they are never returned prematurely.
- Avoid persisting raw Gemini request/response logs unless necessary for debugging and safe to retain.
- Add a cleanup policy for old abandoned cases if hosting/storage limits require it.

---

## 9. API Contract

Base path: `/api`

All JSON errors should use a consistent structure:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Please enter a premise between 10 and 500 characters."
  }
}
```

Do not return stack traces, secrets, or internal solution data in error responses.

### 9.1 `POST /api/cases` — Generate a case

Request:
```json
{
  "premise": "A valuable painting disappears during a locked-room dinner.",
  "difficulty": "medium"
}
```

Success: `201 Created`
```json
{
  "case": {
    "id": "public_case_id",
    "title": "The Vanishing Canvas",
    "story": "The public opening story.",
    "difficulty": "medium",
    "suspects": [
      {
        "id": "suspect_1",
        "name": "Mira Vale",
        "description": "The gallery manager.",
        "motive": "A dispute about the painting.",
        "alibi": "Claims to have been in the office."
      }
    ],
    "clueCount": 7,
    "discoveredClueIds": [],
    "usedHintCount": 0,
    "hintsAvailable": 3,
    "score": 0,
    "status": "active"
  }
}
```

The response above illustrates the shape only; the actual suspect count must match the difficulty. Do not return the `solution`, hidden clue text, or unrequested internal fields.

Errors:
- `400 VALIDATION_ERROR` for invalid premise or difficulty.
- `429 RATE_LIMITED` for excessive generation requests.
- `502 AI_GENERATION_FAILED` if Gemini fails or returns invalid output after retries.
- `500 INTERNAL_ERROR` for unexpected failures.

### 9.2 `GET /api/cases/:id` — Retrieve active case

Returns public case details, suspect cards, discovered clue IDs, hint usage, score state, status, and clue metadata that does not disclose hidden text.

- `200 OK` if found.
- `404 CASE_NOT_FOUND` if unknown.
- Must not reveal the solution or undiscovered clue text.
- If completed, return a safe completed-case summary and let the frontend fetch the result.

### 9.3 `POST /api/cases/:id/clues/:clueId/discover` — Discover a clue

Request body: none.

Success:
```json
{
  "clue": {
    "id": "clue_1",
    "title": "The Broken Watch",
    "text": "The watch stopped at a time that conflicts with an alibi."
  },
  "progress": {
    "discoveredClueIds": ["clue_1"],
    "score": 5
  }
}
```

Rules:
- Verify the case exists and is active.
- Verify the clue ID belongs to that case.
- Return the clue text only after successful discovery.
- Add a clue to `discoveredClueIds` only once.
- Award +5 points only on the first discovery.
- Re-discovering a clue must not award duplicate points.
- Do not reveal `relevance` or `clueReasoning` if they contain hidden answer information.
- `404` for unknown case/clue; `409` if completed.

### 9.4 `POST /api/cases/:id/hints` — Request a hint

Request body: none.

Success:
```json
{
  "hint": {
    "index": 1,
    "text": "Compare the timeline with the alibis."
  },
  "progress": {
    "usedHintCount": 1,
    "hintsRemaining": 2,
    "score": 0
  }
}
```

Rules:
- Verify the case exists and is active.
- Return the next unused hint in order.
- Each hint can be used only once; each use deducts 10 points.
- Do not allow more hints than the configured difficulty limit.
- Score cannot fall below zero.
- `409 HINT_LIMIT_REACHED` when no hints remain; `409 CASE_COMPLETED` when completed.

### 9.5 `POST /api/cases/:id/accuse` — Submit an accusation

Request:
```json
{
  "suspectId": "suspect_2"
}
```

Success:
```json
{
  "result": {
    "isCorrect": false,
    "accusedSuspectId": "suspect_2",
    "score": 0,
    "status": "completed"
  }
}
```

Rules:
- Validate that the suspect belongs to the case.
- Accept only one final accusation per case.
- Calculate correctness by comparing the submitted suspect ID with the server-stored culprit ID.
- Base score is awarded only for a correct accusation.
- Formula: `max(0, (isCorrect ? baseScore[difficulty] : 0) + (discoveredClueCount * 5) - (usedHintCount * 10))`.
- A wrong accusation receives no base points but may retain points earned from clues minus hint penalties, with a minimum of zero. If the desired game design is instead to force a wrong-accusation score of zero, change this rule explicitly before implementation.
- Mark the case completed and store the accusation and final score atomically.
- Do not return the culprit or explanation from this endpoint; the frontend should navigate to the result page.
- `400` for invalid suspect ID; `404` for missing case; `409 CASE_COMPLETED` for a duplicate/finalized submission.

### 9.6 `GET /api/cases/:id/result` — Retrieve result

- `200 OK` only for a completed case.
- Return verdict, selected suspect, correct culprit, final score, explanation, and relevant clue reasoning.
- `409 CASE_NOT_COMPLETED` if the case is still active.
- `404 CASE_NOT_FOUND` if unknown.
- A completed case can be refreshed and its result retrieved again.

### 9.7 `GET /api/health` — Health check

Returns a simple status indicating that the server is running. Do not return secrets or detailed infrastructure information.

---

## 10. Scoring Rules

All scoring is calculated by the backend.

| Action | Points |
|---|---:|
| Correct accusation | Easy +100; Medium +150; Hard +200 |
| Each newly discovered clue | +5 |
| Each hint used | −10 |
| Minimum final score | 0 |

Formula:

```text
base = correct accusation ? difficulty base score : 0
finalScore = max(0, base + (unique clues discovered × 5) - (hints used × 10))
```

Additional requirements:
- Clue points are awarded once per unique clue.
- Hint penalties apply once per hint used.
- A wrong accusation earns no difficulty base score.
- The client may display server-returned score state but must not calculate the authoritative final score.
- The accusation endpoint must be idempotency-safe: a repeated request must never change a completed result or award extra points. Return a conflict or the existing safe status according to the implementation, without accepting a new accusation.

---

## 11. Frontend Architecture

Suggested folder structure:

```text
client/
  src/
    api/
      axios.js
      cases.js
    components/
      Header.jsx
      Button.jsx
      DifficultyBadge.jsx
      SuspectCard.jsx
      ClueCard.jsx
      HintPanel.jsx
      LoadingState.jsx
      ErrorMessage.jsx
      ConfirmAccusationDialog.jsx
    pages/
      HomePage.jsx
      CreateMysteryPage.jsx
      InvestigationPage.jsx
      AccusationPage.jsx
      ResultsPage.jsx
      NotFoundPage.jsx
    hooks/
      useCase.js
    utils/
      formatters.js
    App.jsx
    main.jsx
    index.css
  .env.example
  package.json
```

Use React Router for page navigation and Axios for API requests. Keep reusable components separate from pages. Store only public case data in frontend state. Do not place the Gemini API key or solution in frontend code, source maps, or public environment variables.

Suggested frontend environment variable:

```env
VITE_API_BASE_URL=http://localhost:5000/api
```

This variable is public and must contain no secret.

---

## 12. Backend Architecture

Suggested folder structure:

```text
server/
  src/
    config/
      db.js
      env.js
    models/
      Case.js
    routes/
      caseRoutes.js
    controllers/
      caseController.js
    services/
      geminiService.js
      scoringService.js
      caseService.js
    validators/
      caseSchemas.js
      generatedMysterySchema.js
    middleware/
      errorHandler.js
      rateLimiter.js
      notFound.js
    utils/
      publicCaseDTO.js
      resultDTO.js
    app.js
    server.js
  .env.example
  package.json
```

Responsibilities:
- **Routes:** Map URLs to controllers.
- **Controllers:** Parse validated input and return HTTP responses.
- **Services:** Handle AI generation, game rules, persistence, and scoring.
- **Model:** Define MongoDB schema and validation.
- **Validators:** Validate requests and generated AI data.
- **DTO serializers:** Explicitly control which fields can leave the backend.
- **Middleware:** Handle errors, rate limits, and unknown routes.

Keep business logic out of route handlers where practical.

---

## 13. Environment Variables

Backend `.env` (never commit this file):

```env
PORT=5000
NODE_ENV=development
MONGODB_URI=your_mongodb_connection_string
GEMINI_API_KEY=your_gemini_api_key
CLIENT_URL=http://localhost:5173
```

Frontend `client/.env`:

```env
VITE_API_BASE_URL=http://localhost:5000/api
```

Commit `.env.example` files with placeholder values only. Add `.env`, `client/.env`, and other secret files to `.gitignore`. If a secret is accidentally committed, revoke/rotate it; deleting the file in a later commit does not remove it from Git history.

---

## 14. Error Handling and Reliability

- Use one centralized Express error handler.
- Return consistent JSON errors with stable error codes.
- Never expose stack traces in production.
- Handle MongoDB connection failures and Gemini timeouts.
- Use request timeouts and limited retries for AI generation.
- Do not retry non-idempotent game actions blindly.
- Show useful frontend messages with a retry option where safe.
- Keep a loading indicator visible during generation and disable repeated clicks.
- Handle offline/network failures without losing the case ID.
- If the database is unavailable, fail safely rather than pretending an action was saved.
- Add a not-found route for unknown frontend paths.

Suggested status codes: `400` invalid input, `404` not found, `409` invalid state/conflict, `429` rate limit, `502` AI provider failure, and `500` unexpected server error.

---

## 15. Testing Plan

### Backend
- Valid and invalid premise/difficulty validation.
- Exact suspect, clue, and hint counts for all difficulties.
- Generated data with duplicate IDs or invalid culprit references is rejected.
- Gemini failures and malformed JSON are handled.
- Public case responses do not contain the solution or hidden clue text.
- Discovering a clue awards points once only.
- Hint limits and penalties are enforced.
- Score never falls below zero.
- Invalid suspect IDs are rejected.
- Accusation can only be finalized once.
- Result endpoint rejects active cases and reveals the solution only after completion.
- API errors never include secrets or stack traces.

### Frontend
- Home-to-create navigation works.
- Form validation works.
- Loading and error states are visible.
- Investigation progress refreshes correctly from the server.
- Clue and hint buttons update the UI from server responses.
- Accusation confirmation and results navigation work.
- Layout works on mobile and desktop.
- Keyboard navigation and accessible labels are usable.

### End-to-End
1. Generate an Easy case and verify 3 suspects, 5 clues, and 2 hints.
2. Generate a Medium case and verify 4 suspects, 7 clues, and 3 hints.
3. Generate a Hard case and verify 5 suspects, 9 clues, and 3 hints.
4. Discover clues and confirm duplicate discovery does not add points twice.
5. Use all hints and confirm the limit is enforced.
6. Submit a correct accusation in a controlled test and verify scoring.
7. Submit a wrong accusation in a controlled test and verify scoring.
8. Try to access the result before accusation; it must not reveal the solution.
9. Refresh an active case and verify progress persists.
10. Refresh a completed result and verify it remains available.
11. Repeat key tests against the deployed production website.

Use mocked Gemini responses for deterministic automated tests. Manually review a sample of live generated cases for narrative and logical quality.

---

## 16. Deployment Plan

### 16.1 MongoDB Atlas
- Create a database and a restricted database user.
- Configure network access appropriately for the backend host.
- Store the connection string only as a backend environment variable.
- Do not commit the connection string.

### 16.2 Backend
- Deploy the Node.js/Express server to a Node-compatible host such as Render.
- Set `MONGODB_URI`, `GEMINI_API_KEY`, `CLIENT_URL`, `NODE_ENV`, and any required port configuration in the hosting dashboard.
- Ensure the start command runs the production server.
- Add a health-check path such as `/api/health`.
- Confirm that the backend is reachable over HTTPS.

### 16.3 Frontend
- Deploy the Vite application to Vercel.
- Set `VITE_API_BASE_URL` to the deployed backend API base URL.
- Configure the framework/build settings for Vite.
- Add SPA rewrites if needed so refreshing a nested route works.
- Ensure production CORS configuration permits the deployed frontend origin.

### 16.4 Production Verification
- Test case generation, clue discovery, hints, accusation, and result retrieval on the live site.
- Test browser refresh on both active and completed cases.
- Verify no secret appears in frontend bundles, network responses, or public repository history.
- Verify error messages are user-friendly and the server logs are safe.
- Confirm the site works on mobile and desktop.

---

## 17. Implementation Milestones

Build incrementally; do not generate the entire app in one step.

### Milestone 1 — Specification and Skeleton
- Finalize this `SPEC.md`.
- Create the `client/` and `server/` folders.
- Configure scripts, environment examples, and `.gitignore`.
- Confirm Vite and Express start locally.

**Done when:** Frontend loads and backend health endpoint responds.

### Milestone 2 — Database and API Foundation
- Connect MongoDB.
- Create the Case model, request schemas, error handler, and route skeleton.
- Implement health endpoint and basic validation.

**Done when:** Server connects to the database and invalid requests return consistent errors.

### Milestone 3 — Gemini Generation
- Implement Gemini service and prompt.
- Validate AI JSON with Zod.
- Add bounded retries and safe failure handling.
- Save a valid case and return only the public DTO.

**Done when:** All three difficulty levels generate cases with exact counts and no solution leakage.

### Milestone 4 — Investigation APIs
- Implement get case, discover clue, and request hint.
- Add backend score updates and state checks.
- Test duplicate discovery and hint limits.

**Done when:** API tests prove that game state and scoring cannot be manipulated through request bodies.

### Milestone 5 — Accusation and Results
- Implement accusation finalization and result retrieval.
- Ensure completion is atomic and the solution is revealed only after completion.
- Test correct and incorrect accusations.

**Done when:** A case can be completed once and its result can be safely retrieved again.

### Milestone 6 — Frontend
- Build Home, Create Mystery, Investigation Room, Accusation, and Results pages.
- Connect pages to APIs.
- Add responsive styling, loading states, validation, and error handling.

**Done when:** A player can finish the full game locally.

### Milestone 7 — QA and Deployment
- Run backend, frontend, and end-to-end tests.
- Deploy MongoDB, backend, and frontend.
- Test the production workflow and fix deployment-specific issues.

**Done when:** All acceptance criteria pass on the deployed website.

---

## 18. Acceptance Criteria

The project is complete when all of the following are true:

- [ ] Home page clearly explains the game and links to mystery creation.
- [ ] Players can create mysteries at Easy, Medium, and Hard difficulty.
- [ ] Generated cases have the exact required suspect, clue, and hint counts.
- [ ] AI-generated content is validated before it is saved.
- [ ] The investigation story and public suspect information are displayed correctly.
- [ ] Players can discover clues and receive points only once per clue.
- [ ] Players can request hints within the allowed limit.
- [ ] Hint penalties and difficulty base scores are enforced by the backend.
- [ ] Players can submit one accusation per case.
- [ ] The solution and undiscovered clue text are not exposed before accusation.
- [ ] The result shows the verdict, culprit, explanation, and final score.
- [ ] Case progress persists when the player refreshes the page.
- [ ] API and AI failures are handled gracefully.
- [ ] Secrets are kept out of the frontend and Git repository.
- [ ] The app is responsive and usable on mobile and desktop.
- [ ] The complete game works on the deployed production website.

---

## 19. Final Deliverables

1. `SPEC.md` — this project blueprint.
2. Complete React/Vite/Tailwind frontend.
3. Complete Node.js/Express backend.
4. Mongoose Case model and MongoDB integration.
5. Gemini generation service with strict validation.
6. API endpoints for generation, investigation, hints, accusation, and results.
7. Tests for validation, privacy, scoring, and the main user flow.
8. `.env.example` files and a `.gitignore` that excludes secrets.
9. README with setup, scripts, environment variables, API overview, testing, and deployment instructions.
10. Deployed frontend URL and backend health-check URL.

---

## 20. Instructions for AI Coding Assistants

When implementing this project from this specification:

1. Read the complete `SPEC.md` before changing files.
2. Inspect the existing repository and preserve working code unless a change is required.
3. Implement one milestone at a time.
4. Before each milestone, state which files will be created or changed.
5. Write complete, runnable code rather than pseudocode or unexplained placeholders.
6. Keep frontend and backend responsibilities separate.
7. Validate every request and every Gemini response.
8. Never expose the Gemini key, solution, hidden clue text, or server-authoritative score state.
9. Add tests for each milestone and run them before moving on.
10. Report the commands run, tests passed/failed, environment variables required, and any remaining issues.
11. Do not claim a feature is complete unless it has been implemented and verified.
12. If the specification is ambiguous, choose the safest simple implementation and document the decision instead of silently changing core requirements.
13. Do not put real secrets in source files, prompts, logs, examples, or generated documentation.
