# spring-boot-migrator-plus – APM Implementation Plan
**Memory Strategy:** Dynamic-MD
**Last Modification:** Plan creation by the Setup Agent.
**Project Overview:** 基于 spring-boot-migrator 开发的通用多版本 Spring Boot 升级工具（Java 17 + Maven 3.6+）。支持 Spring Boot 1.x/2.x/3.x 之间的任意版本升级（如 1.5.x→2.7.x、2.x→3.x、1.x→3.x 跨版本自动分步升级）。核心功能：自动版本检测、交互式目标版本选择、增强的 jar 依赖分析（支持 shaded 包递归解析）、智能兼容性检测（内置数据+在线查询）、YAML 规则驱动的配置迁移（工具开发者+用户可自定义）、完整备份回滚机制、详细 Markdown 迁移报告。基于 OpenRewrite 实现代码迁移，完全保留 spring-boot-migrator 原有功能与向后兼容性。全平台支持（macOS/Linux/Windows），40% 单元测试覆盖率，遵循 Google Java Style。

## Phase 1: 架构研究与多版本升级设计

### Task 1.1 – spring-boot-migrator 架构深度分析 - Agent_Research
**Objective:** 深入理解 spring-boot-migrator 的架构设计、核心模块、扩展机制和 OpenRewrite 集成方式。
**Output:** 架构分析文档（可以是内部笔记或 Markdown 文档），包含核心模块职责、扩展点位置、OpenRewrite 集成方式等关键信息。
**Guidance:** 用户对 spring-boot-migrator 完全不熟悉，需要系统性的循序渐进探索。重点关注扩展点和 OpenRewrite 使用方式，为后续集成新功能奠定基础。

1. 分析项目整体结构：浏览 components/、applications/ 等目录，识别主要模块和职责
2. 深入研究核心模块：重点分析 sbm-core、sbm-openrewrite、sbm-recipes-boot-upgrade 等核心组件的设计
3. 研究 OpenRewrite 集成方式：理解如何使用 OpenRewrite 实现代码迁移规则
4. 识别扩展点和插件机制：找到如何添加新迁移规则、新功能的扩展点
5. 分析 Maven 项目解析模块：理解如何解析和操作 pom.xml、依赖管理等
6. 总结架构关键发现：文档化核心模块、扩展点、集成方式，为后续设计提供基础

### Task 1.2 – 多版本升级架构设计 - Agent_Research
**Objective:** 设计支持 1.x/2.x/3.x 任意版本升级的核心架构，这是项目的核心创新。
**Output:** 详细的多版本升级架构设计文档，包含类图、流程图、规则组织方案、版本管理策略等。
**Guidance:** **Depends on: Task 1.1 Output**. 每个版本对应独立规则集（代码硬编码）。设计需要支持分步升级（如 1.x→2.x→3.x）。这是关键设计决策，需要用户审核确认。

1. 分析多版本升级需求：明确支持的升级路径（1.x→2.x, 2.x→3.x, 1.x→3.x）、每个版本独立规则集、分步升级流程
2. 设计版本规则管理架构：定义规则集组织方式（如何存储、加载、选择对应版本的规则）
3. 设计分步升级流程：定义跨大版本升级时的分步逻辑（如 1.5.x→2.7.x→3.5.x 的自动分步执行）
4. 提交设计文档供用户审核：包含类图、流程图、规则组织方案，等待用户反馈
5. 根据用户反馈调整设计：迭代优化架构方案直到用户确认

### Task 1.3 – 版本检测机制设计与实现 - Agent_Research
**Objective:** 设计并实现自动检测 Spring Boot 版本的机制。
**Output:** 版本检测模块代码（VersionDetector 接口和实现类）及单元测试。
**Guidance:** 检测优先级：依赖推断 → spring-boot-starter-parent → spring-boot-dependencies。需要支持多模块项目。遵循 Google Java Style，中文注释。

1. 设计版本检测接口和数据模型：定义 VersionDetector 接口、Version 类等
2. 实现基于依赖推断的版本检测：分析项目依赖中的 Spring Boot 相关 jar，推断版本（优先级最高）
3. 实现基于 spring-boot-starter-parent 的版本检测：从 pom.xml 读取 parent 版本
4. 实现基于 spring-boot-dependencies 的版本检测：从 dependencyManagement 读取版本
5. 整合多种检测策略：按优先级顺序尝试，返回最准确的版本
6. 编写单元测试：测试各种场景（有 parent、无 parent、多模块等）

