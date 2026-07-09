# 营养服务 MVP 设计

## 目标

为全自动和面机项目构建一个独立的 `nutrition-service` MVP。该服务接收微信小程序传来的饮食描述，记录用户当天已经吃过的食物，并为下一次和面任务推荐最终面团总重量、杂粮比例和杂粮重量。

第一阶段该服务保持独立，不直接修改 `xiaozhi-server`。等营养计算逻辑稳定后，再让 `xiaozhi-server` 通过 HTTP 工具或 MCP 调用它，使语音交互也能复用同一套营养逻辑。

## 范围

MVP 包含：

- 小程序输入自然语言饮食记录。
- 使用 LLM 解析饮食描述，并用确定性规则兜底。
- 将可编辑的结构化解析结果返回给小程序确认。
- 按用户保存当天饮食记录。
- 支持轻量用户画像：年龄组、食量等级、消化敏感程度。
- 识别目标面食种类，例如饺子、面条、包子、馒头、烙饼。
- 当用户没有明确输入面团重量时，根据用户画像和面食种类估算面团重量。
- 识别“荞麦面条、全麦馒头、全麦面包、燕麦粥、杂粮饭、玉米面条”等常见杂粮和复合主食，避免只命中“面条”或“馒头”主体。
- 基于通用成人饮食规则推荐杂粮比例。
- 输出最终面团总重量、杂粮比例和杂粮重量。

MVP 不包含：

- ESP32 或和面机控制。
- 长期健康趋势分析。
- 基于身高、体重、年龄、性别、运动量、疾病状态的个人画像建模。
- QLoRA 微调。
- 医学级营养诊断。
- 完整粉料配方、水比例动态控制、和面时长或醒面建议。

## 用户流程

1. 用户在小程序中输入或说出一句话，例如：“我早上吃了两个包子，中午吃了一碗米饭和一个鸡蛋，晚上想吃 500g 面团。”
2. 小程序把文本发送给 `nutrition-service`。
3. `nutrition-service` 调用 LLM，提取结构化饮食记录和用户想要的面团重量。
4. `nutrition-service` 校验 LLM 输出。如果输出无效或不完整，则使用规则解析器提取已知食物。
5. 小程序以可编辑表单展示解析出的食物记录和面团重量。
6. 用户确认或修改解析结果。
7. `nutrition-service` 保存当天确认过的饮食记录。
8. `nutrition-service` 计算杂粮推荐方案。
9. 小程序展示最终推荐结果。

## 架构

第一阶段使用独立 Python HTTP 服务：

```text
微信小程序
  -> nutrition-service
      -> 输入解析器
          -> LLM 解析器
          -> 规则兜底解析器
      -> 当天饮食记录存储
      -> 杂粮推荐引擎
      -> 响应格式化模块

xiaozhi-server
  -> MVP 阶段保持不变
  -> 后续通过 MCP 或 HTTP 工具调用 nutrition-service
```

推荐引擎必须是确定性的、可测试的。LLM 只负责把自然语言转换成结构化数据，不负责最终营养决策。

## 数据模型

### 食物记录

```json
{
  "id": "record_001",
  "user_id": "user_001",
  "date": "2026-07-05",
  "meal": "breakfast",
  "food_name": "baozi",
  "display_name": "包子",
  "amount_text": "2个",
  "estimated_weight_g": 160,
  "category": "refined_staple",
  "confidence": 0.86,
  "source": "llm",
  "base_food": "包子",
  "modifier": ""
}
```

复合主食示例：

```json
{
  "meal": "lunch",
  "food_name": "buckwheat_noodles",
  "display_name": "荞麦面条",
  "amount_text": "1碗",
  "estimated_weight_g": 200,
  "category": "coarse_grain",
  "confidence": 1.0,
  "source": "rule",
  "base_food": "面条",
  "modifier": "荞麦"
}
```

### 用户画像

```json
{
  "user_id": "popkik",
  "age_group": "adult",
  "appetite_level": "normal",
  "digestive_sensitivity": false,
  "health_goal": "balanced"
}
```

字段说明：

