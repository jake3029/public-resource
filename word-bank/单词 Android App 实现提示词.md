# 背单词 Android App 实现提示词（定稿 v1.0）

你是一名资深 Android 客户端工程师 + 教育算法产品经理。请设计并实现一个完整可运行的「背单词 Android App」。本提示词为定稿版，所有产品决策已确认，不得自行更改；若发现不可行，必须先停下提问，不得静默假设。

---

## 0. 交付形态（硬性）

- 必须输出**完整可运行 Android 项目代码**，不是方案或片段。
- 按阶段交付，每阶段给出：新增/修改文件清单、完整文件内容、验证步骤。每阶段结束项目必须可编译运行。
- 最终提供：README、示例 CSV、示例 .xlsx 模板说明、助记/词根词缀编辑说明、算法单元测试、构建运行说明、许可证清单、隐私说明、已知限制。
- 面向中文用户，界面中文。
- 不虚构库、API、算法参数、版本号、词根词缀数据来源。

---

## 1. 核心技术决策（已定稿）

1. **调度算法**：采用 FSRS-6，不使用 SM-2，不使用固定 1/3/7/15 天作为唯一策略；不得混用 FSRS-5 与 FSRS-6 公式。算法版本号必须写入代码与文档。
2. **学习方式**：只做卡片记忆。正面单词可显示音标；背面释义、例句、助记。显示答案后四档评分：忘记、模糊、认识、熟练。
3. **词库导入**：必须支持 CSV 与 .xlsx（仅 .xlsx，不支持 .xls）。
4. **导入模板字段固定**：`word, meaning, phonetic, example, tags, mnemonic`，CSV 与 .xlsx 同一套字段，`mnemonic` 可为空。
5. **助记**：内置少量词根词缀种子数据；用户可增改停用内置、可新增自定义；支持导入 mnemonic；用户助记优先级高于内置。
6. **助记优先级**：用户助记 > 内置词根词缀匹配 > 不显示。内置多条匹配按匹配长度和类型排序，避免误匹配。
7. **发音**：Android 系统 TTS 朗读单词，不要求音频文件。
8. **账号与云同步**：MVP 不需要，默认本地离线。
9. **统计与可视化**：MVP 必须保留统计数字，且必须包含遗忘曲线/回忆概率曲线图。
10. **隐私**：默认不上传词库、助记、学习记录；禁用系统自动云备份。

---

## 2. 已定稿的 15 项口径（必须遵守）

### 2.1 FSRS-6 基准
- 以 open-spaced-repetition 官方组织下 **py-fsrs 的 FSRS-6 实现**为公式与默认权重基准。
- 方案阶段查证并锁定具体 release tag/commit、默认权重数组、decay/factor 常量、许可证；写入 README 与代码常量（`Fsrs6Params.kt`）。
- Android 端**纯 Kotlin 移植**，不引入 Python 运行时。
- 若无法查证，必须停下向用户确认，**不得编造权重或版本号**。

### 2.2 评分与状态机
- 忘记/模糊/认识/熟练 → Again/Hard/Good/Easy。
- 状态转换、学习步骤、Again 后进入 Learning 还是 Relearning、New 首评后的目标状态，**完全照搬 py-fsrs FSRS-6 参考实现**，不自定义。
- 单元测试间隔单调性断言：**Again ≤ Hard ≤ Good ≤ Easy**，允许相等和浮点容差，不强制严格递增。
- MVP **不做撤销**；评分后按钮立即禁用并进入下一张；卡片状态更新与复习日志插入必须在**同一事务**内完成。

### 2.3 时间口径
- 数据库统一存 **UTC epoch millis**，界面按设备本地时区展示。
- 每日边界按**本地时区 0 点**统计今日学习/复习/连续天数。
- `elapsedDays`、`scheduledDays` 计算**严格照搬 py-fsrs 参考实现**，不改 24 小时制或自然日制。
- 今日复习队列只取 `due <= 当前时刻` 的 Review/Relearning 卡，不提前纳入当天晚些到期的卡。
- 算法模块注入 `Clock`，单元测试用固定时钟。
- 跨时区、DST、手动改系统时间：MVP 不做特殊补偿，按注入时钟和本地日期边界自然处理。

### 2.4 今日队列与上限
- 队列分两段：**先做到期复习，再做新词**；复习按 `due` 升序，新词按创建时间升序。
- 每日新词数、每日复习上限独立计算；达到上限后**允许继续学，但提示“已超额”**，不硬拦截。
- 新词首次学习计入“今日学习数量”，不计入“今日复习数量”。
- 到期复习超过上限，超出部分**顺延到明天**，不丢失。
- 逾期多天按 `due` 升序优先。
- 学习会话中断后**重新按规则生成队列**，不做原队列恢复。
- MVP **不做跳过、暂停、归档、标记已掌握**。

