# 安卓端 App 功能增强计划

## 一、当前状态分析

### 1.1 桌面版 (PyQt6) 已具备的功能
| 功能 | 文件 | 状态 |
|------|------|------|
| 课程数据 CRUD | `ui/course_editor.py` | 完整：添加/删除课程，日期/星期/类型/时间选择 |
| 习惯打卡 | `ui/habit_panel.py` | 完整：打卡、一键全打卡、连续天数、历史记录 |
| 每日复盘 | `ui/review_dialog.py` | 完整：收获/疑问/明日预告 |
| 提醒引擎 | `core/reminder_engine.py` | 完整：30秒轮询，课程提醒+作息提醒+复盘提醒 |
| 设置面板 | `ui/settings_dialog.py` | 完整：提醒、外观、作息、启动、快捷键 |
| 深色/浅色主题 | `core/theme.py` | 完整 |
| 配置管理 | `core/config.py` | JSON文件，含habits配置 |
| 数据库 | `core/database.py` | SQLite：courses, habit_checks, study_logs, reminders, app_meta |
| 开机自启 | `core/auto_start.py` | Windows注册表 |

### 1.2 移动端 (uni-app) 当前状态
| 功能 | 状态 | 缺失内容 |
|------|------|----------|
| 首页 | 有 | 基本功能完整 |
| 习惯打卡 | 有 | 缺少习惯编辑/新增/删除功能 |
| 课程表 | 只读 | 缺少编辑/删除/批量操作 |
| 每日复盘 | 有 | 缺少历史复盘查看 |
| 设置 | 简化 | 缺少习惯管理、数据库管理、数据备份恢复 |
| 提醒引擎 | 有 | 基本功能完整 |
| 数据库 | uni.storage JSON | **与桌面SQLite不兼容**，无法数据互通 |
| 数据管理 | 无 | 完全缺失 |

### 1.3 核心问题
1. **数据格式不兼容**：桌面用SQLite (`study_reminder.db`)，移动端用JSON (`uni.storage`)，无法互通
2. **缺少数据增删改查**：课程只能看不能改，习惯不能编辑，复盘不能查看历史
3. **缺少数据管理**：无备份、恢复、导入、导出功能
4. **设置页功能不完整**：少习惯管理、数据库查看、数据操作入口

---

## 二、改进方案

### 2.1 核心变更：数据库层重写为 plus.sqlite

**文件**: `common/database.js` — 完全重写

**为什么**: 实现与桌面端SQLite数据完全兼容，表结构、字段名一一对应。

**具体变更**:
- 使用 `plus.sqlite` API（已在manifest.json中声明SQLite模块）
- 数据库文件: `_doc/study_reminder.db`（与桌面版同名）
- 表结构完全一致：
  ```sql
  courses (id, date, weekday, title, course_type, start_time, end_time, description, is_rest, is_self_study, created_at)
  habit_checks (id, habit_name, habit_index, check_date, checked, created_at)
  study_logs (id, log_date, gain1, gain2, gain3, question, courses_today, created_at)
  reminders (id, reminder_type, title, trigger_time, status, created_at)
  app_meta (key, value)
  ```
- 所有CRUD方法改为异步（SQLite需要异步操作）：
  - `insertCourse()`, `updateCourse()`, `deleteCourse()`, `getCoursesByDate()`, `getAllCourses()`
  - `checkHabit()`, `getHabitStatus()`, `getStreak()`
  - `saveStudyLog()`, `getStudyLog()`, `getAllStudyLogs()`
  - `getWeekStats()`
  - 新增：`getDbInfo()` — 返回各表行数统计
  - 新增：`exportDb()` — 导出数据库文件
  - 新增：`importDb()` — 导入数据库文件
  - 新增：`clearTable()` — 清空指定表
  - 新增：`batchInsertCourses()` — 批量导入课程

### 2.2 新增页面：课程编辑器

**文件**: `pages/course-editor/course-editor.vue` — 新建

**功能**:
- 课程列表（表格形式）：日期、星期、课程名、类型、时间、操作
- 添加课程：日期选择器、星期选择、课程名输入、类型选择（上课/项目/自习/休息）、时间选择
- 编辑课程：点击课程行弹出编辑表单
- 删除课程：单个删除 + 确认弹窗
- 批量导入：JSON格式粘贴导入（支持从桌面版导出的课程数据）
- 一键清空当前课程数据

**导航**: 从课程表页面的"编辑"按钮进入，或从设置页进入

### 2.3 新增页面：数据管理中心

**文件**: `pages/data-manager/data-manager.vue` — 新建

**功能**:
- **数据库概览**：显示各表行数统计（courses N条, habit_checks N条, study_logs N条）
- **数据库浏览**：按表切换查看所有数据（可折叠展开）
- **数据备份**：
  - 导出数据库文件(.db) — 使用plus.io复制文件到可访问目录
  - 导出为JSON — 所有表数据导出为JSON，可分享/保存
- **数据恢复**：
  - 导入数据库文件(.db) — 从桌面版study_reminder.db直接导入
  - 导入JSON数据 — 从JSON文件批量导入
- **数据清空**：按表清空数据（带二次确认）
- **数据更新**：支持导入已更新好的数据库文件实现数据更新