- `age_group`: `child`、`adult`、`elderly`。
- `appetite_level`: `light`、`normal`、`large`。
- `digestive_sensitivity`: 是否消化敏感。
- `health_goal`: 第一阶段默认 `balanced`。

### 面团请求

用户请求的重量定义为最终面团总重量：

```text
最终面团总重量 = 粉料重量 + 水重量
```

MVP 固定使用 24% 水比例。这里的水比例定义为：

```text
水比例 = 水重量 / 最终面团总重量
```

```json
{
  "dough_total_weight_g": 500,
  "water_ratio": 0.24,
  "target_food": "dumpling",
  "target_food_display": "饺子",
  "weight_source": "estimated_by_profile_and_food_type"
}
```

当水比例为 24% 时：

```text
水重量 = 面团总重量 * 水比例
粉料重量 = 面团总重量 - 水重量
```

示例：

```text
500 g 最终面团 = 380 g 粉料 + 120 g 水
```

如果用户明确说“500g 面团”，则 `weight_source` 为 `explicit`，优先使用用户输入的重量。  
如果用户只说“晚上想吃饺子”，则根据用户画像和面食种类估算面团重量。

默认成人 normal 食量。这里按“单人一餐面团量”估算，不等同于全天主食摄入量：

```text
面条: 280 g
饺子: 300 g
包子: 320 g
馒头: 300 g
烙饼: 260 g
```

画像系数：

```text
child: 0.6
adult: 1.0
elderly: 0.7
large appetite: 1.3
```

## API 设计

### 解析饮食输入

`POST /api/v1/intake/parse`

请求：

```json
{
  "user_id": "user_001",
  "text": "中午吃了一碗荞麦面条，晚上想吃饺子",
  "profile": {
    "age_group": "adult",
    "appetite_level": "normal",
    "digestive_sensitivity": false
  }
}
```

响应：

```json
{
  "items": [
    {
      "meal": "breakfast",
      "food_name": "buckwheat_noodles",
      "display_name": "荞麦面条",
      "amount_text": "1碗",
      "estimated_weight_g": 200,
      "category": "coarse_grain",
      "confidence": 0.86,
      "source": "llm",
      "base_food": "面条",
      "modifier": "荞麦"
    }
  ],
  "dough_request": {
    "dough_total_weight_g": 500,
    "water_ratio": 0.24,
    "target_food": "dumpling",
    "target_food_display": "饺子",
    "weight_source": "estimated_by_profile_and_food_type"
  },
  "needs_confirmation": true
}
```

### 确认饮食记录并推荐杂粮比例

`POST /api/v1/recommendations/coarse-grain`

请求：

```json
{
  "user_id": "user_001",
  "date": "2026-07-05",
  "confirmed_items": [
    {
      "meal": "breakfast",
      "food_name": "baozi",
      "display_name": "包子",
      "amount_text": "2个",
      "estimated_weight_g": 160,
      "category": "refined_staple"
    },
    {
      "meal": "lunch",
      "food_name": "rice",
      "display_name": "米饭",
      "amount_text": "1碗",
      "estimated_weight_g": 150,
      "category": "refined_staple"
    }
  ],
  "dough_request": {
    "dough_total_weight_g": 500,
    "water_ratio": 0.24
  }
}
```

响应：

```json
{
  "dough_total_weight_g": 500,
  "flour_weight_g": 380,
  "water_weight_g": 120,
  "coarse_grain_ratio": 0.35,
  "coarse_grain_weight_g": 133,
  "reason": "今日精制主食摄入偏多，建议提高本次面团中的杂粮比例。"
}
```

### 查询当天饮食记录

`GET /api/v1/intake/today?user_id=user_001`

响应：

```json
{
  "user_id": "user_001",
  "date": "2026-07-05",
  "items": [
    {
      "meal": "breakfast",
      "display_name": "包子",
      "amount_text": "2个",
      "estimated_weight_g": 160,
      "category": "refined_staple"
    }
  ]
}
```

## 食物分类

MVP 使用一组较小且受控的食物分类：

