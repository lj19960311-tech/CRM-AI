# CRM 数据埋点 PRD

> 版本：V1.3  
> 日期：2026-08-18  
> 范围：智能创建线索、智能拜访记录、CRM 左侧菜单的数据埋点

## 1. 埋点统计指标

| 指标 | 定义/公式 | 去重规则 |
|---|---|---|
| 按钮点击次数 | 目标按钮点击事件总次数 | 不去重 |
| 按钮点击人数 | 触发目标按钮点击事件的 `user_id` 数 | 按统计周期去重 |
| 创建成功数量 | 创建或提交成功的业务记录数 | 按 `record_id` 去重 |
| 创建成功人数 | 创建或提交成功的 `user_id` 数 | 按统计周期去重 |
| 成功率（次数） | 创建成功数量 ÷ 提交点击次数 × 100% | 使用同一场景、同一统计周期 |
| 成功率（人数） | 创建成功人数 ÷ 提交点击人数 × 100% | 使用同一场景、同一统计周期 |
| 编辑数量 | 创建成功记录中，至少一个字段最终值与 AI 初始值不同的记录数 | 按 `record_id` 去重；一条记录编辑多个字段仍计一条 |
| 编辑人数 | 至少产生一条编辑记录的用户数 | 按统计周期对 `user_id` 去重 |
| 编辑率 | 编辑数量 ÷ 创建成功数量 × 100% | 越高表示人工修改越多 |
| 免编辑率 | 未发生编辑的创建成功数量 ÷ 创建成功数量 × 100% | 等于 1 − 编辑率 |
| 字段编辑数量 | 创建成功记录中，该字段最终值与 AI 初始值不同的记录数 | 同一记录同一字段多次修改只计一次 |
| 字段编辑人数 | 编辑过该字段的用户数 | 按字段、统计周期对 `user_id` 去重 |
| 字段编辑率 | 字段编辑数量 ÷ 该字段适用的创建成功记录数 × 100% | 越高表示该字段人工修改越多 |
| 字段免编辑率 | 字段未编辑记录数 ÷ 该字段适用的创建成功记录数 × 100% | 等于 1 − 字段编辑率 |
| 字段漏解析补全数量 | AI 初始值为空、用户补全为非空且成功提交的记录数 | 按 `record_id + field_code` 去重，同时计入字段编辑数量 |
| 字段漏解析补全人数 | 发生字段漏解析补全的用户数 | 按字段、统计周期对 `user_id` 去重 |
| 菜单点击率 | 点击目标菜单的去重人数 ÷ 目标菜单曝光的去重人数 × 100% | 按 `menu_code`、统计周期分别计算 |

## 2. 字段编辑判定规则

- 以解析完成时的 AI 初始字段值与创建或提交成功时的最终字段值进行比较。
- AI 初始值与最终值不同，计为字段编辑。
- AI 初始值为空、用户补全为非空，计为字段编辑和字段漏解析补全。
- AI 初始值非空、用户清空，计为字段编辑。
- 用户修改后又改回 AI 初始值，最终不计字段编辑。
- 公司名称支持编辑，按 AI 初始值与最终提交值比较。
- 条件字段只有在成功记录中适用且展示时才进入字段编辑率分母。
- 比较前统一处理首尾空格和全角/半角；电话去除空格及连接符；枚举按稳定值或 `option_id` 比较；日期时间统一格式后比较。
- 同一成功记录的同一字段只计一次字段编辑，不按输入次数重复累计。

## 3. 埋点公共参数

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `event_id` | string | 是 | 事件唯一标识，用于事件去重 |
| `event_time` | datetime | 是 | 事件发生时间，ISO 8601 |
| `user_id` | string | 是 | 登录用户唯一标识 |
| `session_id` | string | 是 | 单次访问会话标识 |
| `attempt_id` | string | 是 | 一次智能创建或提交尝试标识，贯穿整条链路 |
| `source_attempt_id` | string | 否 | 跨端继续操作时关联原始尝试标识 |
| `scene` | enum | 是 | `voice_lead` / `smart_visit` |
| `platform` | enum | 是 | `mini_program` / `pc` / 其他实际端 |
| `page_name` | string | 是 | 事件发生页面 |
| `entry_source` | string | 否 | 事件入口 |
| `business_line_id` | string | 否 | 业务板块 ID |
| `product_type_id` | string | 否 | 末级产品类型 ID |
| `app_version` | string | 是 | 应用版本 |
| `trace_id` | string | 否 | 前后端链路追踪 ID |
| `is_test_user` | boolean | 是 | 是否为测试、自动化或压测账号 |

