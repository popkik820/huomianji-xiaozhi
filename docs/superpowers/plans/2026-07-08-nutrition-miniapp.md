# Nutrition Miniapp Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an independent uni-app/Vue 3 WeChat mini program for chat-style evening coarse-grain dough recommendations, backed by `xiaozhi-server` nutrition APIs and persistent recommendation history.

**Architecture:** The mini program calls only `xiaozhi-server`; `xiaozhi-server` delegates to `nutrition-service`; `nutrition-service` stores food records and recommendation snapshots in SQLite. Backend work comes first so the frontend can be developed against stable contracts.

**Tech Stack:** Python unittest, aiohttp, FastAPI TestClient, SQLite, uni-app, Vue 3, TypeScript, pnpm, WeChat Developer Tools.

---

## Scope Check

This plan covers three connected deliverables from the approved spec:

- Backend recommendation snapshots and history APIs.
- Independent `nutrition-miniapp` frontend.
- WeChat Developer Tools import guide.

These are coupled by the mini program's required history feature, so they stay in one plan. Do not add WeChat `openid` login, real voice input, full meal planning, or trend charts in this implementation.

## File Structure

Backend files:

- Modify `nutrition-service/nutrition_service/models.py`: add a `RecommendationSnapshot` dataclass.
- Modify `nutrition-service/nutrition_service/storage.py`: add snapshot table and history methods.
- Modify `nutrition-service/nutrition_service/service.py`: save snapshots and expose history methods.
- Modify `nutrition-service/nutrition_service/api.py`: expose FastAPI history endpoints for standalone service testing.
- Modify `main/xiaozhi-server/core/api/nutrition_handler.py`: expose history handlers through the main HTTP API.
- Modify `main/xiaozhi-server/core/http_server.py`: register main history routes.
- Modify `nutrition-service/tests/test_service_api_cli.py`: add service and FastAPI history tests.
- Modify `nutrition-service/tests/test_main_http_nutrition_integration.py`: add main HTTP history tests.

Frontend files:

- Create `nutrition-miniapp/package.json`: scripts and dependencies.
- Create `nutrition-miniapp/vite.config.ts`: uni-app Vite config.
- Create `nutrition-miniapp/tsconfig.json`: TypeScript config.
- Create `nutrition-miniapp/src/main.ts`: app bootstrap.
- Create `nutrition-miniapp/src/App.vue`: app shell.
- Create `nutrition-miniapp/src/pages.json`: page registration.
- Create `nutrition-miniapp/src/manifest.json`: mini program manifest placeholder.
- Create `nutrition-miniapp/src/uni.scss`: global style tokens.
- Create `nutrition-miniapp/src/types/nutrition.ts`: API and UI types.
- Create `nutrition-miniapp/src/services/currentUser.ts`: fixed `popkik` user helper.
- Create `nutrition-miniapp/src/services/nutritionApi.ts`: API wrapper.
- Create `nutrition-miniapp/src/components/*.vue`: focused UI components.
- Create `nutrition-miniapp/src/pages/chat/index.vue`: chat-first home page.
- Create `nutrition-miniapp/src/pages/history/index.vue`: history list page.
- Create `nutrition-miniapp/src/pages/history-detail/index.vue`: history detail page.

Docs:

- Create `docs/nutrition-miniapp-wechat-devtools-guide.md`: import, build, preview, and release guide.

## Tasks

### Task 1: Recommendation Snapshot Storage Tests

**Files:**
- Modify: `nutrition-service/tests/test_service_api_cli.py`
- Test command: `D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_service_api_cli.py -v`

- [ ] **Step 1: Add failing tests for snapshot save, history list, and history detail**

Add these tests to `ServiceApiCliTests` after `test_service_parse_confirm_recommend_and_today_records`:

```python
    def test_confirm_and_recommend_saves_snapshot_and_history_detail(self):
        parsed = self.service.parse_intake(
            "popkik",
            "早上吃了两个包子，中午一碗米饭，晚上想吃500g面团",
        )

        recommendation = self.service.confirm_and_recommend(
            user_id="popkik",
            date="2026-07-08",
            confirmed_items=parsed["items"],
            dough_request=parsed["dough_request"],
        )

        self.assertIn("snapshot_id", recommendation)
        self.assertEqual(recommendation["coarse_grain_weight_g"], 133)

        history = self.service.list_history("popkik")
        self.assertEqual(len(history["items"]), 1)
        self.assertEqual(history["items"][0]["date"], "2026-07-08")
        self.assertEqual(history["items"][0]["snapshot_id"], recommendation["snapshot_id"])
        self.assertEqual(history["items"][0]["coarse_grain_weight_g"], 133)

        detail = self.service.get_history_detail("popkik", "2026-07-08")
        self.assertEqual(detail["user_id"], "popkik")
        self.assertEqual(detail["date"], "2026-07-08")
        self.assertEqual(len(detail["items"]), 2)
        self.assertEqual(len(detail["snapshots"]), 1)
        self.assertEqual(
            detail["snapshots"][0]["recommendation"]["snapshot_id"],
            recommendation["snapshot_id"],
        )

    def test_history_list_is_descending_by_latest_snapshot(self):
        parsed = self.service.parse_intake(
            "popkik",
            "早上吃了两个包子，中午一碗米饭，晚上想吃500g面团",
        )
        self.service.confirm_and_recommend(
            "popkik",
            "2026-07-07",
            parsed["items"],
            parsed["dough_request"],
        )
        self.service.confirm_and_recommend(
            "popkik",
            "2026-07-08",
            parsed["items"],
            parsed["dough_request"],
        )

        history = self.service.list_history("popkik")

        self.assertEqual([item["date"] for item in history["items"]], ["2026-07-08", "2026-07-07"])
```