### Task 1.4 – 分步升级策略设计 - Agent_Research
**Objective:** 设计跨大版本分步升级的策略和流程控制逻辑。
**Output:** 分步升级策略文档和核心流程控制器代码。
**Guidance:** 用户从 1.x 直接升级到 3.x 时，工具应自动分步执行（如 1.x→2.x→3.x）。需要状态管理、错误恢复机制。

1. 分析所有可能的升级路径：定义版本图（如 1.x→2.y→3.z 的可能路径）
2. 设计分步策略：确定中间版本选择逻辑（如 1.5.x 升级到 3.x 时选择 2.7.x 作为中间版本）
3. 实现流程控制器：管理多步升级的执行顺序、状态传递
4. 实现状态管理：记录当前升级进度、中间结果，支持断点续传或错误恢复
5. 编写单元测试：测试各种分步升级场景（1→2→3、1→3 直接等）

### Task 1.5 – 兼容性数据源研究 - Agent_Research
**Objective:** 研究 Spring 官方或社区是否提供在线 API 用于查询组件兼容性，评估可行性。
**Output:** 可行性研究报告和推荐方案（官方 API、社区数据源、离线数据等）。
**Guidance:** 需要执行中研究在线 API 是否存在。如果不存在，研究替代方案（爬取文档、社区数据源）。离线优先设计，在线查询作为补充。

1. 搜索 Spring 官方兼容性 API：查找 Spring Boot 官方文档、API、工具
2. 测试 API 可行性：如果存在 API，测试访问方式、数据格式、覆盖范围
3. 研究替代方案：如果官方 API 不存在或不可用，研究社区数据源、文档爬取等方案
4. 评估方案优劣：对比官方 API、社区数据源、离线数据等方案的优缺点
5. 总结推荐方案：编写研究报告，提出最佳方案和备选方案

### Task 1.6 – 扩展点识别与集成策略规划 - Agent_Research
**Objective:** 识别 spring-boot-migrator 的具体扩展点，规划如何集成新功能而尽量减少对现有代码的修改。
**Output:** 扩展点清单和集成策略文档，明确各模块的集成位置和方式。
**Guidance:** **Depends on: Task 1.1 Output**. 需要尽量独立添加新类，减少对现有代码修改。完全保留 spring-boot-migrator 原有功能，确保向后兼容。

1. 基于 Task 1.1 成果识别具体扩展点：确定在哪些位置可以插入新功能（jar 分析、兼容性检测等）
2. 规划 Jar 分析模块集成方式：确定如何在现有流程中调用 jar 分析功能
3. 规划兼容性检测和配置迁移集成方式：设计集成接口，减少对现有代码修改
4. 编写集成策略文档：总结扩展点、集成方式、预期代码变更范围

## Phase 2: 核心 Jar 分析模块开发

### Task 2.1 – Jar 文件解析模块设计与实现 - Agent_Jar_Analysis
**Objective:** 实现 jar 文件的底层解析能力，提取 pom.xml/pom.properties 信息。
**Output:** JarParser 核心模块和相关数据模型类（JarInfo, PomInfo 等），包含单元测试。
**Guidance:** 使用 JarFile/ZipFile API。遵循 Google Java Style，中文注释。实现健壮的异常处理和 SLF4J 日志记录。数据模型使用资源文件（YAML/JSON）存储配置。

1. 设计 JarParser 接口和数据模型：定义 JarParser 接口、JarInfo/PomInfo 等数据类
2. 实现 jar 文件读取：使用 JarFile/ZipFile API 读取 jar 内容
3. 实现 pom.xml 提取和解析：从 META-INF/maven 目录提取并解析 groupId/artifactId/version
4. 实现 pom.properties 提取和解析：作为 pom.xml 的补充或备用方案
5. 实现异常处理和日志记录：使用 SLF4J，处理损坏的 jar 或缺失 pom 信息，中文日志
6. 编写单元测试：测试正常 jar、缺失 pom 的 jar、损坏 jar 等场景，确保 40% 覆盖率