### 2.5 错词本
- 一次评分 **Again** 即自动进入错词本。
- 连续 **2 次 Good/Easy** 自动移出；也支持手动移出。
- 错词专项复习**正常写复习日志、正常更新 FSRS 状态**。
- 错词本**不影响**每日队列生成，只作为独立筛选视图和专项复习入口。
- 错词专项复习**不受**每日新词/复习上限限制。

### 2.6 导入去重与冲突
- 去重键：`word` 经 **trim + 转小写**后唯一；同词不同释义视为重复。
- 已存在单词：**整行跳过**，导入报告标记“已存在跳过”；MVP 不提供覆盖/合并，避免误覆盖 FSRS 状态和用户助记。
- 导入 `mnemonic` 仅在新建单词时写入。
- `tags` 多标签用**英文分号 `;`** 分隔，也兼容竖线 `|`，trim 后入库。
- `word` 或 `meaning` 为空：整行跳过，计入失败报告并给出行号。
- 同一文件重复导入幂等，不产生重复数据。
- CSV 支持 UTF-8 和带 BOM 的 UTF-8；GBK 不保证，导入页提示“请另存为 UTF-8”。

### 2.7 词根词缀匹配
- 大小写不敏感，按“词根/词缀文本包含于单词”匹配。
- 最小匹配长度 **3 个字符**，低于 3 字符不参与自动匹配。
- 不做词形变化还原，不做词边界判断。
- 排序：**匹配长度降序 > 类型权重 词根 > 前缀 > 后缀 > 字典序**。
- 内置匹配最多展示 **3 条**；用户助记优先展示，内置作补充。
- 用户修改内置数据：**复制为用户自定义记录并停用原内置记录**，不改内置种子表。
- 停用记录不参与匹配。
- 设置关闭助记展示：用户助记和内置助记**全部隐藏**。
- 卡片背面折叠/展开状态**不持久化**，默认展开。

### 2.8 数据来源
- 内置词根词缀种子数据和内置示例词库**均为项目自编示例数据**，不引用第三方词根词缀库或词典。
- README 标注：“内置词根词缀与示例词库为项目自编示例，仅供功能演示，许可证随项目。”
- 不虚构外部来源。

### 2.9 .xlsx 解析
- 使用 **Apache POI `poi-ooxml`** 解析 .xlsx，仅支持 .xlsx。
- 轻量模式读取，只解析首个工作表、前 6 列固定字段；合并单元格/公式/日期做基本处理。
- 配置 ProGuard/R8 规则；README 记录 POI 版本、许可证、包大小影响。
- 文件大小上限 **5 MB**，行数上限 **5000 行**。
- 若 Android 17 实测 POI 不可用，换轻量 xlsx 库，但必须在方案阶段实测后锁定并说明。

### 2.10 本地提醒
- 使用 **WorkManager** 周期任务，接受**非精确提醒**；README 和设置页注明“受系统省电策略影响，可能延迟”。
- 设置页只提供“提醒开关 + 提醒时间”；改时间取消旧任务并重新入队。
- Android 13+ 通知权限被拒绝：开关置灰并提示去系统设置开启，不崩溃。
- 独立通知渠道；点击通知打开今日队列页。
- 通知内容**不显示单词和释义**，只显示“该复习了”类文案。
- 设备重启、时区变化、DST：MVP 不做额外补偿，README 注明已知限制。
- 提醒任务与今日队列生成**解耦**。

### 2.11 统计与曲线
- “今日学习数量” = 今天首次评分的 New 卡数量；“今日复习数量” = 今天对非 New 卡评分次数；均按单词去重。
- 连续天数：当天至少 1 次评分即算打卡，按本地 0 点切日。
- “待复习数量” = `due <= now` 且 state 为 Review/Relearning，不含 New。
- **单词详情页展示单卡回忆概率曲线**；**统计页展示当前待复习队列的平均回忆概率曲线**（卡片等权平均），横轴未来 30 天、每天 1 个采样点。
- 实际评分趋势：按天聚合，纵轴为当天平均评分（Again=1、Hard=2、Good=3、Easy=4），与预测曲线**双轴叠加**，图例标明。
- 空数据显示空状态文案，不画假曲线。
- 曲线只依赖 FSRS-6 stability 和可提取性公式，不引入装饰性数据。