- `refined_staple`：精制主食，例如白米饭、面条、馒头、包子、面包、饺子皮。
- `coarse_grain`：杂粮和全谷物，例如燕麦、荞麦、玉米粉、全麦粉、小米、杂豆。
- `protein`：主要作为蛋白质来源的食物，例如鸡蛋、肉、鱼、牛奶、豆腐、豆类。
- `vegetable`：蔬菜。
- `fruit`：水果。
- `fat_sugar`：油炸食品、甜点、含糖饮料、高油食物。
- `unknown`：已经识别出来但无法安全分类的食物。

当前规则词库覆盖的常见食物包括：

```text
精制主食：米饭、白米饭、面条、米线、河粉、馒头、包子、面包、饺子、馄饨、年糕
杂粮/薯类：燕麦、燕麦粥、小米粥、玉米、红薯、紫薯、土豆、山药、杂豆、绿豆、红豆
复合杂粮主食：荞麦面条、荞麦面、全麦馒头、全麦面包、全麦吐司、杂粮包子、玉米面条、玉米馒头、黑米饭、糙米饭、杂粮饭
蛋白质：鸡蛋、牛奶、酸奶、豆浆、豆腐、鸡胸肉、鸡肉、牛肉、猪肉、鱼、虾
蔬菜：青菜、白菜、菠菜、生菜、西兰花、胡萝卜、番茄、黄瓜
水果：苹果、香蕉、橙子、梨、葡萄、火龙果
高油高糖：油条、炸鸡、薯条、奶茶、可乐、蛋糕、饼干
```

## 推荐规则

MVP 从简单的通用成人规则开始。系统应该把结果表述为健康饮食建议，而不是医学诊断。

基础值：

```text
base_coarse_grain_ratio = 0.20
min_ratio = 0.10
max_ratio = 0.40
water_ratio = 0.24
```

调整规则：

```text
如果今天精制主食摄入量 >= 300 g:
  ratio += 0.15

如果今天精制主食摄入量在 150-299 g:
  ratio += 0.10

如果今天杂粮摄入量 >= 100 g:
  ratio -= 0.05

如果用户提到胃不舒服、消化不好、儿童、老人或类似消化风险词:
  ratio = min(ratio, 0.20)

将 ratio 限制在 0.10-0.40 之间。
将 ratio 四舍五入到最接近的 5%。
```

重量计算：

```text
water_weight_g = round(dough_total_weight_g * water_ratio)
flour_weight_g = dough_total_weight_g - water_weight_g
coarse_grain_weight_g = round(flour_weight_g * coarse_grain_ratio)
```

示例：

```text
dough_total_weight_g = 500
water_ratio = 0.24
water_weight_g = round(500 * 0.24) = 120
flour_weight_g = 500 - 120 = 380
coarse_grain_ratio = 0.35
coarse_grain_weight_g = round(380 * 0.35) = 133
```

## LLM 解析协议

LLM 解析器必须返回严格 JSON。服务在使用结果前必须先校验 JSON。

当前实现采用“LLM 优先、规则兜底”：

```text
用户输入
  -> LLM 解析为结构化 JSON
  -> 校验 JSON 字段、分类、置信度和重量
  -> 过滤“想吃/想包/准备做”的目标面食，避免计入当天已吃记录
  -> 如果校验失败或 LLM 调用失败，回退到规则解析器
```

启用 CLI LLM 解析：

```powershell
D:\conda\envs\xiaozhi-esp32-server\python.exe nutrition-service\nutrition_cli.py --llm --user-id popkik --text "早饭随便啃了点全麦吐司，中午外面吃了碗牛肉面，晚上想包点饺子"
```

启用 API LLM 解析时，在启动 `uvicorn` 前设置环境变量：

```powershell
$env:NUTRITION_LLM_ENABLED="true"
$env:NUTRITION_LLM_MODEL="deepseek-chat"
$env:NUTRITION_LLM_BASE_URL="https://api.deepseek.com"
```

如果没有设置 `NUTRITION_LLM_API_KEY`，服务会尝试读取 `main/xiaozhi-server/data/.config.yaml` 中的 `LLM.DeepSeekLLM.api_key`。

期望的 LLM 输出：

```json
{
  "items": [
    {
      "meal": "breakfast",
      "display_name": "包子",
      "amount_text": "2个",
      "estimated_weight_g": 160,
      "category": "refined_staple",
      "confidence": 0.86
    }
  ],
  "dough_request": {
    "dough_total_weight_g": 500
  },
  "digestive_risk": false
}
```