### Task 2.2 – 组件识别引擎开发 - Agent_Jar_Analysis
**Objective:** 实现多种方式的组件识别逻辑，判断 jar 所属组件。
**Output:** ComponentIdentifier 模块，支持可扩展的策略模式架构。
**Guidance:** **Depends on: Task 2.1 Output**. 支持文件名、pom 信息、特征类三种识别方式。组件信息使用资源文件（YAML/JSON）配置。无法识别时返回 unknown 状态供后续处理。

1. 设计组件识别策略接口：支持多种识别方式的可扩展架构（策略模式）
2. 实现基于 jar 文件名的组件识别：解析文件名模式（如 logback-core-*.jar）
3. 实现基于 pom 信息的组件识别：根据 groupId/artifactId 匹配已知组件
4. 实现基于特征类的组件识别：检查 jar 内是否存在特定的标志性类
5. 实现组件识别策略的优先级整合：pom 信息优先，文件名和特征类作为补充；无法识别时返回 unknown 状态供后续处理
6. 编写单元测试：测试各种组件识别场景，包括 Spring Boot、MyBatis、Logback 等常见组件

### Task 2.3 – Shaded Jar 递归分析实现 - Agent_Jar_Analysis
**Objective:** 实现 shaded jar 的递归分析能力，识别被 shaded 的原始依赖。
**Output:** ShadedJarAnalyzer 模块，支持递归深度遍历。
**Guidance:** **Depends on: Task 2.1 Output**. 处理包名重定位和嵌套依赖，使用递归算法。以 duckling 项目的 shaded jar 为主要测试目标。

1. 实现 shaded jar 检测逻辑：识别包含重定位包名或内部依赖的 jar（通过包结构特征或 pom 中的 shaded 插件配置）
2. 提取 shaded jar 内部的 pom.xml 文件：可能在非标准位置（如 META-INF/maven 的子目录）
3. 递归分析 shaded 依赖：从内部 pom 提取原始依赖信息，递归处理嵌套 shaded 依赖
4. 整合 shaded 依赖信息：构建完整的依赖树，包括原始组件和版本
5. 编写单元测试：使用 duckling 项目的 shaded jar 作为测试用例，验证识别准确性

### Task 2.4 – Maven 依赖树集成 - Agent_Jar_Analysis
**Objective:** 集成 Maven 依赖树解析，获取项目完整依赖列表。
**Output:** MavenDependencyResolver 模块，提供完整的依赖分析能力。
**Guidance:** **Depends on: Task 2.1 Output**. **Depends on: Task 2.2 Output**. 使用 Maven Resolver API 或 mvn 命令。整合 JarParser 和 ComponentIdentifier 实现完整的依赖分析流程。

1. 集成 Maven Resolver API 或使用 mvn dependency:tree 命令：解析项目依赖
2. 构建完整的依赖树：包括直接依赖和传递依赖
3. 定位 Maven 本地仓库中 jar 文件的路径：根据 groupId/artifactId/version 定位文件
4. 集成 JarParser 和 ComponentIdentifier 模块：对每个依赖调用 jar 解析和组件识别
5. 编写集成测试：使用实际 Maven 项目测试依赖解析和 jar 分析的完整流程

## Phase 3: 迁移引擎扩展与多版本支持

### Task 3.1 – spring-boot-migrator 扩展点集成 - Agent_Migration
**Objective:** 将 jar 分析模块集成到 spring-boot-migrator 的扩展点中。
**Output:** 集成适配器和迁移流程配置，实现 jar 分析在迁移流程中的调用。
**Guidance:** **Depends on: Task 2.4 Output by Agent_Jar_Analysis**. **Depends on: Task 1.6 Output by Agent_Research**. 基于 Task 1.6 的集成策略，使用适配器模式桥接模块。尽量独立添加新类，减少对现有代码修改。

1. 基于 Task 1.6 的集成策略识别具体扩展点位置
2. 开发集成适配器：将 JarParser、ComponentIdentifier、ShadedJarAnalyzer 桥接到 spring-boot-migrator
3. 修改迁移流程配置：在依赖分析阶段调用 jar 分析模块
4. 配置 jar 分析在迁移流程中的执行顺序和数据传递
5. 编写集成测试：验证 jar 分析结果能正确传递给后续迁移步骤