### 2.12 数据备份与删除
- 禁用 Android 自动云备份：`allowBackup=false`，配置 `dataExtractionRules`。
- MVP **不提供**全量导出/导入、云同步、跨设备迁移。
- 删除单词：**级联删除**复习日志、错词本记录、用户助记；内置词根词缀不删。
- 提供“清空全部学习数据”入口，二次确认。
- Room 开启 schema 导出，提供基础迁移策略，保证版本升级不丢数据。
- 评分更新卡片状态与插入复习日志必须在**同一事务**内完成。
- 索引：`Word.normalizedWord` 唯一、`FsrsCard.due`、`FsrsCard.state`、`ReviewLog(wordId, reviewTime)`。
- 复习日志**不可变**，不提供编辑。

### 2.13 TTS
- 使用 Android 系统 `TextToSpeech`，语言 **English (US)**。
- 未安装英文语音包：提示用户去系统 TTS 设置安装，不静默失败。
- TTS 实例页面级或 Application 级单例，页面销毁时释放，不每次点击新建。
- 发音按钮防抖；失败 Toast 提示；不自动发音，仅点击发音。
- 音标只展示不朗读；语速、口音 MVP 不设。
- 不申请音频焦点，不处理与其他音频 App 抢占；README 注明已知限制。

### 2.14 UI 与 MVP 非目标
交互边界：
- 首次启动幂等初始化种子数据（内置示例词库、少量词根词缀），只初始化一次。
- 空词库、空队列、空图表均显示中文空状态，不画假数据。
- 卡片显示答案后出现四档评分，评分后自动下一张，评分按钮点击后禁用防重复。
- 屏幕旋转/进程重建：恢复当前卡片和是否已显示答案；队列重新按规则生成。
- 单词编辑：编辑六字段不重置 FSRS 状态；改 word 需重新去重校验。
- 词库页支持搜索、按 tags 筛选、按创建时间排序、单条删除；MVP 不做批量删除、分页。
- `tags` 仅用于筛选和展示，不参与调度。
- 深色模式跟随系统；大字体基本适配；无障碍只保证可点击区域和内容描述。
- 导航：底部导航为 学习 / 词库 / 统计 / 设置；错词本从词库或学习页进入。

MVP 非目标：
- 无账号、云同步、跨设备迁移、社交、排行榜。
- 无 AI 生成助记、无在线词典、无词库市场。
- 无 FSRS 参数在线训练/个性化拟合。
- 无 .xls 支持、无全量数据导出、无复习撤销、无跳过/暂停/归档/标记已掌握。
- 无音频文件发音、无发音口音/语速设置、无精确闹钟提醒。

### 2.15 工程交付
- **不用 Hilt**，采用手动依赖注入（`AppContainer` + ViewModel Factory）。
- **minSdk 26**；compileSdk 和 targetSdk 锁定 **Android 17 对应 API 级别**；方案阶段查证官方 SDK 与稳定 AGP/Gradle/Kotlin/Compose/Room 版本后写入 README，**不虚构版本号**。
- JDK 17；若 Android 17 工具链要求更高 JDK，以官方要求为准并注明。
- 必须在 **Android 17 模拟器或真机**完成冒烟验证。
- 适配 targetSdk 提升带来的行为变更：edge-to-edge、通知权限、后台/提醒限制、SAF、TTS 生命周期。
- 算法模块为**纯 Kotlin 模块，无 Android 依赖**，注入 `Clock`，可独立跑 JUnit。
- 测试范围：算法 JUnit 单元测试；导入解析单元测试；Room DAO 仪器测试或 Robolectric；WorkManager 测试入队；TTS 用接口 mock。
- Release 开启 R8，配置 POI/Room 混淆规则，并实际构建验证。
- 包名 **`com.gj.word`**。
- 分 5 阶段交付，每阶段可编译运行并给验证步骤。
- 示例 CSV 放 `samples/words_sample.csv`；示例 .xlsx 模板放 `samples/words_template.xlsx`；README 说明字段和导入方式；MVP 不在 App 内生成模板。
- README 包含：构建运行说明、FSRS-6 来源与权重、许可证清单、数据隐私说明、已知限制。

---

## 3. 产品目标

1. 本地优先、离线可用。
2. 支持词库管理、新词学习、按 FSRS-6 自动生成今日复习队列。
3. 每次复习后根据四档评分更新 FSRS-6 状态并计算下次复习时间。
4. 提供错词本、每日目标、本地提醒、学习统计、遗忘曲线/回忆概率可视化、助记展示与编辑。
5. MVP 先保证可运行、可验证，代码结构为后续增强留扩展点。

