# Nutrition Miniapp Design

## Goal

Build an independent WeChat mini program that lets popkik interact with the existing nutrition recommendation backend through a chat-style flow. The first version focuses on the evening coarse-grain dough allocation workflow:

1. Record what was eaten today.
2. Parse the natural-language input.
3. Confirm or edit the parsed food records.
4. Calculate the evening dough and coarse-grain allocation.
5. Review historical records and recommendation results.

The mini program will use uni-app and Vue 3. It will call the existing `xiaozhi-server` nutrition HTTP endpoints instead of calling `nutrition-service` directly.

## Scope

In scope:

- Independent `nutrition-miniapp` project using uni-app and Vue 3.
- WeChat mini program as the primary target.
- H5 build as a development convenience only.
- Chat-first home page.
- Text input for MVP.
- Voice button placeholder in the input bar.
- Fixed local user identity through `user_id = "popkik"`.
- A user identity helper that can later be replaced by WeChat `openid`.
- Mandatory confirmation before recommendation.
- Editable parsed food records.
- Editable dinner dough request.
- Recommendation result card with numbers first and a short reason.
- Full history pages for past food records and recommendation snapshots.
- Backend support for recommendation snapshots and history APIs.
- A delivery document explaining how to import and run the completed mini program in WeChat Developer Tools.

Out of scope for the first version:

- WeChat `openid` login.
- Real voice input or ASR.
- Full meal planning beyond coarse-grain dough allocation.
- Long-term nutrition trend charts.
- Medical-grade nutrition diagnosis.
- A dedicated miniapp gateway namespace such as `/api/v1/miniapp/...`.
- Public release compliance automation.

## Product Decisions

### App Shape

The mini program will be an independent app, not a page inside the existing `manager-mobile` admin app. This keeps the diet assistant focused and avoids mixing personal food workflows with the management console.

### Interaction Model

The home page is a chat tool. A typical user input is:

```text
早上全麦吐司，中午牛肉面，晚上想包 500g 饺子
```

The app shows:

1. A user message bubble.
2. An assistant parsing status.
3. A parsed-intake confirmation card.
4. A recommendation result card after confirmation.

The user must confirm before the backend saves records and computes the recommendation. This keeps accuracy ahead of speed for the first version.

### Recommendation Display

The result card prioritizes executable kitchen numbers:

- Total dough weight.
- Normal flour weight.
- Coarse-grain flour weight.
- Water weight.
- Coarse-grain ratio.
- One short reason.

The short reason explains why the ratio was chosen, but it does not dominate the UI.

### History

The app includes complete history pages. History is not a complex health analytics feature. It exists so the user can review:

- What food records were confirmed on a date.
- What recommendation was returned.
- What input dough request produced that recommendation.
- Why the recommendation was made.

## Architecture

```text
nutrition-miniapp
  -> xiaozhi-server HTTP API
    -> nutrition-service
      -> SQLite
```

The mini program only talks to `xiaozhi-server`. This keeps deployment simpler because the mini program needs only one HTTPS backend domain when moving toward formal release.

The existing nutrition service remains the source of truth for parsing, storage, and recommendation. `xiaozhi-server` exposes routes and delegates to the service through `NutritionHandler`.

## Frontend Design

### Project

Create an independent `nutrition-miniapp` project. It should follow the existing repository's uni-app and Vue 3 direction, but it does not reuse the admin app's page structure.

Suggested structure:

```text
nutrition-miniapp/
  src/
    pages/
      chat/
        index.vue
      history/
        index.vue
      history-detail/
        index.vue
    components/
      ChatMessageList.vue
      ChatInputBar.vue
      ParsedIntakeCard.vue
      FoodRecordEditor.vue
      DoughRequestEditor.vue
      RecommendationCard.vue
      HistoryListItem.vue
      EmptyState.vue
      ErrorState.vue
    services/
      nutritionApi.ts
      currentUser.ts
    types/
      nutrition.ts
```

### Pages

#### Chat Page

The chat page is the default page. It contains:

- Compact title bar with "晚餐杂粮助手".
- History entry button.
- Message list.
- Text input bar.
- Voice placeholder button.

Flow:

1. User enters text.
2. Page appends a user message.
3. Page calls parse API.
4. Page inserts a parsed confirmation card.
5. User edits or confirms.
6. Page calls recommendation API.
7. Page inserts a recommendation result card.

The chat page can use local reactive state. A global store is not required for the first version.

#### History List Page

Shows historical recommendation summaries in descending date order.

Each item shows:

- Date.
- Target food display.
- Dough total weight.
- Coarse-grain flour weight.
- Coarse-grain ratio.

Clicking an item opens the detail page.

#### History Detail Page

Shows:

- Food records grouped by meal.
- Dinner dough request.
- Recommendation numbers.
- Recommendation reason.
- Snapshot creation time.

### Components

- `ChatMessageList`: renders user bubbles, assistant bubbles, parsed cards, and recommendation cards.
- `ChatInputBar`: text input, send action, disabled/loading state, and voice placeholder.
- `ParsedIntakeCard`: owns the confirmation UI for parsed records and dough request.
- `FoodRecordEditor`: edits one food item.
- `DoughRequestEditor`: edits total dough weight, water ratio if exposed, target food, and target display name.
- `RecommendationCard`: shows kitchen-ready numbers first.
- `HistoryListItem`: compact history row.
- `EmptyState` and `ErrorState`: shared empty and error display.

### Frontend Services

`services/currentUser.ts`:

```ts
export function getCurrentUserId(): string {
  return 'popkik'
}
```

This isolates the identity decision so a later `openid` login change does not spread through pages.

`services/nutritionApi.ts` wraps:

- `POST /api/v1/nutrition/intake/parse`
- `POST /api/v1/nutrition/recommendations/coarse-grain`
- `GET /api/v1/nutrition/intake/today`
- `GET /api/v1/nutrition/history`
- `GET /api/v1/nutrition/history/{date}`

Pages should not build raw URLs directly.

## Backend Design

### Existing Endpoints

Keep these endpoints:

- `POST /api/v1/nutrition/intake/parse`
- `POST /api/v1/nutrition/recommendations/coarse-grain`
- `GET /api/v1/nutrition/intake/today`

The recommendation endpoint should remain backward compatible while adding `snapshot_id` to the response.

### New Model

Add a recommendation snapshot model with:

- `id`
- `user_id`
- `record_date`
- `confirmed_items`
- `dough_request`
- `recommendation`
- `created_at`

For the MVP, `confirmed_items`, `dough_request`, and `recommendation` can be stored as JSON text in SQLite. This is simpler than over-normalizing early and preserves exactly what the user saw.

### Storage

Extend `SQLiteFoodRecordStore` with:

- `save_recommendation_snapshot(user_id, record_date, confirmed_items, dough_request, recommendation)`
- `list_history(user_id, limit, offset)`
- `get_history_detail(user_id, record_date)`

Current `food_records` behavior deletes and rewrites the current date for a user. The first version can keep that model and treat the saved records as the final confirmed state for the day.

Recommendation snapshots should be append-only. This makes it possible to see multiple recommendations generated on the same date.

### New Endpoints

Add routes through `xiaozhi-server`:

```text
GET /api/v1/nutrition/history?user_id=popkik&limit=30&offset=0
GET /api/v1/nutrition/history/{date}?user_id=popkik
```

History list response should return date-level summaries, not full detailed payloads. Detail response should include full food records and recommendation snapshots for that date.

### Recommendation Flow

`NutritionService.confirm_and_recommend(...)` should:

1. Convert confirmed items into `FoodItem`.
2. Convert the dough request into `DoughRequest`.
3. Save confirmed food records for the date.
4. Run `recommend_coarse_grain`.
5. Save a recommendation snapshot.
6. Return the recommendation payload plus `snapshot_id`.

## Data Contracts

### Parse Request

```json
{
  "user_id": "popkik",
  "text": "早上全麦吐司，中午牛肉面，晚上想包 500g 饺子",
  "profile": {
    "age_group": "adult",
    "appetite_level": "normal",
    "digestive_sensitivity": false,
    "health_goal": "balanced"
  }
}
```

### Recommendation Request

```json
{
  "user_id": "popkik",
  "date": "2026-07-08",
  "confirmed_items": [
    {
      "meal": "breakfast",
      "food_name": "whole_wheat_toast",
      "display_name": "全麦吐司",
      "amount_text": "2片",
      "estimated_weight_g": 80,
      "category": "coarse_grain",
      "confidence": 0.9,
      "source": "llm"
    }
  ],
  "dough_request": {
    "dough_total_weight_g": 500,
    "water_ratio": 0.24,
    "target_food": "dumpling",
    "target_food_display": "饺子",
    "weight_source": "explicit"
  }
}
```

### Recommendation Response

```json
{
  "snapshot_id": "snapshot_001",
  "dough_total_weight_g": 500,
  "flour_weight_g": 380,
  "water_weight_g": 120,
  "coarse_grain_ratio": 0.35,
  "coarse_grain_weight_g": 133,
  "reason": "今日精制主食摄入偏多，建议提高本次面团中的杂粮比例。"
}
```

## Error Handling

Frontend behavior:

- Parse failure: show an editable empty confirmation card and let the user manually enter records.
- Recommendation failure: keep confirmed card content and show retry.
- Network failure: show a clear retry state and mention server address configuration.
- Empty history: show a friendly empty state.
- Invalid dough weight: block submission client-side and show a 100-3000 g hint.

Backend behavior:

- Validate `user_id`, `date`, `confirmed_items`, and `dough_request`.
- Reject unreasonable dough weight.
- Return JSON errors consistently.
- Never claim records or snapshots were saved if storage fails.

## Testing Strategy

Backend tests:

- Recommendation saves a snapshot and returns `snapshot_id`.
- History list returns date summaries in descending order.
- History detail returns food records and snapshots for a date.
- Existing parse/recommend/today tests still pass.
- Invalid recommendation requests return errors without saving snapshots.

Frontend tests or manual verification:

- User input creates a user bubble.
- Parse result creates an editable confirmation card.
- Edited confirmed records are submitted to recommendation API.
- Result card displays all key numbers correctly.
- History list handles empty, error, and normal states.
- History detail shows food records and recommendation snapshots.

## WeChat Developer Tools Guide Deliverable

After implementation, write a separate guide:

```text
docs/nutrition-miniapp-wechat-devtools-guide.md
```

The guide must explain:

- Where the mini program project lives.
- Which commands install dependencies.
- Which command builds the WeChat mini program target.
- Which generated directory should be imported into WeChat Developer Tools.
- How to fill AppID for development.
- How to use no-AppID or test mode if applicable.
- How to configure backend base URL.
- How to handle HTTPS and legal request domain settings.
- How to enable local development options such as not validating legal domains.
- How to preview on a phone.
- How to upload an experience version.
- What is needed before formal audit submission.

## Release Expectations

Development can use WeChat Developer Tools preview immediately after the mini program builds.

Formal public release requires:

- Mini program account.
- AppID.
- Valid service category.
- Privacy disclosures.
- HTTPS backend domain.
- Domain allowlist configuration in the mini program platform.
- Audit submission and manual publish after approval.

The first implementation should optimize for self-use and experience-version validation before formal public release. The WeChat Developer Tools guide is a required deliverable of the implementation, not an optional follow-up.