### Task 3.2 – 多版本兼容性检测模块 - Agent_Migration
**Objective:** 实现组件与多版本 Spring Boot 的兼容性检测功能。
**Output:** CompatibilityChecker 模块和兼容性数据源（资源文件格式）。
**Guidance:** **Depends on: Task 2.2 Output by Agent_Jar_Analysis**. **Depends on: Task 1.5 Output by Agent_Research**. 内置兼容性数据聚焦核心组件（Spring Boot 全家桶+MyBatis/Logback 等常用第三方），使用 YAML/JSON 资源文件存储。基于 Task 1.5 的研究结果实现在线查询（如果可行）。设计自动化数据更新机制。离线优先，在线查询作为补充。

1. 收集并整理多版本兼容性数据：从 Spring Boot 官方文档、发布说明整理 1.x/2.x/3.x 各版本的组件兼容性
2. 构建内置兼容性数据库：使用 YAML/JSON 资源文件存储，支持版本范围匹配
3. 实现兼容性检测逻辑：根据组件名称、版本和目标 Spring Boot 版本判断兼容性，返回兼容/不兼容/未知状态
4. 实现在线查询和缓存机制：基于 Task 1.5 研究结果，如果在线 API 可用则集成，缓存查询结果
5. 设计兼容性数据自动更新机制：规划如何定期更新兼容性数据（如脚本、工具等），尽量自动化
6. 编写单元测试：测试内置数据查询、在线查询、缓存机制、多版本匹配等场景

### Task 3.3 – 依赖版本自动更新 - Agent_Migration
**Objective:** 实现自动更新 pom.xml 中依赖版本的功能，支持多版本升级。
**Output:** PomUpdater 模块，支持单模块和多模块项目。
**Guidance:** **Depends on: Task 3.2 Output**. 支持单模块和多模块项目，保持 pom.xml 格式。处理 dependencyManagement 和 dependencies。基于兼容性检测结果决定版本更新策略。

1. 实现 pom.xml 解析：读取项目的依赖声明，支持单模块和多模块项目
2. 匹配 jar 分析结果：将 pom 中的依赖与 jar 分析识别的组件关联
3. 实现版本更新决策逻辑：基于兼容性检测结果和目标 Spring Boot 版本决定是否更新、更新到哪个版本
4. 实现 pom.xml 修改：使用 DOM/JDOM 等库更新依赖版本，保持原有格式和注释
5. 支持多模块项目：处理父 pom 和子模块 pom 的 dependencyManagement 和 dependencies
6. 编写单元测试和集成测试：测试单模块、多模块、dependencyManagement 等场景

### Task 3.4 – 分步升级逻辑实现 - Agent_Migration
**Objective:** 实现跨大版本自动分步升级的执行逻辑。
**Output:** 分步升级执行器，集成 Task 1.4 的设计。
**Guidance:** **Depends on: Task 1.4 Output by Agent_Research**. **Depends on: Task 3.3 Output**. 基于 Task 1.4 的策略设计实现执行器。自动连续执行所有步骤。整合版本检测、兼容性检测、依赖更新等功能。实现状态管理和错误处理。

1. 基于 Task 1.4 的设计实现分步升级执行器
2. 集成版本检测（Task 1.3）、兼容性检测（Task 3.2）、依赖更新（Task 3.3）功能
3. 实现自动分步执行逻辑：如 1.5.x→2.7.x→3.5.x 的自动连续执行
4. 实现状态管理和日志记录：记录每步的执行结果、中间状态
5. 实现错误处理：遇到错误时让用户选择（继续/停止），记录错误信息
6. 编写集成测试：测试各种分步升级场景（1→2→3、2→3、1→3 等）

### Task 3.5 – OpenRewrite 迁移规则扩展 - Agent_Migration
**Objective:** 基于 OpenRewrite 扩展 spring-boot-migrator 的迁移规则，支持多版本特定场景。
**Output:** 新增的 OpenRewrite 迁移规则（javax→jakarta、废弃 API、配置属性等），支持多版本。
**Guidance:** **Depends on: Task 1.1 Output by Agent_Research**. 基于 Task 1.1 对 OpenRewrite 集成的理解，扩展规则。研究 Spring Boot 官方迁移指南确保规则准确性。支持 1.x→2.x、2.x→3.x 的不同迁移场景。