---

## 4. 核心功能

- **词库管理**：内置示例词库；手动添加单词；CSV 导入；.xlsx 导入；字段固定 `word, meaning, phonetic, example, tags, mnemonic`。
- **导入**：SAF 选择文件；导入预览；固定字段模板；去重策略；成功/失败报告；示例 CSV 和 .xlsx 模板说明。
- **助记管理**：内置少量词根词缀种子（含词根/词缀、含义、示例词，项目自编）；用户可增改停用内置、新增自定义；支持 mnemonic 列写入用户助记；单词详情/编辑页可单独编辑用户助记；匹配规则按 2.7。
- **学习模式**：单词卡片；显示/隐藏释义；系统 TTS 发音；显示答案后四档评分；背面展示助记。
- **复习模式**：FSRS-6 生成今日复习队列；新词与到期复习分开统计。
- **错词本**：低评分自动进入，可专项复习。
- **每日目标**：可设每日新词数、每日复习上限。
- **提醒**：WorkManager 本地通知，可设提醒时间。
- **统计**：今日学习/复习数量、连续天数、待复习数量、FSRS 预测回忆概率曲线、实际评分趋势。
- **遗忘曲线图**：横轴时间，纵轴预测回忆概率；曲线必须真实来自 FSRS-6 状态与 stability，不得使用装饰性假数据；可叠加实际复习评分趋势。
- **持久化**：Room；设置用 DataStore；重启不丢数据。
- **隐私**：默认不上传词库、助记内容和学习记录。

---

## 5. FSRS-6 算法要求

1. 独立 FSRS-6 调度模块，不把算法散落在 UI 或 ViewModel。
2. 卡片状态至少包含：`state`（New/Learning/Review/Relearning）、`stability`、`difficulty`、`due`、`lastReview`、`reps`、`lapses`、`scheduledDays`、`elapsedDays`。
3. 复习日志至少包含：单词 ID、复习时间、四档评分、复习前状态、复习后状态、`elapsedDays`、`scheduledDays`、预测可提取性、下次到期时间。
4. 四档评分映射 FSRS-6 的 Again/Hard/Good/Easy。
5. 使用公开的 FSRS-6 公式与默认权重；默认参数集中配置，允许重置；不得虚构算法、参数或库。
6. 提供算法单元测试，至少验证：
   - 新词首次评分后进入正确状态；
   - 四档评分下次间隔满足 Again ≤ Hard ≤ Good ≤ Easy（允许相等/容差）；
   - 到期时间晚于当前时间；
   - 复习日志与卡片状态一致；
   - 预测回忆概率曲线可计算；
   - 遗忘曲线图数据来自真实 FSRS-6 状态。
7. 不得用固定间隔替代 FSRS-6。

---

## 6. 技术约束

- 原生 Android：Kotlin + Jetpack Compose。
- 架构：MVVM + Repository，模块边界清晰。
- 本地数据库 Room；设置 DataStore；后台提醒 WorkManager。
- 文件导入 SAF 选择 CSV、.xlsx。
- TTS 使用 Android 系统 `TextToSpeech`。
- CSV、.xlsx 解析必须真实可用；.xlsx 用 Apache POI 并处理 Android 兼容性、包大小、混淆。
- 不用 Hilt，手动 `AppContainer` + ViewModel Factory。
- 代码可运行、结构清晰，避免过度设计。

---

## 7. Room 数据模型

```text
Word
  id PK, word, normalizedWord(unique), meaning, phonetic,
  example, tags, userMnemonic, createdAt, updatedAt

FsrsCard
  wordId PK/FK->Word, state, stability, difficulty,
  due(epochMillis), lastReview, reps, lapses,
  scheduledDays, elapsedDays, isInWrongBook, wrongBookGoodStreak, updatedAt

ReviewLog（不可变）
  id PK, wordId, reviewTime, rating, stateBefore, stateAfter,
  stabilityBefore, stabilityAfter, difficultyBefore, difficultyAfter,
  elapsedDays, scheduledDays, retrievabilityBefore, dueAfter

Morpheme（词根词缀，内置+用户统一表）
  id PK, text, type(PREFIX/ROOT/SUFFIX), meaning, examples,
  isBuiltin, isEnabled, sourceNote, createdAt, updatedAt

设置走 DataStore：每日新词数、复习上限、助记开关、提醒开关/时间、FSRS 参数重置标记
```