## 4. 埋点事件清单

| 事件名 | 触发时机 | 关键参数 | 适用场景 |
|---|---|---|---|
| `smart_entry_click` | 点击智能创建入口 | `button_name`, `entry_source` | 两者 |
| `record_start_click` | 点击开始录音 | `record_round` | 语音线索；智能拜访如有录音则复用 |
| `record_stop_click` | 点击停止录音 | `record_round`, `duration_ms` | 同上 |
| `rerecord_click` | 点击重新录音 | `record_round`, `text_length` | 同上 |
| `parse_submit_click` | 点击下一步或智能生成 | `button_name` | 两者 |
| `parse_result` | 解析请求结束 | `result`, `error_code`, `latency_ms`, `field_snapshot_id` | 两者 |
| `confirm_page_view` | 进入确认或编辑页 | `record_temp_id`, `field_count` | 两者 |
| `field_edit` | 字段发生用户修改并失焦或确认 | `field_code`, `original_empty`, `final_empty`, `edit_type` | 两者 |
| `submit_click` | 点击创建或提交按钮 | `button_name`, `validation_result` | 两者 |
| `submit_result` | 返回创建或提交结果 | `result`, `record_id`, `error_code`, `edited_field_count`, `missed_field_count` | 两者 |
| `crm_menu_exposure` | 菜单项进入左侧菜单可视区域 | `menu_code`, `menu_name`, `menu_level`, `parent_menu_code`, `menu_path` | CRM 左侧菜单 |
| `crm_menu_click` | 用户点击左侧菜单项 | `menu_code`, `menu_name`, `menu_level`, `parent_menu_code`, `menu_path` | CRM 左侧菜单 |

`edit_type` 取值：

- `correction`：修改 AI 已解析出的字段值。
- `missed_extraction_supplement`：AI 初始值为空，用户补全为非空。
- `deletion`：AI 初始值非空，用户清空字段。

## 5. 字段级埋点

### 5.1 智能创建线索字段

| field_code | 字段名称 | 编辑比较规则 |
|---|---|---|
| `company_name` | 公司名称 | 支持用户编辑；比较标准化后的企业标识或文本 |
| `business_product` | 业务板块-产品类型 | 按末级 `option_id` 比较 |
| `detail_requirement` | 详细需求 | 比较标准化后的文本 |
| `lead_source` | 线索来源 | 按 `option_id` 比较 |
| `contact_name` | 客户联系人姓名 | 比较标准化后的文本，不上报字段原值 |
| `contact_phone` | 客户联系人电话 | 去除空格和连接符后比较，不上报字段原值 |

### 5.2 智能拜访记录字段

| field_code | 字段名称 | 编辑比较规则 |
|---|---|---|
| `company_name` | 企业名称 | 比较标准化后的企业标识或文本 |
| `visit_time` | 拜访时间 | 比较统一格式后的日期时间值 |
| `customer_name` | 拜访客户姓名 | 比较标准化后的文本，不上报字段原值 |
| `customer_position` | 拜访人职务 | 比较标准化后的文本，不上报字段原值 |
| `next_visit_time` | 下次拜访时间 | 比较统一格式后的日期时间值 |
| `is_key_project` | 是否是重点项目 | 按布尔值或稳定枚举 ID 比较 |
| `visit_location` | 拜访位置 | 比较标准化后的位置标识或文本 |
| `visit_content` | 拜访内容 | 比较标准化后的文本，不上报字段原值 |
| `follow_up_items` | 需跟进事项 | 比较标准化后的文本，不上报字段原值 |

