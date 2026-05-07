---
name: task-manager-web
description: 单页 HTML 任务管理系统的架构与逻辑：第一部分为层级任务列表（含一级任务日历排期），第二部分为日历时间表；localStorage 持久化；导出/导入与 server.py 同步到当前文件夹的备份逻辑。在扩展或复刻 task-manager-web、或构建类似「纯前端 + 本地存储」的任务/日历界面时使用。
tools: ["Read", "Write", "Bash"]
model: sonnet
effort: low
---
# task-manager-web 逻辑说明

单页、无后端的任务管理，分为**两部分**：**任务列表**（纯文本层级 + 一级任务可挂多条日历排期）与**日历时间表**（按日展示全部日历项，含独立项与一级任务排期）。列表与日历在 `localStorage` 中仍分键存储，日历项可通过 `linkedTaskId` 引用一级任务 `id`。

**维护约定**：本 skill 与项目代码（`task-manager-web/`）同步维护。修改功能或逻辑时**先更新本 skill**，再按 skill 描述改实际代码；这样可通过 skill 把握整体逻辑，并基于 skill 做后续迭代。

---

## 第一部分：任务列表

以文本形式展示层级任务；支持展开/折叠子项、每级状态标记、已完成的可视化选项、页面内直接编辑。**一级任务排期**：每个一级任务有「+ 排期」按钮，在 `taskManagerItems` 中新增带 `linkedTaskId` 的日历项；该行下方展示条列出该任务全部排期（日期 + 起止时间），可点击编辑或「删」快速删除。

### 数据模型（任务列表）

- **存储键**：`taskManagerList`，值为树形结构；`taskManagerListOptions` 存「已完成」展示选项、**类别顺序**等。
- **树节点**（任意层级）：
  - `id`：字符串，唯一
  - `title`：文本
  - `status`：`'unmarked'` | `'in_progress'` | `'completed'`
  - `children`：子节点数组（递归同结构）
- **一级任务**：可有 `category` 字符串（trim 后空则归入展示名「未分类」）。**不再提供行内类别输入框**；类别由「添加类别」或某类别 banner 上「+ 任务」写入。
- **选项**：`completedVisibility`；`categoryOrder`：可选字符串数组，为类别展示名（含「未分类」）的自定义顺序；缺省或无效时按「未分类」优先、其余按中文排序，再与数据合并。

### 行为约定

- **层级**：任务 A 下可有多个子步骤（子任务），子任务可再含子任务；渲染时按层级缩进，可点击展开/折叠该节点下的子项。
- **类别与分组**：工具栏「+ 添加类别」弹出输入新类别名并创建该类别下首条一级任务；每个类别 banner 右侧有 **↑ ↓** 调整 `categoryOrder`、**+ 任务** 向该类别追加一级任务（`category` 存空或对应名）。展示顺序由 *getOrderedCategoryKeys* 结合 `categoryOrder` 与当前分组计算。
- **状态**：每个节点（不论层级）都可标记「未标记」「进行中」「已完成」；**状态栏用不同 emoji 区分**（存储值仍为 `unmarked` / `in_progress` / `completed`）。当某任务设为「已完成」时，其下所有子任务在数据上同步设为已完成（级联写入）。
  - **未标记** 备选：⚪ ○ ◻️ 📋 🔘（当前默认 ○）；**按钮样式**：浅灰色背景，与未选中的中性态区分。
  - **进行中** 备选：🔄 ▶️ 🟡 ⏳ 🔃（当前默认 ⏳）；**按钮样式**：浅琥珀/黄调，表示进行中。
  - **已完成**：✅（固定）；**按钮样式**：浅绿色调，表示已完成。
  - 实现方式：状态下拉带 class `status-unmarked` / `status-in_progress` / `status-completed`，在 CSS 中按类设置背景与边框色。
- **已完成展示**：面板上方提供配置——隐藏已完成，或显示已完成并用横线贯穿（删除线）样式。
- **内联编辑**：所有层级的标题均可在页面上直接编辑（如 contenteditable 或点击切输入框），编辑后写回数据并保存。

### 核心逻辑（任务列表）