- [ ] **Step 2: Run the focused tests and verify failure**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_service_api_cli.py -v
```

Expected: fail with `AttributeError` for missing `list_history` or missing `snapshot_id`.

- [ ] **Step 3: Commit the failing tests**

```powershell
git add nutrition-service/tests/test_service_api_cli.py
git commit -m "test: cover nutrition recommendation history"
```

### Task 2: Recommendation Snapshot Storage Implementation

**Files:**
- Modify: `nutrition-service/nutrition_service/models.py`
- Modify: `nutrition-service/nutrition_service/storage.py`
- Modify: `nutrition-service/nutrition_service/service.py`
- Test: `nutrition-service/tests/test_service_api_cli.py`

- [ ] **Step 1: Add `RecommendationSnapshot` to `models.py`**

Add imports and dataclass:

```python
import json
```

```python
@dataclass
class RecommendationSnapshot:
    id: str
    user_id: str
    record_date: str
    confirmed_items: list[FoodItem]
    dough_request: DoughRequest
    recommendation: Recommendation
    created_at: str

    def to_dict(self) -> dict[str, Any]:
        recommendation = self.recommendation.to_dict()
        recommendation["snapshot_id"] = self.id
        return {
            "id": self.id,
            "user_id": self.user_id,
            "date": self.record_date,
            "confirmed_items": [item.to_dict() for item in self.confirmed_items],
            "dough_request": self.dough_request.to_dict(),
            "recommendation": recommendation,
            "created_at": self.created_at,
        }

    @staticmethod
    def encode_json(payload: Any) -> str:
        return json.dumps(payload, ensure_ascii=False, separators=(",", ":"))
```

- [ ] **Step 2: Add snapshot table in `SQLiteFoodRecordStore._init_db`**

Add this `CREATE TABLE` after `food_records`:

```python
            conn.execute(
                """
                CREATE TABLE IF NOT EXISTS recommendation_snapshots (
                  id TEXT PRIMARY KEY,
                  user_id TEXT NOT NULL,
                  record_date TEXT NOT NULL,
                  confirmed_items_json TEXT NOT NULL,
                  dough_request_json TEXT NOT NULL,
                  recommendation_json TEXT NOT NULL,
                  created_at TEXT NOT NULL
                )
                """
            )
```

- [ ] **Step 3: Add snapshot storage imports**

At the top of `storage.py`, add:

```python
import json
from typing import Any
```

Update the models import:

```python
from .models import DoughRequest, FoodItem, Recommendation
```

- [ ] **Step 4: Add snapshot storage methods**

Add these methods to `SQLiteFoodRecordStore`:

```python
    def save_recommendation_snapshot(
        self,
        user_id: str,
        record_date: str,
        confirmed_items: list[FoodItem],
        dough_request: DoughRequest,
        recommendation: Recommendation,
    ) -> str:
        snapshot_id = uuid.uuid4().hex
        created_at = datetime.utcnow().isoformat(timespec="seconds")
        recommendation_payload = recommendation.to_dict()
        recommendation_payload["snapshot_id"] = snapshot_id
        with sqlite3.connect(self.db_path) as conn:
            conn.execute(
                """
                INSERT INTO recommendation_snapshots (
                    id, user_id, record_date, confirmed_items_json,
                    dough_request_json, recommendation_json, created_at
                ) VALUES (?, ?, ?, ?, ?, ?, ?)
                """,
                (
                    snapshot_id,
                    user_id,
                    record_date,
                    json.dumps([item.to_dict() for item in confirmed_items], ensure_ascii=False),
                    json.dumps(dough_request.to_dict(), ensure_ascii=False),
                    json.dumps(recommendation_payload, ensure_ascii=False),
                    created_at,
                ),
            )
        return snapshot_id

    def list_history(self, user_id: str, limit: int = 30, offset: int = 0) -> dict[str, Any]:
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            rows = conn.execute(
                """
                SELECT record_date, id, dough_request_json, recommendation_json, created_at
                FROM recommendation_snapshots
                WHERE user_id = ?
                ORDER BY record_date DESC, created_at DESC, rowid DESC
                LIMIT ? OFFSET ?
                """,
                (user_id, limit, offset),
            ).fetchall()
        items = []
        for row in rows:
            dough_request = json.loads(row["dough_request_json"])
            recommendation = json.loads(row["recommendation_json"])
            items.append(
                {
                    "date": row["record_date"],
                    "snapshot_id": row["id"],
                    "target_food": dough_request.get("target_food", "unknown"),
                    "target_food_display": dough_request.get("target_food_display", "unknown"),
                    "dough_total_weight_g": recommendation.get("dough_total_weight_g", 0),
                    "coarse_grain_weight_g": recommendation.get("coarse_grain_weight_g", 0),
                    "coarse_grain_ratio": recommendation.get("coarse_grain_ratio", 0),
                    "created_at": row["created_at"],
                }
            )
        return {"user_id": user_id, "items": items, "limit": limit, "offset": offset}

    def get_history_detail(self, user_id: str, record_date: str) -> dict[str, Any]:
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            rows = conn.execute(
                """
                SELECT id, confirmed_items_json, dough_request_json, recommendation_json, created_at
                FROM recommendation_snapshots
                WHERE user_id = ? AND record_date = ?
                ORDER BY created_at DESC, rowid DESC
                """,
                (user_id, record_date),
            ).fetchall()
        snapshots = []
        for row in rows:
            snapshots.append(
                {
                    "id": row["id"],
                    "confirmed_items": json.loads(row["confirmed_items_json"]),
                    "dough_request": json.loads(row["dough_request_json"]),
                    "recommendation": json.loads(row["recommendation_json"]),
                    "created_at": row["created_at"],
                }
            )
        return {
            "user_id": user_id,
            "date": record_date,
            "items": [item.to_dict() for item in self.get_records(user_id, record_date)],
            "snapshots": snapshots,
        }
```

- [ ] **Step 5: Update `NutritionService.confirm_and_recommend` and add history methods**

Replace the final lines of `confirm_and_recommend`:

```python
        recommendation = recommend_coarse_grain(items, request)
        snapshot_id = self.store.save_recommendation_snapshot(
            user_id=user_id,
            record_date=date,
            confirmed_items=items,
            dough_request=request,
            recommendation=recommendation,
        )
        payload = recommendation.to_dict()
        payload["snapshot_id"] = snapshot_id
        return payload
