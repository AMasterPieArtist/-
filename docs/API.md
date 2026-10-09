# 本地 API

同源路径 `/api`，JSON 请求和响应。仅绑定本机，不开放 CORS。浏览器从 Python 服务打开页面，不能直接使用 `file://`。

写请求需 `Content-Type: application/json`；上限32KB。错误结构：

```json
{"error":{"code":"conflict","message":"计划已变化，请刷新后重试。"}}
```

| 接口 | 方法 | 主要输入 / 返回 |
| --- | --- | --- |
| `/api/status` | GET | 可用性、版本、是否检测到Key，不返回Key |
| `/api/profile` | GET | `{profile:{data,version,updated_at}}` 或 null |
| `/api/profile` | PUT | `{profile:{…},expected_version:0}`，首次版本0 |
| `/api/plans?date=YYYY-MM-DD` | GET | `{plan:…}` 或 null |
| `/api/plans/generate` | POST | `{date,mode:"demo"或"deepseek",expected_version:0}` |
| `/api/meals/select` | POST | `{date,expected_version,meal,option_id}` |
| `/api/meals/refresh` | POST | `{date,expected_version,meal,reason:"refresh"或"unavailable"或"dislike"}` |
| `/api/meals/complete` | POST | `{date,expected_version,meal,completed:true或false}` |
| `/api/adjustments?date=YYYY-MM-DD` | GET | 当日草案和历史快照 |
| `/api/adjustments` | POST | `{date,expected_version,mode,request}`，返回草案，`plan_changed:false` |
| `/api/adjustments/{id}/confirm` | POST | `{}`，事务内检查版本并更新 |
| `/api/adjustments/{id}/discard` | POST | `{}`，保留正式计划 |
| `/api/strategy/draft` | POST | `{date,expected_version,focus}` 或 `request`，策略模板草案 |
| `/api/snacks` | GET | 小食目录，未知营养为null |
| `/api/snacks/add` | POST | `{date,expected_version,snack_id}` |
| `/api/snacks/remove` | POST | `{date,expected_version,entry_id}` |
| `/api/activities/add` | POST | `{date,expected_version,kind:"walk"或"swim",minutes}` |
| `/api/activities/remove` | POST | `{date,expected_version,entry_id}` |
| `/api/activities/rest` | POST | `{date,expected_version}`，明确休息并清空当日运动 |
| `/api/foods?q=米饭&category=主食` | GET | 基础食品、份量基准、参考营养与来源 |
| `/api/feedback` | POST | `{date,expected_version,score:1至5,comment:"可选"}` |
| `/api/export` | GET | 全部画像、计划、调整、事件；包含个人信息 |
| `/api/data/delete` | POST | `{confirmation:"删除我的本地数据"}`，保留食品库 |

`meal` 为 `breakfast / lunch / snack / dinner`。`focus` 为 `饮食优先 / 运动优先 / 作息优先 / 平衡维持 / 自动匹配`。

所有日期使用浏览器本地日期传入，不由 UTC 自动替换。UI内默认今天，可切换查询已有日期。

## 画像示例

```json
{
  "expected_version":0,
  "profile":{
    "nickname":"小满","age":28,"sex":"female",
    "height_cm":165,"current_weight_kg":60,"target_weight_kg":57,
    "target_weeks":12,"goal":"规律饮食",
    "work_intensity":"normal","meal_sources":["canteen","delivery"],
    "exercise_frequency":"rare","available_time":"10–20 分钟",
    "sleep":"偶尔晚睡","preferences":"清淡家常菜",
    "exclusions":[],"exclusions_note":"","main_difficulty":"吃饭不规律",
    "health_status":"general","ai_consent":false
  }
}
```

枚举：生理性别 `female/male`；工作 `normal/busy`；饮食 `canteen/delivery/home`；运动 `rare/some/often`；健康 `general/pregnant/condition/eating_concern`。其他可选字段的枚举见 `core.profile_value`。

## 计划语义

`meals[meal].selected` 是已采用方案，`candidates` 是当前待选批次。刷新候选不改变 selected。`completed` 只表示用户标记完成，不代表称重后的实际摄入。

`totals` 和 `checks` 描述基础四餐。`snacks` 是额外主动选择；UI加总已知营养、标注未知部分。不能把四餐的检查结论当作额外小食也通过了规则。`activities` 的估算单独显示。

普通修改需要当前 `expected_version`。版本不符返回409。画像改变也会要求重新生成。生成前有完成餐次时，必须经用户确认后传 `reset_completed:true`；UI重新生成还会确认重置小食和运动。

草案状态：`pending / confirmed / discarded / superseded`。确认使用 `BEGIN IMMEDIATE`，匹配正式计划版本和画像版本，确认成功后使同一计划其他旧草案过期。客户端不能直接提交任意候选JSON，只能选择服务端保存的候选ID。

## DeepSeek

Key 从进程环境或项目 `.env` 读取，前端不接收。仅授权后发送最小必要上下文，固定 HTTPS 目的地。模拟测试覆盖成功、无效JSON、未知候选、鉴权、配额、超时和风险拒绝，不代表真实账号调用已验证。