- *loadList* / *saveList*：读写 `taskManagerList`；*loadListOptions* / *saveListOptions*：读写 `completedVisibility`、`categoryOrder` 等。
- *setTaskStatus(id, status)*：设该节点 `status`；若 `status === 'completed'`，递归将全部子孙节点设为 `completed`，再 *saveList*、重绘。
- *findNodeById(roots, id)*：在树中查找节点（递归）。
- *buildCategoryGroups* / *getOrderedCategoryKeys* / *defaultCategoryKeyOrder*：分组与排序。
- *addNewCategory* / *addRootInCategory(displayCat)*：新建类别（首条一级任务）或在指定类别下追加一级任务。
- *moveCategory(catKey, delta)*：在有序类别列表中上移/下移，写回 `categoryOrder` 并 *saveListOptions*。
- *renderTaskTree*：按 *getOrderedCategoryKeys* 顺序渲染；每个类别 `tree-group-header` 内含标题、↑↓、+ 任务；banner 为半透明主题色块；类别间无粗横线；一级任务块之间 *tree-sep-root*；*appendSubtree* 逻辑同前；一级行右侧为「+ 排期」「+ 子项」、删除（无类别输入）。
- **主题**：`html[data-theme]` + `localStorage` `taskManagerTheme`；可选 `blue` / `red` / `green` / `yellow` / `purple` / `gray`；*applyTheme*。
- **内容宽度**：左右同屏布局的最大宽度分别由 `split-width-dock--left/right` 控制（滑块 `#splitLeftWidthSlider` / `#splitRightWidthSlider`，写入 `localStorage` 的 `taskManagerSplitLeftWidth` / `taskManagerSplitRightWidth`），以 `--split-left-width` / `--split-right-width` 影响 `.split-layout` 网格。
- *removeTaskFromList(id)*：删除前 `confirm`；一级与次级文案区分（一级提示含子任务与关联排期一并删除）。
- *openTaskEventAdd(taskId)* / *getCalendarEventsForTask(taskId)* / *deleteCalendarItem(id)*：一级任务排期的创建、列举与删除；删除一级任务时同时移除 `linkedTaskId === id` 的日历项。
- 折叠状态：用内存 Set 或对象记录哪些 `id` 被折叠，点击切换，仅影响展示不写存储。
- 内联编辑：标题处 blur/Enter 时取当前文本，更新对应节点 `title`，*saveList*，并 *renderCalendar*（以便关联排期在日历格中的标题与任务名一致）。

---

## 第二部分：日历时间表

日历与任务列表在存储上分键，但日历项可**可选关联**一级任务 `id`（`linkedTaskId`）。

### 数据模型（日历）

- **存储键**：`taskManagerItems`，值为 JSON 数组。
- **单条日历项**：`id`、`title`、`date`（`YYYY-MM-DD`）、`time`（可选，开始时间 `HH:mm`）、`endTime`（可选，结束时间 `HH:mm`）、`note`、`createdAt`（可选）、`linkedTaskId`（可选，必须为某一级任务的 `id`）。
- **一级任务排期**：与独立日历项共用同一数组；`linkedTaskId` 有值时表示该条为某一级任务的执行时段。展示标题用当前一级任务标题（表单中标题只读，保存时写回任务名）。

### 核心逻辑（日历）

- *loadTasks* / *saveTasks*：读写 `taskManagerItems`。
- *getDaysInMonth*：生成当月日历格（含前后月占位）。
- *getTasksForDate(date)*：按 `formatDateKey(date)` 筛选当日项。
- *getCalendarItemDisplayTitle* / *formatTimeRange*：格内与排期条展示用（关联项显示任务名 + 起止时间文案）。
- *renderCalendar*：支持「月视图 / 日视图」切换；月视图按日历格渲染，日视图按 24 小时时间轴网格渲染选中日期当天的所有事项（独立项 + 一级任务排期）。事件条背景与边框随当前主题 `--accent` / `--accent-hover` 等变化；`linkedTaskId` 存在时加 `task-dot-linked`（略深于普通条）。月视图双击格打开添加；日视图双击时间轴网格打开添加；点击事项打开编辑。
- 表单 *openCalendarAdd* / *openTaskEventAdd* / *openCalendarEdit* / *submitCalendarForm*：支持开始/结束时间、关联一级任务（隐藏域 `linkedTaskId`）；编辑时显示「删除」按钮调用 *deleteCalendarItem*。保存后 *renderCalendar* 与 *renderTaskTree* 均执行，保证两侧一致。