```

Add methods:

```python
    def list_history(self, user_id: str, limit: int = 30, offset: int = 0) -> dict[str, Any]:
        return self.store.list_history(user_id, limit=limit, offset=offset)

    def get_history_detail(self, user_id: str, date: str) -> dict[str, Any]:
        return self.store.get_history_detail(user_id, date)
```

- [ ] **Step 6: Run service tests**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_service_api_cli.py -v
```

Expected: pass.

- [ ] **Step 7: Commit storage implementation**

```powershell
git add nutrition-service/nutrition_service/models.py nutrition-service/nutrition_service/storage.py nutrition-service/nutrition_service/service.py
git commit -m "feat: store nutrition recommendation snapshots"
```

### Task 3: Standalone Nutrition API History Endpoints

**Files:**
- Modify: `nutrition-service/tests/test_service_api_cli.py`
- Modify: `nutrition-service/nutrition_service/api.py`
- Test: `nutrition-service/tests/test_service_api_cli.py`

- [ ] **Step 1: Add failing FastAPI history endpoint test**

Add this test after `test_fastapi_endpoints_return_json`:

```python
    def test_fastapi_history_endpoints_return_snapshots(self):
        client = TestClient(create_app(self.service))
        parsed = self.service.parse_intake(
            "popkik",
            "早上吃了两个包子，中午一碗米饭，晚上想吃500g面团",
        )
        recommendation = self.service.confirm_and_recommend(
            "popkik",
            "2026-07-08",
            parsed["items"],
            parsed["dough_request"],
        )

        list_response = client.get("/api/v1/history", params={"user_id": "popkik"})
        self.assertEqual(list_response.status_code, 200)
        self.assertEqual(list_response.json()["items"][0]["snapshot_id"], recommendation["snapshot_id"])

        detail_response = client.get("/api/v1/history/2026-07-08", params={"user_id": "popkik"})
        self.assertEqual(detail_response.status_code, 200)
        detail = detail_response.json()
        self.assertEqual(detail["date"], "2026-07-08")
        self.assertEqual(detail["snapshots"][0]["recommendation"]["snapshot_id"], recommendation["snapshot_id"])
```

- [ ] **Step 2: Run the test and verify failure**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_service_api_cli.py -v
```

Expected: fail with 404 for `/api/v1/history`.

- [ ] **Step 3: Add FastAPI routes**

In `nutrition-service/nutrition_service/api.py`, add routes inside `create_app`:

```python
    @app.get("/api/v1/history")
    def history(
        user_id: str = Query(...),
        limit: int = Query(30),
        offset: int = Query(0),
    ):
        return active_service.list_history(user_id, limit=limit, offset=offset)

    @app.get("/api/v1/history/{record_date}")
    def history_detail(record_date: str, user_id: str = Query(...)):
        return active_service.get_history_detail(user_id, record_date)
```

- [ ] **Step 4: Run service API tests**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_service_api_cli.py -v
```

Expected: pass.

- [ ] **Step 5: Commit API endpoints**

```powershell
git add nutrition-service/tests/test_service_api_cli.py nutrition-service/nutrition_service/api.py
git commit -m "feat: expose nutrition history API"
```

### Task 4: Main HTTP History Routes

**Files:**
- Modify: `nutrition-service/tests/test_main_http_nutrition_integration.py`
- Modify: `main/xiaozhi-server/core/api/nutrition_handler.py`
- Modify: `main/xiaozhi-server/core/http_server.py`
- Test: `nutrition-service/tests/test_main_http_nutrition_integration.py`

- [ ] **Step 1: Extend the stub service**

In `StubNutritionService`, add:

```python
    def list_history(self, user_id, limit=30, offset=0):
        return {
            "user_id": user_id,
            "items": [
                {
                    "date": "2026-07-08",
                    "snapshot_id": "snapshot-test",
                    "target_food": "dumpling",
                    "target_food_display": "饺子",
                    "dough_total_weight_g": 300,
                    "coarse_grain_weight_g": 80,
                    "coarse_grain_ratio": 0.35,
                    "created_at": "2026-07-08T18:00:00",
                }
            ],
            "limit": limit,
            "offset": offset,
        }

    def get_history_detail(self, user_id, date):
        return {
            "user_id": user_id,
            "date": date,
            "items": [],
            "snapshots": [
                {
                    "id": "snapshot-test",
                    "recommendation": {"snapshot_id": "snapshot-test"},
                }
            ],
        }
```

- [ ] **Step 2: Add failing main HTTP history tests**

Add tests:

```python
    async def test_main_http_server_exposes_nutrition_history_route(self):
        response = await self.client.get(
            "/api/v1/nutrition/history",
            params={"user_id": "popkik", "limit": "10", "offset": "0"},
        )

        self.assertEqual(response.status, 200)
        payload = await response.json()
        self.assertEqual(payload["items"][0]["snapshot_id"], "snapshot-test")

    async def test_main_http_server_exposes_nutrition_history_detail_route(self):
        response = await self.client.get(
            "/api/v1/nutrition/history/2026-07-08",
            params={"user_id": "popkik"},
        )

        self.assertEqual(response.status, 200)
        payload = await response.json()
        self.assertEqual(payload["date"], "2026-07-08")
        self.assertEqual(payload["snapshots"][0]["id"], "snapshot-test")
```

- [ ] **Step 3: Run the integration test and verify failure**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_main_http_nutrition_integration.py -v
```

Expected: fail with 404 for history routes.

- [ ] **Step 4: Add handler methods**

In `NutritionHandler`, add:

```python
    async def handle_history(self, request):
        try:
            user_id = str(request.query.get("user_id", ""))
            limit = int(request.query.get("limit", 30))
            offset = int(request.query.get("offset", 0))
            result = self.service.list_history(user_id, limit=limit, offset=offset)
            return self._json_response(result)
        except Exception as e:
            self.logger.bind(tag=TAG).error(f"Nutrition history request failed: {e}")
            return self._json_response({"success": False, "message": str(e)}, status=400)

    async def handle_history_detail(self, request):
        try:
            user_id = str(request.query.get("user_id", ""))
            record_date = str(request.match_info.get("date", ""))
            result = self.service.get_history_detail(user_id, record_date)
            return self._json_response(result)
        except Exception as e:
            self.logger.bind(tag=TAG).error(f"Nutrition history detail request failed: {e}")
            return self._json_response({"success": False, "message": str(e)}, status=400)