1. 研究多版本迁移场景：从 Spring Boot 官方文档整理 1.x→2.x、2.x→3.x 的主要迁移场景
2. 实现 javax→jakarta 包名迁移规则（2.x→3.x）：基于 OpenRewrite 自动替换 import 语句和类引用
3. 实现废弃 API 替换规则：识别并替换不同版本中移除的 API
4. 实现配置属性迁移规则：更新 application.properties/yml 中变更的属性名（不同版本）
5. 实现版本特定规则路由：根据源版本和目标版本选择适用的规则集
6. 编写测试用例：使用示例项目验证各版本迁移规则的正确性

## Phase 4: 配置迁移功能实现

### Task 4.1 – YAML 规则格式设计 - Agent_Configuration
**Objective:** 设计 YAML 规则的格式和语法，支持工具开发者和用户自定义。
**Output:** 规则格式规范文档、示例规则文件和验证 schema。
**Guidance:** 规则需要支持属性重命名、删除、新增、值转换等场景。支持多版本（不同版本使用不同规则文件）。支持用户自定义规则文件扩展。设计应简洁易懂。这是关键设计决策，需要用户审核确认。

1. 分析配置迁移场景：属性重命名、删除、新增、值转换、条件执行等典型场景，覆盖 1.x→2.x、2.x→3.x
2. 设计规则类型和语法：rename、remove、add、transform 等规则类型及其参数，支持条件表达式
3. 定义 YAML 格式和版本组织：规则文件结构、版本标识、规则声明格式
4. 编写示例规则文件：覆盖常见的多版本配置变更场景
5. 设计用户自定义规则加载机制：规划如何加载和验证用户提供的规则文件
6. 提交规则格式文档供用户审核：包含格式说明、示例、schema，等待用户反馈并迭代

### Task 4.2 – 规则引擎开发 - Agent_Configuration
**Objective:** 实现规则引擎，解析 YAML 规则文件并构建规则执行计划。
**Output:** RuleEngine 模块，支持规则验证、冲突检测、执行计划生成。
**Guidance:** **Depends on: Task 4.1 Output**. 使用 SnakeYAML 或 Jackson YAML 模块。实现规则验证、冲突检测、执行顺序优化。支持加载工具内置规则和用户自定义规则。

1. 设计规则数据模型：Rule、RenameRule、RemoveRule、AddRule、TransformRule 等类
2. 实现 YAML 规则文件解析：使用 SnakeYAML 或 Jackson YAML，支持多版本规则文件
3. 实现规则验证：检查规则语法正确性、必填字段、版本匹配等
4. 实现规则冲突检测：识别相互冲突的规则（如同一属性的多次重命名）
5. 构建规则执行计划：根据规则依赖关系排序、优化执行顺序
6. 实现条件表达式引擎：支持规则的条件判断（如仅在特定配置存在时执行）
7. 编写单元测试：测试 YAML 解析、规则验证、冲突检测、执行计划生成等

### Task 4.3 – 配置文件迁移执行器实现 - Agent_Configuration
**Objective:** 实现配置文件迁移执行器，根据规则修改配置文件。
**Output:** ConfigMigrator 模块，支持 application.properties/yml、bootstrap.*、logback 等配置文件。
**Guidance:** **Depends on: Task 4.2 Output**. 保持配置文件的原有格式、注释、缩进。支持多种配置文件类型（application.*/bootstrap.*/logback.xml 等）。实现备份和验证机制。

1. 实现 application.properties 解析和修改：保持注释、顺序、格式
2. 实现 application.yml 和 bootstrap.yml 解析和修改：保持 YAML 结构、注释、缩进
3. 实现 logback.xml 等其他配置文件的解析和修改：支持 XML 格式配置
4. 实现规则应用逻辑：执行 rename、remove、add、transform 等规则操作
5. 实现配置文件格式保持：确保修改后的文件保持原有的格式风格
6. 实现配置文件备份和验证：修改前备份，修改后验证语法正确性
7. 编写集成测试：使用实际的 Spring Boot 配置文件和规则测试迁移效果

## Phase 5: CLI、备份回滚与报告生成

### Task 5.1 – CLI 框架集成与交互式界面 - Agent_UX
**Objective:** 实现命令行界面，集成 CLI 框架，实现交互式版本选择。
**Output:** CLI 模块，支持交互式和命令行参数两种方式。
**Guidance:** 选择 picocli 或 JCommander（我来选择）。实现交互式目标版本输入（默认 3.5.x 最新版）。支持详细模式（-v/--verbose）。保持向后兼容（原有命令行参数仍有效）。全平台支持（macOS/Linux/Windows）。