删除 Word 级联删除 FsrsCard、ReviewLog。

---

## 8. 目录结构（建议）

```text
app/
  src/main/java/com/gj/word/
    di/AppContainer.kt
    data/local/{entity,dao,AppDatabase,Converters}.kt
    data/repo/{WordRepository,ReviewRepository,MorphemeRepository,SettingsRepository}.kt
    data/importer/{CsvImporter,XlsxImporter,ImportMapper,ImportReport}.kt
    data/tts/TtsManager.kt
    data/reminder/{ReminderWorker,ReminderScheduler}.kt
    domain/fsrs/  (或独立 :core-fsrs 模块)
      Fsrs6.kt, Fsrs6Params.kt, FsrsModels.kt
    ui/{learn,library,import,stats,settings,morpheme,wrongbook}/...
    ui/theme/, ui/nav/
  src/test/java/...            // Fsrs6 JUnit
  src/androidTest/java/...     // DAO / WorkManager
samples/words_sample.csv
samples/words_template.xlsx
README.md
```

---

## 9. 验收标准

**算法**
- New 首评后状态正确（照搬参考实现）。
- 下次间隔 Again ≤ Hard ≤ Good ≤ Easy（允许相等/容差）。
- 所有评分后 `due > now`。
- 日志字段与卡状态一致（事务）。
- 单卡与队列平均回忆概率曲线可计算，数据来自真实 stability。

**功能**
- CSV 和 .xlsx 均能导入固定六字段，预览、去重、失败报告正确，重复导入幂等。
- 卡片正反面、四档评分、TTS、助记优先级与折叠、设置关闭助记生效。
- 今日队列先复习后新词、新词/复习分开统计、超额提示不拦截。
- 错词本进入/移出规则正确，专项复习写日志。
- 统计数字与曲线正确，空数据空状态，无假曲线。
- 提醒开关+时间可设，通知不含单词/释义。
- 重启不丢数据；`allowBackup=false`；清空数据二次确认。
- 旋转/进程重建恢复当前卡与显示状态。

**工程**
- `./gradlew assembleDebug` 与 `testDebugUnitTest` 通过；R8 release 可构建。
- Android 17 模拟器/真机冒烟：冷启动、导入、学习、复习、统计、提醒权限流程无崩溃。
- README 含构建运行、FSRS-6 来源与权重、许可证清单、隐私说明、已知限制、示例模板说明。

---

## 10. 阶段划分

| 阶段 | 交付 | 验证 |
|---|---|---|
| 1 | 工程骨架、主题、导航、Room 实体/DAO/DB、DataStore、AppContainer、种子数据、设置页雏形 | 编译运行、DB 迁移测试、种子幂等 |
| 2 | `:core-fsrs` 纯 Kotlin 模块 + 参数常量 + 单元测试 | 算法验收 6 条 |
| 3 | 词库管理、手动增删改、CSV/.xlsx 导入、预览与报告、模板文件 | 导入验收、幂等、失败报告 |
| 4 | 学习/复习队列、卡片、四档评分、事务写日志、TTS | 队列与评分验收 |
| 5 | 助记匹配与管理、错词本、统计与曲线、提醒、清空数据、README | 全量验收 + Android 17 冒烟 |

---

## 11. 开工前锁定项（必须先处理）

- **锁定项 A**：py-fsrs 的 release tag/commit、`DEFAULT_WEIGHTS` 数组、decay/factor 常量、许可证。未锁定前不得编造数值，必须停下向用户确认来源。
- **锁定项 B**：Android 17 对应 compileSdk/targetSdk API 级别，以及能稳定支持该 API 的 AGP/Gradle/Kotlin/Compose/Room 版本。若当前工具链尚不支持 Android 17，先用可用最高稳定 API 起工程并在 README 标注升级路径，不虚构版本号，并在阶段 1 开工前向用户确认是否接受过渡方案。

---

## 12. 输出要求

1. 先输出最终实现方案摘要：MVP 范围、非目标、页面清单、用户流程、Room 数据模型、FSRS-6 调度伪代码、助记匹配与导入方案、技术架构、目录结构、依赖清单、验收标准。
2. 然后按阶段输出完整可运行 Android 项目代码，直到覆盖全部 MVP 功能。不要只给关键片段。
3. 每阶段给出：新增/修改文件清单、完整文件内容、验证步骤。
4. 最终提供：README、示例 CSV、示例 .xlsx 模板说明、助记/词根词缀编辑说明、算法单元测试、构建运行说明。
5. 如果需求有歧义，先提问确认，不要自行假设关键产品决策。