```

- [ ] **Step 5: Register main HTTP routes**

In `SimpleHttpServer.create_app`, add:

```python
                web.get(
                    "/api/v1/nutrition/history",
                    self.nutrition_handler.handle_history,
                ),
                web.options(
                    "/api/v1/nutrition/history",
                    self.nutrition_handler.handle_options,
                ),
                web.get(
                    "/api/v1/nutrition/history/{date}",
                    self.nutrition_handler.handle_history_detail,
                ),
                web.options(
                    "/api/v1/nutrition/history/{date}",
                    self.nutrition_handler.handle_options,
                ),
```

- [ ] **Step 6: Run main HTTP nutrition integration tests**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_main_http_nutrition_integration.py -v
```

Expected: pass.

- [ ] **Step 7: Commit main HTTP routes**

```powershell
git add nutrition-service/tests/test_main_http_nutrition_integration.py main/xiaozhi-server/core/api/nutrition_handler.py main/xiaozhi-server/core/http_server.py
git commit -m "feat: add main nutrition history routes"
```

### Task 5: Miniapp Project Scaffold

**Files:**
- Create: `nutrition-miniapp/package.json`
- Create: `nutrition-miniapp/vite.config.ts`
- Create: `nutrition-miniapp/tsconfig.json`
- Create: `nutrition-miniapp/src/main.ts`
- Create: `nutrition-miniapp/src/App.vue`
- Create: `nutrition-miniapp/src/pages.json`
- Create: `nutrition-miniapp/src/manifest.json`
- Create: `nutrition-miniapp/src/uni.scss`

- [ ] **Step 1: Create package scripts**

Create `nutrition-miniapp/package.json`:

```json
{
  "name": "nutrition-miniapp",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev:h5": "uni",
    "dev:mp-weixin": "uni -p mp-weixin",
    "build:h5": "uni build",
    "build:mp-weixin": "uni build -p mp-weixin",
    "type-check": "vue-tsc --noEmit"
  },
  "dependencies": {
    "@dcloudio/uni-app": "3.0.0-4060620250520001",
    "@dcloudio/uni-components": "3.0.0-4060620250520001",
    "@dcloudio/uni-h5": "3.0.0-4060620250520001",
    "@dcloudio/uni-mp-weixin": "3.0.0-4060620250520001",
    "vue": "^3.4.21"
  },
  "devDependencies": {
    "@dcloudio/types": "^3.4.8",
    "@dcloudio/vite-plugin-uni": "3.0.0-4060620250520001",
    "@vue/tsconfig": "^0.1.3",
    "typescript": "^5.7.2",
    "vite": "5.2.8",
    "vue-tsc": "^2.2.10"
  }
}
```

- [ ] **Step 2: Create Vite config**

Create `nutrition-miniapp/vite.config.ts`:

```ts
import uni from '@dcloudio/vite-plugin-uni'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [uni()],
  resolve: {
    alias: {
      '@': '/src',
    },
  },
})
```

- [ ] **Step 3: Create TypeScript config**

Create `nutrition-miniapp/tsconfig.json`:

```json
{
  "extends": "@vue/tsconfig/tsconfig.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "types": ["@dcloudio/types"]
  },
  "include": ["src/**/*.ts", "src/**/*.vue"]
}
```

- [ ] **Step 4: Create app bootstrap**

Create `nutrition-miniapp/src/main.ts`:

```ts
import { createSSRApp } from 'vue'
import App from './App.vue'

export function createApp() {
  const app = createSSRApp(App)
  return { app }
}
```

Create `nutrition-miniapp/src/App.vue`:

```vue
<script setup lang="ts">
</script>

<template>
  <slot />
</template>

<style lang="scss">
page {
  min-height: 100%;
  background: #f6f8f5;
  color: #202923;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}
</style>
```

- [ ] **Step 5: Create pages and manifest config**

Create `nutrition-miniapp/src/pages.json`:

```json
{
  "pages": [
    {
      "path": "pages/chat/index",
      "style": {
        "navigationBarTitleText": "晚餐杂粮助手",
        "navigationBarBackgroundColor": "#f6f8f5",
        "navigationBarTextStyle": "black"
      }
    },
    {
      "path": "pages/history/index",
      "style": {
        "navigationBarTitleText": "历史记录"
      }
    },
    {
      "path": "pages/history-detail/index",
      "style": {
        "navigationBarTitleText": "历史详情"
      }
    }
  ],
  "globalStyle": {
    "navigationBarTextStyle": "black",
    "navigationBarTitleText": "晚餐杂粮助手",
    "navigationBarBackgroundColor": "#f6f8f5",
    "backgroundColor": "#f6f8f5"
  }
}
```

Create `nutrition-miniapp/src/manifest.json`:

```json
{
  "name": "晚餐杂粮助手",
  "appid": "__UNI__NUTRITION",
  "description": "Chat-style coarse-grain dinner dough assistant",
  "versionName": "0.1.0",
  "versionCode": "1",
  "mp-weixin": {
    "appid": "",
    "setting": {
      "urlCheck": false
    },
    "usingComponents": true
  }
}
```

Create `nutrition-miniapp/src/uni.scss`:

```scss
$nutrition-bg: #f6f8f5;
$nutrition-surface: #ffffff;
$nutrition-border: #dfe7df;
$nutrition-primary: #2f6f45;
$nutrition-text: #202923;
$nutrition-muted: #657268;
```

- [ ] **Step 6: Commit scaffold**