---

## 文件与整体布局

```
task-manager-web/
├── index.html   # 顶部 Tab「任务列表」|「日历时间表」；任务列表区（选项栏 + 树容器）+ 日历区（toolbar + 网格）；日历用弹窗表单
├── styles.css   # 任务树缩进、折叠图标、状态标签、删除线、内联编辑；日历格、modal 等
├── app.js       # 两套状态与存储；任务列表的树渲染/展开折叠/状态/内联编辑；日历的渲染与表单
└── README.md
```

- **视图切换**：Tab 切换「任务列表」与「日历时间表」；仅日历视图显示月份工具栏（上一月/下一月/今天）。
- 任务列表与日历使用不同 localStorage 键；日历项可通过 `linkedTaskId` 引用列表中的一级任务，行为上联动展示与删除。

---

## 数据持久化与文件备份

### 存储来源（origin）隔离

- **直接打开** `index.html`（`file://`）与 **http://localhost:8080** 属于不同来源，localStorage 各存各的，**两边数据不同步**。固定用一种方式打开，数据才会持续积累在同一处。
- 编辑内容实际保存在**当前浏览器**为该来源分配的 localStorage 里，不在项目目录的任意文件里（除非使用下方「同步到文件夹」或「导出」）。

### 导出 / 从文件导入

- **导出到文件**：将当前内存中的 `taskManagerList`、`taskManagerListOptions`、`taskManagerItems` 序列化为一个 JSON 对象，用 `getAllData()` 取数，触发浏览器下载为 `task-manager-data.json`。用户可把该文件存到项目文件夹或任意位置，用于备份或换机恢复。
- **从文件导入**：通过 `<input type="file">` 选择本地 `task-manager-data.json`，读入后 `setAllData(data)` 覆盖当前任务列表与日历数据，并写入 localStorage；若当前由带 API 的服务打开，会再调用 *syncToServer* 写回文件夹。
- 任意打开方式下均可使用导出/导入；不换浏览器时不必特意备份，换浏览器或换设备前先导出，在新环境里再「从文件导入」即可恢复。

### 同步到当前文件夹（server.py）

- **server.py**：除提供静态页面外，提供 `GET /api/data` 与 `POST /api/data`，读写当前目录下的 `task-manager-data.json`（与 *getAllData* 结构一致）。
- **加载**：页面初始化时请求 `GET /api/data`；若成功且返回体含 `taskManagerList` 或 `taskManagerItems`，则用 *setAllData(data, true)* 加载并跳过本次同步，否则从 localStorage 加载。
- **保存**：*saveList*、*saveListOptions*、*saveCalendar* 在写 localStorage 后都会调用 *syncToServer*；*syncToServer* 防抖后向 `POST /api/data` 提交 *getAllData()*，从而把编辑内容同步到当前文件夹。直接打开 HTML 时无后端，*syncToServer* 的 fetch 会静默失败，不影响使用。
- **约定**：*setAllData(data, true)* 的第二个参数为 `skipSync`，为 true 时只写 localStorage 不触发 *syncToServer*，用于从服务端刚拉取数据后避免多余回写。

### 文件结构（含备份相关）

```
task-manager-web/
├── index.html
├── styles.css
├── app.js
├── server.py        # 可选：静态服务 + GET/POST /api/data，读写 task-manager-data.json
├── task-manager-data.json   # 由 server.py 或「导出到文件」生成，可放项目内便于备份与检查
└── README.md
```

---

## 扩展约定

- 任务列表：id 不可变；新增层级或字段时在节点对象上扩展，读盘时对缺字段做默认值。`categoryOrder` 与旧数据兼容（缺省则按默认排序）。
- 日历：同上，日期一律 `YYYY-MM-DD`，`time` / `endTime` 可选 `HH:mm`；`linkedTaskId` 仅指向一级任务。