1. 选择并集成 CLI 框架：评估 picocli 和 JCommander，选择合适的框架集成
2. 设计命令行参数结构：项目路径、目标版本、详细模式、备份选项、回滚选项等，保持向后兼容
3. 实现交互式版本选择：自动检测当前版本，提示用户输入目标版本（默认 3.5.x 最新版）
4. 实现详细模式和日志管理：默认显示关键信息，-v/--verbose 显示详细日志
5. 实现进度显示：尽量实现进度条，但不影响性能和效果（如果实现困难则使用简单输出）
6. 编写跨平台测试：在 macOS、Linux、Windows 上测试 CLI 功能

### Task 5.2 – 备份回滚机制实现 - Agent_UX
**Objective:** 实现完整的备份和回滚功能。
**Output:** BackupManager 和 RollbackManager 模块。
**Guidance:** 备份整个项目目录。回滚到最初始状态。实现备份索引和版本管理。提供命令行界面供用户选择回滚。

1. 设计备份目录结构：在项目根目录创建 .spring-boot-upgrade-backup 目录
2. 实现整个项目目录备份：复制项目目录到备份位置，保持相对路径结构
3. 实现备份索引：记录备份时间、备份文件列表、原始路径、升级路径等元数据（JSON 格式）
4. 实现回滚功能：从备份目录恢复整个项目到原始位置
5. 实现回滚验证：确认所有文件成功恢复，检测恢复过程中的错误
6. 集成到 CLI：提供 --rollback 参数和交互式回滚选择
7. 编写集成测试：测试完整的备份和回滚流程

### Task 5.3 – Markdown 迁移报告生成器 - Agent_UX
**Objective:** 实现详细的 Markdown 迁移报告生成器。
**Output:** ReportGenerator 模块和报告模板。
**Guidance:** 报告应包含：依赖分析、迁移操作、兼容性检测、风险提示、手动处理项等所有信息。仅 Markdown 格式。

1. 设计报告模板结构：概述、检测到的版本、升级路径、依赖分析、迁移操作、兼容性检测、风险提示、手动处理项、建议等章节
2. 实现依赖分析汇总：整合 jar 分析结果，列出识别的组件、版本、shaded 依赖等
3. 实现迁移操作汇总：记录所有文件修改、依赖更新、配置变更、代码迁移等操作
4. 实现兼容性检测报告：列出兼容、不兼容、未知兼容性的组件，按版本分组
5. 实现风险提示和手动处理项：标记需要手动处理的项、潜在问题、后续步骤建议
6. 实现报告生成和文件输出：生成 Markdown 格式报告，保存到项目目录
7. 编写单元测试：测试报告生成的各个部分，验证 Markdown 格式正确性

### Task 5.4 – 错误处理与用户交互 - Agent_UX
**Objective:** 实现统一的错误处理和用户交互机制。
**Output:** 错误处理框架和用户交互工具类。
**Guidance:** **Depends on: Task 3.4 Output by Agent_Migration**. 遇到错误时让用户选择（继续/停止）。实现友好的错误提示和日志记录。处理无法识别组件时的用户手动指定流程。

1. 设计统一的错误处理框架：异常分类、错误码、错误消息模板
2. 实现错误拦截和用户交互：遇到错误时暂停执行，提示用户选择（继续/停止/重试）
3. 实现无法识别组件的处理流程：提示用户手动指定组件名称和版本
4. 实现友好的错误提示：中文错误消息，包含错误原因和建议的解决方案
5. 集成到 CLI 和迁移流程：在关键节点集成错误处理
6. 编写集成测试：测试各种错误场景和用户交互流程

## Phase 6: 测试验证与文档完善

### Task 6.1 – 单元测试覆盖率完善 - Agent_QA
**Objective:** 完善所有新增和修改代码的单元测试，确保达到 40% 覆盖率。
**Output:** 完整的测试套件和 JaCoCo 覆盖率报告。
**Guidance:** 使用 JaCoCo 工具统计覆盖率。重点为 jar 分析、组件识别、兼容性检测、配置迁移等核心模块补充测试。覆盖率针对新增代码和修改的现有代码。