```powershell
git add nutrition-miniapp/package.json nutrition-miniapp/vite.config.ts nutrition-miniapp/tsconfig.json nutrition-miniapp/src
git commit -m "feat: scaffold nutrition miniapp"
```

### Task 6: Miniapp API Types and Service Wrapper

**Files:**
- Create: `nutrition-miniapp/src/types/nutrition.ts`
- Create: `nutrition-miniapp/src/services/currentUser.ts`
- Create: `nutrition-miniapp/src/services/nutritionApi.ts`

- [ ] **Step 1: Create nutrition types**

Create `nutrition-miniapp/src/types/nutrition.ts`:

```ts
export type Meal = 'breakfast' | 'lunch' | 'dinner' | 'unknown'

export type FoodCategory =
  | 'refined_staple'
  | 'coarse_grain'
  | 'protein'
  | 'vegetable'
  | 'fruit'
  | 'fat_sugar'
  | 'unknown'

export interface FoodItem {
  meal: Meal
  food_name: string
  display_name: string
  amount_text: string
  estimated_weight_g: number
  category: FoodCategory
  confidence?: number
  source?: string
  base_food?: string
  modifier?: string
}

export interface DoughRequest {
  dough_total_weight_g: number
  water_ratio: number
  target_food: string
  target_food_display: string
  weight_source: string
}

export interface ParseResponse {
  items: FoodItem[]
  dough_request: DoughRequest
  needs_confirmation: boolean
  digestive_risk?: boolean
}

export interface RecommendationResponse {
  snapshot_id: string
  dough_total_weight_g: number
  flour_weight_g: number
  water_weight_g: number
  coarse_grain_ratio: number
  coarse_grain_weight_g: number
  reason: string
}

export interface HistorySummary {
  date: string
  snapshot_id: string
  target_food: string
  target_food_display: string
  dough_total_weight_g: number
  coarse_grain_weight_g: number
  coarse_grain_ratio: number
  created_at: string
}

export interface HistoryDetail {
  user_id: string
  date: string
  items: FoodItem[]
  snapshots: Array<{
    id: string
    confirmed_items: FoodItem[]
    dough_request: DoughRequest
    recommendation: RecommendationResponse
    created_at: string
  }>
}
```

- [ ] **Step 2: Create fixed user helper**

Create `nutrition-miniapp/src/services/currentUser.ts`:

```ts
export function getCurrentUserId(): string {
  return 'popkik'
}
```

- [ ] **Step 3: Create API wrapper**

Create `nutrition-miniapp/src/services/nutritionApi.ts`:

```ts
import type {
  DoughRequest,
  FoodItem,
  HistoryDetail,
  HistorySummary,
  ParseResponse,
  RecommendationResponse,
} from '@/types/nutrition'

const DEFAULT_BASE_URL = 'http://127.0.0.1:8003'

function getBaseUrl(): string {
  return uni.getStorageSync('nutrition_api_base_url') || DEFAULT_BASE_URL
}

function request<T>(path: string, options: UniApp.RequestOptions = {}): Promise<T> {
  return new Promise((resolve, reject) => {
    uni.request({
      url: `${getBaseUrl()}${path}`,
      method: options.method || 'GET',
      data: options.data,
      header: {
        'content-type': 'application/json',
        ...(options.header || {}),
      },
      success(response) {
        if (response.statusCode >= 200 && response.statusCode < 300) {
          resolve(response.data as T)
        } else {
          reject(new Error(`HTTP ${response.statusCode}`))
        }
      },
      fail(error) {
        reject(new Error(error.errMsg))
      },
    })
  })
}

export function parseIntake(userId: string, text: string): Promise<ParseResponse> {
  return request<ParseResponse>('/api/v1/nutrition/intake/parse', {
    method: 'POST',
    data: {
      user_id: userId,
      text,
      profile: {
        age_group: 'adult',
        appetite_level: 'normal',
        digestive_sensitivity: false,
        health_goal: 'balanced',
      },
    },
  })
}

export function recommendCoarseGrain(
  userId: string,
  date: string,
  confirmedItems: FoodItem[],
  doughRequest: DoughRequest,
): Promise<RecommendationResponse> {
  return request<RecommendationResponse>('/api/v1/nutrition/recommendations/coarse-grain', {
    method: 'POST',
    data: {
      user_id: userId,
      date,
      confirmed_items: confirmedItems,
      dough_request: doughRequest,
    },
  })
}

export function listHistory(userId: string): Promise<{ user_id: string, items: HistorySummary[] }> {
  return request<{ user_id: string, items: HistorySummary[] }>(
    `/api/v1/nutrition/history?user_id=${encodeURIComponent(userId)}`,
  )
}

export function getHistoryDetail(userId: string, date: string): Promise<HistoryDetail> {
  return request<HistoryDetail>(
    `/api/v1/nutrition/history/${encodeURIComponent(date)}?user_id=${encodeURIComponent(userId)}`,
  )
}
```

- [ ] **Step 4: Commit frontend service layer**

```powershell
git add nutrition-miniapp/src/types/nutrition.ts nutrition-miniapp/src/services
git commit -m "feat: add nutrition miniapp API services"
```

### Task 7: Chat Page Components

**Files:**
- Create: `nutrition-miniapp/src/components/ChatInputBar.vue`
- Create: `nutrition-miniapp/src/components/RecommendationCard.vue`
- Create: `nutrition-miniapp/src/components/ParsedIntakeCard.vue`
- Create: `nutrition-miniapp/src/pages/chat/index.vue`

- [ ] **Step 1: Create chat input component**

Create `nutrition-miniapp/src/components/ChatInputBar.vue`:

```vue
<script setup lang="ts">
import { ref } from 'vue'

defineProps<{ loading?: boolean }>()
const emit = defineEmits<{ send: [text: string] }>()
const text = ref('')

function send() {
  const value = text.value.trim()
  if (!value)
    return
  emit('send', value)
  text.value = ''
}

function showVoiceNotice() {
  uni.showToast({ title: '语音输入后续支持', icon: 'none' })
}
</script>

<template>
  <view class="input-bar">
    <button class="voice" @tap="showVoiceNotice">🎙</button>
    <input v-model="text" class="input" confirm-type="send" placeholder="输入今天吃了什么..." @confirm="send" />
    <button class="send" :disabled="loading" @tap="send">发送</button>
  </view>
</template>

<style scoped lang="scss">
.input-bar { display: flex; gap: 8px; padding: 10px 12px; background: #fff; border-top: 1px solid #dfe7df; }
.voice { width: 40px; height: 40px; border-radius: 20px; padding: 0; background: #eef3ef; }
.input { flex: 1; height: 40px; padding: 0 12px; border-radius: 20px; background: #f6f8f5; }
.send { min-width: 64px; height: 40px; border-radius: 20px; background: #2f6f45; color: #fff; }
</style>
```

- [ ] **Step 2: Create recommendation card**

Create `nutrition-miniapp/src/components/RecommendationCard.vue`:

```vue
<script setup lang="ts">
import type { RecommendationResponse } from '@/types/nutrition'
defineProps<{ recommendation: RecommendationResponse }>()
</script>

<template>
  <view class="card">
    <view class="label">推荐杂粮粉</view>
    <view class="primary">{{ recommendation.coarse_grain_weight_g }}g</view>
    <view class="grid">
      <view>总面团 <text>{{ recommendation.dough_total_weight_g }}g</text></view>
      <view>普通面粉 <text>{{ recommendation.flour_weight_g - recommendation.coarse_grain_weight_g }}g</text></view>
      <view>水 <text>{{ recommendation.water_weight_g }}g</text></view>
      <view>比例 <text>{{ Math.round(recommendation.coarse_grain_ratio * 100) }}%</text></view>
    </view>
    <view class="reason">{{ recommendation.reason }}</view>
  </view>
</template>

<style scoped lang="scss">
.card { padding: 16px; border-radius: 8px; background: #fff; border: 1px solid #dfe7df; }
.label { color: #657268; font-size: 13px; }
.primary { margin-top: 4px; font-size: 36px; font-weight: 800; color: #2f6f45; }
.grid { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-top: 12px; }
.grid view { padding: 10px; border-radius: 8px; background: #f6f8f5; color: #657268; }
.grid text { display: block; margin-top: 4px; color: #202923; font-weight: 700; }
.reason { margin-top: 12px; color: #657268; line-height: 1.5; }
</style>
```

- [ ] **Step 3: Create parsed intake card**

Create `nutrition-miniapp/src/components/ParsedIntakeCard.vue`:

```vue
<script setup lang="ts">
import type { DoughRequest, FoodItem } from '@/types/nutrition'
import { reactive } from 'vue'

const props = defineProps<{ items: FoodItem[], doughRequest: DoughRequest }>()
const emit = defineEmits<{ confirm: [items: FoodItem[], doughRequest: DoughRequest] }>()

const localItems = reactive(props.items.map(item => ({ ...item })))
const localDough = reactive({ ...props.doughRequest })

function confirm() {
  emit('confirm', localItems.map(item => ({ ...item })), { ...localDough })
}
</script>

<template>
  <view class="card">
    <view class="title">请确认识别结果</view>
    <view v-for="(item, index) in localItems" :key="index" class="item">
      <input v-model="item.display_name" class="field" placeholder="食物" />
      <input v-model="item.amount_text" class="field" placeholder="数量" />
      <input v-model.number="item.estimated_weight_g" class="field" type="number" placeholder="重量g" />
    </view>
    <view class="dough">
      <text>晚餐面团重量</text>
      <input v-model.number="localDough.dough_total_weight_g" class="field" type="number" />
    </view>
    <button class="confirm" @tap="confirm">确认并计算</button>
  </view>
</template>

<style scoped lang="scss">
.card { padding: 14px; border-radius: 8px; background: #fff; border: 1px solid #dfe7df; }
.title { font-weight: 700; margin-bottom: 10px; }
.item { display: grid; grid-template-columns: 1fr 72px 72px; gap: 8px; margin-bottom: 8px; }
.field { min-height: 36px; padding: 0 8px; border-radius: 6px; background: #f6f8f5; }
.dough { display: flex; align-items: center; gap: 8px; margin-top: 12px; color: #657268; }
.dough .field { width: 100px; }
.confirm { margin-top: 12px; background: #2f6f45; color: #fff; border-radius: 20px; }
</style>
```

- [ ] **Step 4: Create chat page**

Create `nutrition-miniapp/src/pages/chat/index.vue`:

