# CRM 数据埋点 PRD

> 版本：V1.6  
> 日期：2026-08-18  
> 范围：智能创建线索、智能拜访记录、CRM 左侧菜单的数据埋点

## 1. 背景

CRM 已支持智能创建线索和智能拜访记录，需要通过埋点统计入口及按钮使用、创建或提交成功、用户编辑和字段漏解析补全情况。同时需要统计 CRM 左侧各级菜单的点击次数和点击人数，了解菜单的实际使用情况。

本文档仅定义数据埋点及计算口径，不包含模型准确率、看板、报表和业务功能改造需求。


## 2. 埋点统计指标

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
| 菜单点击次数 | 目标菜单点击事件总次数 | 按 `menu_code`、统计周期分别统计，不去重 |
| 菜单点击人数 | 点击目标菜单的 `user_id` 数 | 按 `menu_code`、统计周期对 `user_id` 去重 |

## 3. 智能创建漏斗

点击智能创建 → 开始录音 → 发起智能解析  → 解析成功  → 进入确认编辑页  → 点击提交  → 创建成功

## 4. 埋点公共参数

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

## 5. 埋点事件清单

| 事件名 | 触发时机 | 关键参数 | 适用场景 |
|---|---|---|---|
| `smart_entry_click` | 点击智能创建入口 | `button_name`, `entry_source` | 两者 |
| `record_start_click` | 点击开始录音 | `record_round` | 语音线索；智能拜访 |
| `record_stop_click` | 点击停止录音 | `record_round`, `duration_ms` | 同上 |
| `rerecord_click` | 点击重新录音 | `record_round`, `text_length` | 同上 |
| `parse_submit_click` | 点击下一步或智能生成 | `button_name` | 两者
| `business_product_supplement` | 补充或修改业务板块和产品类型 | `business_line_id`, `product_type_id` | 语言线索 |
| `parse_result` | 解析请求结束 | `result`, `error_code`, `latency_ms`, `field_snapshot_id` | 两者 |
| `confirm_page_view` | 进入确认或编辑页 | `record_temp_id`, `field_count` | 两者 |
| `field_edit` | 字段发生用户修改并失焦或确认 | `field_code`, `original_empty`, `final_empty`, `edit_type` | 两者 |
| `submit_click` | 点击提交按钮 | `button_name`, `validation_result` | 两者 |
| `submit_result` | 返回创建或提交结果 | `result`, `record_id`, `error_code`, `edited_field_count`, `missed_field_count` | 两者 |
| `crm_menu_click` | 用户点击左侧菜单项 | `menu_code`, `menu_name`, `menu_level`, `parent_menu_code`, `menu_path` | CRM 左侧菜单 |

`edit_type` 取值：

- `correction`：修改 AI 已解析出的字段值。
- `missed_extraction_supplement`：AI 初始值为空，用户补全为非空。
- `deletion`：AI 初始值非空，用户清空字段。

## 6. 字段级埋点

### 6.1 智能创建线索字段

| field_code | 字段名称 | 编辑比较规则 |
|---|---|---|
| `company_name` | 公司名称 | 支持用户编辑；比较标准化后的企业标识或文本 |
| `business_product` | 业务板块-产品类型 | 按末级 `option_id` 比较 |
| `detail_requirement` | 详细需求 | 比较标准化后的文本 |
| `lead_source` | 线索来源 | 按 `option_id` 比较 |
| `contact_name` | 客户联系人姓名 | 比较标准化后的文本，不上报字段原值 |
| `contact_phone` | 客户联系人电话 | 去除空格和连接符后比较，不上报字段原值 |

### 6.2 智能拜访记录字段

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

### 6.3 CRM 左侧菜单

仅统计各菜单的点击次数和点击人数。用户点击任一父级菜单或叶子菜单时上报 `crm_menu_click`；父级菜单和叶子菜单按各自的 `menu_code` 分别统计。

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