校验规则：

- `items` 必须是列表。
- 每个条目必须包含 `display_name`、`amount_text`、`category` 和 `confidence`。
- `category` 必须属于受控分类集合。
- `confidence` 必须在 0 到 1 之间。
- 如果存在 `estimated_weight_g`，它必须是正数。
- 如果存在 `dough_total_weight_g`，它必须在 100 到 3000 之间。

如果校验失败，则使用规则解析器兜底，并将 `needs_confirmation` 设置为 `true`。

## 规则兜底解析器

兜底解析器使用一个小型食物词典：

```text
包子: refined_staple, 每个 80 g
馒头: refined_staple, 每个 100 g
米饭: refined_staple, 每碗 150 g
面条: refined_staple, 每碗 200 g
鸡蛋: protein, 每个 50 g
牛奶: protein, 每杯 250 g
青菜: vegetable, 每份 150 g
苹果: fruit, 每个 200 g
油条: fat_sugar, 每根 70 g
```

MVP 阶段只处理常见中文数量表达：

```text
一个, 两个, 2个, 一碗, 1碗, 一杯, 1杯, 一份, 1份
```

无法识别分类的食物返回为 `unknown`，必须由用户确认。

## 存储

MVP 开发阶段使用 SQLite：

```text
nutrition-service/data/nutrition.db
```

数据表：

```sql
CREATE TABLE food_records (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  record_date TEXT NOT NULL,
  meal TEXT NOT NULL,
  food_name TEXT NOT NULL,
  display_name TEXT NOT NULL,
  amount_text TEXT NOT NULL,
  estimated_weight_g INTEGER,
  category TEXT NOT NULL,
  source TEXT NOT NULL,
  created_at TEXT NOT NULL
);
```

第一阶段只需要支持按 `user_id` 和当天日期查询。

## 错误处理

- 如果 LLM 解析超时，使用规则兜底解析器。
- 如果 LLM 和规则解析器都没有返回食物条目，则让小程序展示手动录入表单。
- 如果缺少面团重量，返回 `needs_dough_weight: true`。
- 如果请求的面团重量不在 100-3000 g 范围内，则拒绝请求，并提示用户输入合理重量。
- 如果存储失败，返回服务端错误，并且不能声称记录已经保存。

## 测试策略

单元测试：

- LLM JSON 校验能够接受合法的结构化输出。
- LLM JSON 校验能够拒绝错误分类、缺失字段和不合理重量。
- 规则兜底解析器能够提取常见食物和数量。
- 当精制主食摄入偏高时，推荐引擎会提高杂粮比例。
- 当存在消化风险输入时，推荐引擎会降低或限制杂粮比例。
- 面团重量计算能够正确把最终面团重量转换为水重量、粉料重量和杂粮重量。

集成测试：

- `POST /api/v1/intake/parse` 能够为真实中文句子返回结构化数据。
- `POST /api/v1/recommendations/coarse-grain` 能够保存确认后的条目并返回推荐结果。
- `GET /api/v1/intake/today` 能够返回同一用户当天保存的记录。

## 后续接入 Xiaozhi

独立服务稳定后，`xiaozhi-server` 可以通过两种方式接入：

1. HTTP 工具：定义一个自定义工具，调用 `nutrition-service` 的接口。
2. MCP 接入点：把 `nutrition-service` 暴露为 MCP 工具服务，并配置 `xiaozhi-server` 调用它。

语音侧可以支持如下请求：

```text
我今天早上吃了两个包子，中午吃了一碗米饭，晚上想和 500 克面团，帮我算一下杂粮比例。
```

语音回复应该复用 `nutrition-service` 返回的确定性推荐结果。

## 实施前待确认事项

- `nutrition-service` 使用哪个 Python Web 框架。建议使用 FastAPI，因为该 MVP 依赖清晰的请求和响应数据结构。
- LLM 解析使用哪个模型提供商。第一版可以复用当前已调通的模型，但服务内部应该通过解析器接口隐藏具体 provider。
- 小程序发送的是语音转写后的文本，还是原始音频。MVP 默认小程序发送文本。