```vue
<script setup lang="ts">
import ChatInputBar from '@/components/ChatInputBar.vue'
import ParsedIntakeCard from '@/components/ParsedIntakeCard.vue'
import RecommendationCard from '@/components/RecommendationCard.vue'
import { getCurrentUserId } from '@/services/currentUser'
import { parseIntake, recommendCoarseGrain } from '@/services/nutritionApi'
import type { DoughRequest, FoodItem, RecommendationResponse } from '@/types/nutrition'
import { ref } from 'vue'

type Message =
  | { type: 'user', text: string }
  | { type: 'assistant', text: string }
  | { type: 'parsed', items: FoodItem[], doughRequest: DoughRequest }
  | { type: 'recommendation', recommendation: RecommendationResponse }

const messages = ref<Message[]>([
  { type: 'assistant', text: '告诉我今天吃了什么，以及晚上想做多少面团。' },
])
const loading = ref(false)

function today() {
  return new Date().toISOString().slice(0, 10)
}

async function send(text: string) {
  messages.value.push({ type: 'user', text })
  loading.value = true
  try {
    const parsed = await parseIntake(getCurrentUserId(), text)
    messages.value.push({ type: 'parsed', items: parsed.items, doughRequest: parsed.dough_request })
  } catch (error) {
    messages.value.push({ type: 'assistant', text: '没识别清楚，可以换一种说法再试。' })
  } finally {
    loading.value = false
  }
}

async function confirm(items: FoodItem[], doughRequest: DoughRequest) {
  loading.value = true
  try {
    const recommendation = await recommendCoarseGrain(getCurrentUserId(), today(), items, doughRequest)
    messages.value.push({ type: 'recommendation', recommendation })
  } catch (error) {
    messages.value.push({ type: 'assistant', text: '推荐计算失败，请检查服务器后重试。' })
  } finally {
    loading.value = false
  }
}

function openHistory() {
  uni.navigateTo({ url: '/pages/history/index' })
}
</script>

<template>
  <view class="page">
    <view class="header">
      <text>晚餐杂粮助手</text>
      <button @tap="openHistory">历史</button>
    </view>
    <scroll-view class="messages" scroll-y>
      <view v-for="(message, index) in messages" :key="index" class="message" :class="message.type">
        <text v-if="message.type === 'user' || message.type === 'assistant'">{{ message.text }}</text>
        <ParsedIntakeCard
          v-else-if="message.type === 'parsed'"
          :items="message.items"
          :dough-request="message.doughRequest"
          @confirm="confirm"
        />
        <RecommendationCard
          v-else-if="message.type === 'recommendation'"
          :recommendation="message.recommendation"
        />
      </view>
    </scroll-view>
    <ChatInputBar :loading="loading" @send="send" />
  </view>
</template>

<style scoped lang="scss">
.page { min-height: 100vh; display: flex; flex-direction: column; background: #f6f8f5; }
.header { display: flex; align-items: center; justify-content: space-between; padding: 12px 16px; font-weight: 700; }
.header button { margin: 0; border-radius: 18px; background: #fff; font-size: 13px; }
.messages { flex: 1; min-height: 0; padding: 12px; box-sizing: border-box; }
.message { margin-bottom: 12px; max-width: 88%; }
.message.assistant { padding: 10px 12px; border-radius: 8px; background: #fff; color: #202923; }
.message.user { margin-left: auto; padding: 10px 12px; border-radius: 8px; background: #dcefe2; color: #202923; }
.message.parsed, .message.recommendation { max-width: 100%; }
</style>
```

- [ ] **Step 5: Commit chat UI**

```powershell
git add nutrition-miniapp/src/components/ChatInputBar.vue nutrition-miniapp/src/components/RecommendationCard.vue nutrition-miniapp/src/components/ParsedIntakeCard.vue nutrition-miniapp/src/pages/chat/index.vue
git commit -m "feat: add nutrition chat workflow"
```

### Task 8: History Pages

**Files:**
- Create: `nutrition-miniapp/src/components/HistoryListItem.vue`
- Create: `nutrition-miniapp/src/pages/history/index.vue`
- Create: `nutrition-miniapp/src/pages/history-detail/index.vue`

- [ ] **Step 1: Create history list item**

Create `nutrition-miniapp/src/components/HistoryListItem.vue`:

```vue
<script setup lang="ts">
import type { HistorySummary } from '@/types/nutrition'
defineProps<{ item: HistorySummary }>()
</script>

<template>
  <view class="item">
    <view>
      <view class="date">{{ item.date }}</view>
      <view class="meta">{{ item.target_food_display }} · 面团 {{ item.dough_total_weight_g }}g</view>
    </view>
    <view class="amount">{{ item.coarse_grain_weight_g }}g</view>
  </view>
</template>

<style scoped lang="scss">
.item { display: flex; justify-content: space-between; align-items: center; padding: 14px; border-radius: 8px; background: #fff; border: 1px solid #dfe7df; }
.date { font-weight: 700; color: #202923; }
.meta { margin-top: 4px; color: #657268; font-size: 13px; }
.amount { color: #2f6f45; font-size: 24px; font-weight: 800; }
</style>
```

- [ ] **Step 2: Create history list page**

Create `nutrition-miniapp/src/pages/history/index.vue`:

```vue
<script setup lang="ts">
import HistoryListItem from '@/components/HistoryListItem.vue'
import { getCurrentUserId } from '@/services/currentUser'
import { listHistory } from '@/services/nutritionApi'
import type { HistorySummary } from '@/types/nutrition'
import { onLoad } from '@dcloudio/uni-app'
import { ref } from 'vue'

const items = ref<HistorySummary[]>([])
const error = ref('')

async function load() {
  try {
    const result = await listHistory(getCurrentUserId())
    items.value = result.items
  } catch (err) {
    error.value = '历史记录加载失败'
  }
}

function openDetail(item: HistorySummary) {
  uni.navigateTo({ url: `/pages/history-detail/index?date=${encodeURIComponent(item.date)}` })
}

onLoad(load)
</script>

<template>
  <view class="page">
    <view v-if="error" class="state">{{ error }}</view>
    <view v-else-if="items.length === 0" class="state">暂无历史记录</view>
    <view v-else class="list">
      <HistoryListItem v-for="item in items" :key="item.snapshot_id" :item="item" @tap="openDetail(item)" />
    </view>
  </view>
</template>

<style scoped lang="scss">
.page { min-height: 100vh; padding: 12px; background: #f6f8f5; box-sizing: border-box; }
.list { display: grid; gap: 10px; }
.state { padding: 32px 16px; color: #657268; text-align: center; }
</style>
```

- [ ] **Step 3: Create history detail page**

Create `nutrition-miniapp/src/pages/history-detail/index.vue`:

```vue
<script setup lang="ts">
import RecommendationCard from '@/components/RecommendationCard.vue'
import { getCurrentUserId } from '@/services/currentUser'
import { getHistoryDetail } from '@/services/nutritionApi'
import type { HistoryDetail } from '@/types/nutrition'
import { onLoad } from '@dcloudio/uni-app'
import { ref } from 'vue'

const detail = ref<HistoryDetail | null>(null)
const error = ref('')

onLoad(async (query) => {
  const date = String(query.date || '')
  try {
    detail.value = await getHistoryDetail(getCurrentUserId(), date)
  } catch (err) {
    error.value = '历史详情加载失败'
  }
})
</script>

<template>
  <view class="page">
    <view v-if="error" class="state">{{ error }}</view>
    <view v-else-if="!detail" class="state">加载中...</view>
    <view v-else class="content">
      <view class="section">
        <view class="title">{{ detail.date }} 饮食记录</view>
        <view v-for="(item, index) in detail.items" :key="index" class="record">
          {{ item.meal }} · {{ item.display_name }} · {{ item.amount_text }} · {{ item.estimated_weight_g }}g
        </view>
      </view>
      <view v-for="snapshot in detail.snapshots" :key="snapshot.id" class="section">
        <view class="title">推荐结果 {{ snapshot.created_at }}</view>
        <RecommendationCard :recommendation="snapshot.recommendation" />
      </view>
    </view>
  </view>
</template>

<style scoped lang="scss">
.page { min-height: 100vh; padding: 12px; background: #f6f8f5; box-sizing: border-box; }
.content { display: grid; gap: 12px; }
.section { padding: 14px; border-radius: 8px; background: #fff; border: 1px solid #dfe7df; }
.title { margin-bottom: 10px; font-weight: 700; }
.record { padding: 8px 0; color: #657268; border-top: 1px solid #eef2ef; }
.record:first-of-type { border-top: 0; }
.state { padding: 32px 16px; color: #657268; text-align: center; }
</style>
```

- [ ] **Step 4: Commit history pages**

```powershell
git add nutrition-miniapp/src/components/HistoryListItem.vue nutrition-miniapp/src/pages/history nutrition-miniapp/src/pages/history-detail
git commit -m "feat: add nutrition history pages"
```

### Task 9: WeChat Developer Tools Guide

**Files:**
- Create: `docs/nutrition-miniapp-wechat-devtools-guide.md`

- [ ] **Step 1: Write the guide**

Create `docs/nutrition-miniapp-wechat-devtools-guide.md`:

```markdown
# Nutrition Miniapp WeChat Developer Tools Guide

## Project Location

The mini program source project is:

```text
nutrition-miniapp
```

The WeChat mini program build output is:

```text
nutrition-miniapp/dist/build/mp-weixin
```

Import the build output directory into WeChat Developer Tools after running the build command.

## Install Dependencies

From the repository root:

```powershell
cd nutrition-miniapp
pnpm install
```

## Configure Backend Base URL

The mini program defaults to:

```text
http://127.0.0.1:8003
```

For phone preview or formal release, set the backend to a reachable HTTPS domain. The backend must expose:

```text
POST /api/v1/nutrition/intake/parse
POST /api/v1/nutrition/recommendations/coarse-grain
GET /api/v1/nutrition/history
GET /api/v1/nutrition/history/{date}
```

During local development, WeChat Developer Tools can disable legal domain validation in the project settings.

## Build WeChat Mini Program

```powershell
cd nutrition-miniapp
pnpm build:mp-weixin
```

## Import Into WeChat Developer Tools

1. Open WeChat Developer Tools.
2. Choose Import Project.
3. Select:

```text
nutrition-miniapp/dist/build/mp-weixin
```

4. Enter your AppID. If you do not have an AppID, use the available test/no-AppID mode for local preview.
5. Open project settings.
6. For local development, enable the option that skips legal domain, web-view, TLS, and HTTPS certificate validation.

## Preview On Phone

1. Ensure the backend is reachable from the phone.
2. Build the `mp-weixin` target again.
3. In WeChat Developer Tools, click Preview.
4. Scan the QR code with WeChat.

## Upload Experience Version

1. Make sure the AppID is configured.
2. Make sure the backend domain is HTTPS and configured as a legal request domain in the mini program platform.
3. In WeChat Developer Tools, click Upload.
4. Add a version number and description.
5. In the WeChat public platform, set experience members and test the uploaded version.

## Before Formal Audit

Prepare:

- Mini program account and AppID.
- Valid service category.
- Privacy policy and user data disclosure.
- HTTPS backend domain.
- Legal request domain configuration.
- Working experience version.
- Screenshots and descriptions that match the submitted feature.

Submit for audit only after the experience version works on a real phone.
```

- [ ] **Step 2: Commit guide**

```powershell
git add docs/nutrition-miniapp-wechat-devtools-guide.md
git commit -m "docs: add nutrition miniapp developer tools guide"
```

### Task 10: Full Verification

**Files:**
- All backend and miniapp files from previous tasks.

- [ ] **Step 1: Run backend service API tests**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_service_api_cli.py -v
```

Expected: pass.

- [ ] **Step 2: Run main HTTP integration tests**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest nutrition-service\tests\test_main_http_nutrition_integration.py -v
```

Expected: pass.

- [ ] **Step 3: Run all nutrition-service tests**

Run:

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest discover -s nutrition-service\tests -t nutrition-service -v
```

Expected: pass.

- [ ] **Step 4: Install miniapp dependencies if needed**

Run:

```powershell
cd nutrition-miniapp
pnpm install
```

Expected: dependencies install successfully.

- [ ] **Step 5: Type-check miniapp**

Run:

```powershell
cd nutrition-miniapp
pnpm type-check
```

Expected: pass.

- [ ] **Step 6: Build WeChat mini program**

Run:

```powershell
cd nutrition-miniapp
pnpm build:mp-weixin
```

Expected: build succeeds and produces `nutrition-miniapp/dist/build/mp-weixin`.

- [ ] **Step 7: Commit final verification fixes if any**

Only if changes were needed during verification:

```powershell
git add nutrition-miniapp nutrition-service main/xiaozhi-server docs/nutrition-miniapp-wechat-devtools-guide.md
git commit -m "fix: polish nutrition miniapp verification"
```

If no changes were needed, do not create an empty commit.

## Self-Review

- Spec coverage: The plan includes backend snapshots, history APIs, independent miniapp scaffold, chat workflow, history pages, fixed `popkik` identity, voice placeholder, and WeChat Developer Tools guide.
- Placeholder scan: The plan contains no `TBD`, no `TODO`, and no "implement later" steps.
- Type consistency: The same `snapshot_id`, `FoodItem`, `DoughRequest`, `RecommendationResponse`, and history route names are used across backend, frontend, and docs.