1. 配置 JaCoCo 覆盖率工具：集成到 Maven 构建，配置覆盖率报告生成
2. 分析当前测试覆盖率：识别覆盖率不足的模块和类
3. 补充核心模块单元测试：jar 分析、组件识别、兼容性检测、配置迁移等
4. 验证覆盖率达标：执行测试并生成覆盖率报告，确保新增和修改代码达到 40% 覆盖率

### Task 6.2 – 集成测试与端到端测试 - Agent_QA
**Objective:** 实现集成测试和端到端测试，验证完整迁移流程。
**Output:** 集成测试套件和测试报告。
**Guidance:** **Depends on: Task 3.4 Output by Agent_Migration**. **Depends on: Task 4.3 Output by Agent_Configuration**. **Depends on: Task 5.3 Output by Agent_UX**. 创建测试项目（testcode/ 目录），覆盖典型场景（Spring MVC + MyBatis）。使用 duckling 项目测试 shaded jar 处理。这是关键验证点，需要用户审核测试结果。

1. 在 testcode/ 创建测试项目：Spring MVC + MyBatis 等典型场景，覆盖 1.x、2.x 版本
2. 执行完整迁移流程测试：测试各种升级路径（1.x→2.x、2.x→3.x、1.x→3.x）
3. 验证 duckling 项目 shaded jar 处理：执行 duckling 项目迁移，检查 shaded 依赖识别准确性
4. 分析迁移结果：检查生成的报告、修改的文件、依赖更新、配置变更是否正确
5. 诊断并修复发现的问题：根据测试结果修复工具缺陷或改进功能
6. 提交测试结果供用户审核：包含测试报告、发现的问题、修复方案等

### Task 6.3 – CLI 工具打包与优化 - Agent_QA
**Objective:** 将工具打包为可执行 jar，优化命令行界面，确保跨平台兼容性。
**Output:** 可执行 jar 包和启动脚本（.sh 和 .bat），版本号 0.15.2-plus。
**Guidance:** **Depends on: Task 5.1 Output by Agent_UX**. 使用 maven-shade-plugin 打包。支持 Windows/Linux/macOS。基于 spring-boot-migrator 的版本号（0.15.2-plus）。仅交付 jar，不需要 Docker 镜像等。

1. 配置 Maven 打包插件：使用 maven-shade-plugin 创建可执行 jar，设置版本号 0.15.2-plus
2. 配置主类和依赖打包：确保所有依赖都被正确打包到 jar 中
3. 编写启动脚本：创建 .sh（Linux/macOS）和 .bat（Windows）启动脚本
4. 优化 jar 大小和启动性能：移除不必要的依赖，优化资源文件
5. 跨平台测试：在 macOS、Linux、Windows 环境测试工具执行
6. 编写安装和使用说明：简单的 README，说明如何运行工具

### Task 6.4 – 用户手册编写 - Agent_QA
**Objective:** 编写用户手册，指导用户如何使用工具进行项目升级。
**Output:** 中文 Markdown 格式用户手册。
**Guidance:** 包含安装、使用、配置、故障排除等章节。提供使用示例和常见问题解答。文档类审查可以跳过（根据 Context Synthesis）。

- 设计用户手册结构：简介、安装、快速开始、详细使用、自定义规则配置、回滚操作、故障排除等章节
- 编写安装和环境要求：Java 17、Maven 3.6+、操作系统等要求
- 编写使用指南：命令行参数、交互式流程、多版本升级示例、回滚操作等
- 提供使用示例和常见问题解答：典型使用场景、错误处理、最佳实践
- 编写自定义 YAML 规则配置指南：说明如何编写和使用自定义规则

### Task 6.5 – 开发文档编写 - Agent_QA
**Objective:** 编写开发文档，说明工具的架构、模块设计、扩展方式等技术细节。
**Output:** 中文 Markdown 格式开发文档。
**Guidance:** 包含架构概览、核心模块、扩展开发、API 参考等内容。帮助开发者理解和扩展工具。文档类审查可以跳过（根据 Context Synthesis）。

- 编写架构概览：整体架构设计、技术栈、模块关系图、多版本升级架构
- 编写核心模块文档：jar 分析、组件识别、兼容性检测、迁移引擎、配置迁移等模块的设计和实现
- 编写扩展开发指南：如何添加新的组件识别策略、迁移规则、兼容性数据源、YAML 规则等
- 提供 API 参考和代码示例：关键类和接口的使用方法
- 编写数据更新指南：说明如何更新兼容性数据（未来持续维护）