上述字段均需统计字段编辑数量、字段编辑人数、字段编辑率、字段免编辑率、字段漏解析补全数量和字段漏解析补全人数。

### 5.3 CRM 左侧菜单

仅统计各菜单的点击率：

> 菜单点击率 = 点击该菜单的去重人数 ÷ 该菜单曝光的去重人数 × 100%

菜单展开后，子菜单首次进入可视区域时上报 `crm_menu_exposure`；同一用户在同一统计周期内对同一 `menu_code` 的重复曝光和重复点击分别去重。父级菜单和叶子菜单均单独统计。

| menu_path | menu_code |
|---|---|
| 线索管理 | `lead_management` |
| 线索管理 / 线索管理 | `lead_management_group` |
| 线索管理 / 线索管理 / 企业线索 | `enterprise_lead` |
| 线索管理 / 线索管理 / 个人线索 | `personal_lead` |
| 线索管理 / 线索管理 / 拜访地图 | `visit_map` |
| 线索管理 / 线索管理 / 线索记录 | `lead_record` |
| 线索管理 / 线索管理 / 线索查询 | `lead_search` |
| 营销工具 | `marketing_tools` |
| 营销工具 / 招募明细 | `recruitment_details` |
| 营销工具 / 招募明细 / 调度招募明细 | `dispatcher_recruitment_details` |
| 营销工具 / 招募明细 / 司机招募明细 | `driver_recruitment_details` |
| 营销工具 / 招募明细 / 货主招募明细 | `shipper_recruitment_details` |
| 营销工具 / 招募明细 / 货主拉新排名 | `shipper_acquisition_ranking` |
| 企业档案 | `enterprise_profile` |
| 企业档案 / 企业档案 | `enterprise_profile_group` |
| 企业档案 / 企业档案 / 企业档案 | `enterprise_profile_page` |
| 生态营销产品 | `eco_marketing_products` |
| 生态营销产品 / 线索任务管理 | `lead_task_management` |
| 生态营销产品 / 线索任务管理 / 线索任务广场 | `lead_task_market` |
| 生态营销产品 / 线索任务管理 / 已领取任务 | `claimed_tasks` |
| 智选领域 | `smart_selection_domain` |
| 智选领域 / 智选领域 | `smart_selection_domain_page` |
| 统计分析 | `statistical_analysis` |
| 统计分析 / 统计看板 | `statistics_board` |
| 统计分析 / 统计看板 / 园区地推报表 | `park_promotion_report` |
| 统计分析 / 统计看板 / 拜访情况 | `visit_status` |
| 统计分析 / 统计看板 / 销售激活报表 | `sales_activation_report` |
| 金融业务 | `financial_business` |
| 金融业务 / 金融业务 | `financial_business_group` |
| 金融业务 / 金融业务 / 票据逾期查询 | `bill_overdue_search` |
| 金融业务 / 金融业务 / 报单管理 | `application_management` |
| 合同管理 | `contract_management` |
| 合同管理 / 合同管理 | `contract_management_group` |
| 合同管理 / 合同管理 / 合同申请 | `contract_application` |

## 6. 埋点数据规则

- 统计时区统一为 `Asia/Shanghai`，自然日为 00:00:00—23:59:59。
- 事件按 `event_id` 去重。
- 成功记录按 `record_id` 去重。
- 人数按统计周期对 `user_id` 去重。
- 菜单曝光人数和菜单点击人数按 `menu_code + user_id` 在统计周期内去重。
- 菜单未进入可视区域时不得计为曝光；父级菜单展开不等于所有子菜单均已曝光。
- 同一 `attempt_id` 多次点击提交时，点击次数如实累计，成功记录仅按 `record_id` 计一次。
- 跨端继续操作时应保持同一 `attempt_id`；无法保持时记录 `source_attempt_id`。
- AI 初始值为空、用户补全为非空并成功提交时，必须标记为 `missed_extraction_supplement`。
- 测试账号、自动化账号和内部压测流量通过 `is_test_user` 排除。
- 禁止在埋点中上报语音音频、识别全文、联系人姓名、手机号、详细需求全文、拜访内容全文和需跟进事项全文。