**导航**: 从设置页"数据管理"入口进入

### 2.4 新增页面：复盘历史

**文件**: `pages/review-history/review-history.vue` — 新建

**功能**:
- 按日期列表展示所有复盘记录
- 点击查看详细内容（收获1/2/3、疑问、课程）
- 支持编辑和删除历史复盘

**导航**: 从复盘页面"历史记录"按钮进入

### 2.5 大幅增强：设置页面

**文件**: `pages/settings/settings.vue` — 重写

**新增/变更区域**:

**提醒设置**（保留并增强）:
- 课程提前提醒（分钟）
- 弹窗提醒开关
- 声音提醒开关
- 晨检点名时间设置
- 起床时间设置

**外观**（保留）:
- 深色/浅色模式切换

**作息时间**（保留）:
- 起床时间、晚自习开始、晚自习结束

**习惯管理**（新增）:
- 习惯列表（可编辑）
- 编辑习惯名称
- 新增习惯
- 删除习惯
- 调整习惯顺序（上移/下移）

**数据管理**（新增）:
- 数据库概览入口 → 跳转数据管理中心
- 导出数据库
- 导入数据库
- 清空课程数据
- 恢复默认课程数据

**关于**（保留）:
- 版本信息

### 2.6 增强：课程表页面

**文件**: `pages/course/course.vue` — 修改

**新增**:
- 顶部添加"编辑课表"按钮 → 跳转课程编辑器
- 每个课程项添加左滑删除或长按删除
- 课程项点击可查看详情/编辑

### 2.7 增强：习惯打卡页面

**文件**: `pages/habit/habit.vue` — 修改

**新增**:
- 顶部添加"习惯管理"按钮 → 跳转设置页习惯管理区域

### 2.8 增强：复盘页面

**文件**: `pages/review/review.vue` — 修改

**新增**:
- 顶部添加"历史记录"按钮 → 跳转复盘历史页面

### 2.9 配置路由更新

**文件**: `pages.json` — 修改

**新增路由**:
- `pages/course-editor/course-editor` — 课程编辑器
- `pages/data-manager/data-manager` — 数据管理中心
- `pages/review-history/review-history` — 复盘历史

### 2.10 配置模块增强

**文件**: `common/config.js` — 修改

**新增方法**:
- `addHabit(name, icon)` — 添加新习惯
- `updateHabit(index, data)` — 更新习惯
- `removeHabit(index)` — 删除习惯
- `reorderHabits(newOrder)` — 调整习惯顺序

### 2.11 提醒引擎适配

**文件**: `common/reminder-engine.js` — 修改

**变更**:
- 适配异步数据库API（所有 `db.xxx()` 调用改为 `await`）
- 增加check_in_time提醒支持

### 2.12 App入口适配

**文件**: `App.vue` — 修改

**变更**:
- 数据库初始化改为异步
- 适配异步db API

---

## 三、新增/修改文件清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `common/database.js` | **重写** | JSON → plus.sqlite，完全兼容桌面版Schema |
| `common/config.js` | **修改** | 新增习惯CRUD方法 |
| `common/reminder-engine.js` | **修改** | 适配异步db API |
| `pages/course-editor/course-editor.vue` | **新建** | 课程CRUD完整页面 |
| `pages/data-manager/data-manager.vue` | **新建** | 数据备份/恢复/导入/导出/浏览 |
| `pages/review-history/review-history.vue` | **新建** | 复盘历史查看/编辑/删除 |
| `pages/settings/settings.vue` | **重写** | 增加习惯管理、数据管理 |
| `pages/course/course.vue` | **修改** | 增加编辑入口、删除功能 |
| `pages/habit/habit.vue` | **修改** | 增加习惯管理入口 |
| `pages/review/review.vue` | **修改** | 增加历史记录入口 |
| `pages/index/index.vue` | **修改** | 适配异步db API |
| `pages.json` | **修改** | 新增3个路由 |
| `App.vue` | **修改** | 适配异步初始化 |
| `manifest.json` | **修改** | 确认SQLite模块声明 |

---

## 四、关键设计决策

1. **plus.sqlite vs uni.storage**: 选择 plus.sqlite，因为需要与桌面版数据完全兼容，方便直接导入 `.db` 文件
2. **数据库文件路径**: `_doc/study_reminder.db`（plus.sqlite默认在应用文档目录），与桌面版同名
3. **异步API**: 所有数据库操作改为async/await，后续页面调用需适配
4. **数据导入导出**: 使用 plus.io 进行文件操作，支持 .db 二进制文件和 .json 文本文件
5. **习惯管理**: 习惯数据从config.js移到settings页面直接管理，保持与桌面版config.json一致

---

## 五、验证步骤

1. 在HBuilder X中打开项目，确认所有页面路由正常
2. 测试数据库初始化：首次启动应自动创建表并导入默认课程数据
3. 测试课程CRUD：添加、编辑、删除课程
4. 测试数据导出：导出 .db 文件，用桌面版SQLite工具验证表结构
5. 测试数据导入：将桌面版 `study_reminder.db` 复制到手机，导入后验证数据一致性
6. 测试习惯管理：新增/编辑/删除习惯
7. 测试复盘历史：查看/编辑/删除历史复盘
8. 测试提醒引擎：确认通知正常触发