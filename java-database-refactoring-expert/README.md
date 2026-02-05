# Java Database Refactoring Expert

<div align="center">

**[中文](#中文文档) | [English](#english-documentation)**

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](https://github.com/YOUR_USERNAME/java-database-refactoring-expert)
[![Claude](https://img.shields.io/badge/Clade-Code-purple.svg)](https://claude.ai/code)

**让Java数据库重构变得简单、安全、可控**

一款专为Claude Code设计的AI技能，通过交互式引导完成大型系统的数据库架构重构。

</div>

---

## 中文文档

### 📖 简介

**Java Database Refactoring Expert** 是一款专业的数据库架构重构助手技能。它采用四阶段工作流，通过交互式问答收集系统信息，自动生成完整的架构诊断报告和实施方案。

当你面临以下问题时，本技能可以提供帮助：
- 📊 **单表字段过多**（100+字段）导致查询性能下降
- 💾 **数据量过大**（千万级）需要分库分表
- 🔗 **字段冗余严重**，多表存储重复数据
- 🚀 **系统性能瓶颈**，需要架构升级
- 🔄 **在线不停机**迁移需求

---

### ✨ 核心特性

| 特性 | 说明 |
|------|------|
| 🎯 **交互式引导** | 使用弹窗问答收集信息，体验友好 |
| 📊 **自动分析** | 智能分析表结构，识别性能瓶颈 |
| 📋 **对比文档** | 自动生成字段对比分析文档 |
| 🗃️ **完整脚本** | 输出DDL/迁移/校验脚本 |
| 💻 **代码示例** | 提供完整的Java代码改造示例 |
| 🔄 **在线迁移** | 支持双写和灰度发布方案 |
| 📁 **中文支持** | 生成中文目录和文档 |

---

### 🚀 快速开始

#### 安装

```bash
# 克隆仓库（请替换为实际的仓库地址）
git clone https://github.com/YOUR_USERNAME/java-database-refactoring-expert.git

# 复制技能文件到Claude Code技能目录
cp -r java-database-refactoring-expert ~/.claude/skills/

# 或使用符号链接
ln -s $(pwd)/java-database-refactoring-expert ~/.claude/skills/
```

#### 目录结构

```
.java-database-refactoring-expert/
├── SKILL.md                    # 技能定义文件（核心）
├── README.md                   # 本文档
├── references/                 # 参考文档库
│   ├── vertical-split.md       # 垂直拆分最佳实践
│   ├── horizontal-split.md     # 水平拆分策略详解
│   ├── mysql-optimization.md   # MySQL性能优化指南
│   └── data-migration.md       # 数据迁移方案参考
└── assets/                     # 资源文件（可选）
    └── images/                 # 文档配图
```

#### 使用方法

在Claude Code中调用技能：

```
/user: 我需要对订单表进行拆分，有5000万数据
/assistant: [调用 java-database-refactoring-expert 技能]
```

技能将通过交互式问答收集信息：

1. **项目名称** → 创建工作目录
2. **系统信息** → 业务类型、技术栈、数据量
3. **痛点问题** → 当前遇到的具体问题
4. **数据库现状** → 表结构、索引、查询情况
5. **业务约束** → 一致性要求、停机时间等

---

### 📋 四阶段工作流

```
┌─────────────────────────────────────────────────────────────┐
│                    架构重构四阶段工作流                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  阶段1: 信息收集 (01-信息收集/)                             │
│   ├── 系统基础信息.md                                       │
│   ├── 痛点与问题.md                                         │
│   ├── 数据库现状.md                                         │
│   └── 业务约束.md                                           │
│                                                             │
│  阶段2: 架构诊断 (02-架构诊断/)                             │
│   ├── 现状评估.md       ← 性能瓶颈分析                      │
│   ├── 问题归类.md       ← 问题优先级                        │
│   └── 拆分必要性判断.md  ← 拆分类型建议                     │
│                                                             │
│  阶段3: 方案设计 (03-方案设计/)                             │
│   ├── 垂直拆分方案.md    ← 字段分组策略                     │
│   ├── 水平拆分方案.md    ← 分片算法设计                     │
│   ├── 技术选型.md        ← 中间件选择                       │
│   └── 数据一致性方案.md  ← 一致性保障                       │
│                                                             │
│  阶段4: 实施指导 (04-实施指导/)                             │
│   ├── 字段对比分析.md    ← 自动生成对比文档                 │
│   ├── 双写迁移方案.md    ← 在线迁移方案                     │
│   ├── 00-实施检查清单.md                                   │
│   ├── 01-创建新表DDL.sql                                   │
│   ├── 02-数据迁移脚本.sql                                   │
│   ├── 03-数据校验脚本.sql                                   │
│   └── 代码示例/                                             │
│       └── 应用改造示例.java                                 │
│                                                             │
│  交付: (05-交付物/)                                         │
│       ├── 架构现状分析报告.md                                │
│       ├── 架构升级方案.md                                    │
│       ├── 实施计划.md                                        │
│       └── 数据迁移方案.md                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 📊 输出示例

使用本技能分析某大型业务表（如订单表、用户表等）后，生成以下输出：

#### 表结构对比

| 表名 | 字段数 | 说明 |
|------|--------|------|
| 旧表 `demo_business_table` | **170** | 所有字段在一个表 |
| 新核心表 | **20** | 减少 **88%** ⬇️ |
| 地址扩展表 | 30 | 详细地址信息 |
| 业务扩展表 | 40 | 业务相关字段 |

#### 生成的实施资源

```bash
04-实施指导/
├── 字段对比分析.md           # 详细字段映射关系
├── 双写迁移方案.md           # 6周在线迁移计划
├── 00-实施检查清单.md        # 上线前后检查项
├── 01-创建新表DDL.sql        # 创建新表脚本
├── 02-数据迁移脚本.sql        # 数据迁移脚本
├── 03-数据校验脚本.sql        # 数据一致性校验
└── 代码示例/
    └── 应用改造示例.java      # DAO/Service/Controller示例
```

#### 代码示例

```java
// 双写模式示例
@Override
@Transactional(rollbackFor = Exception.class)
public void insert(DemoDO entity) {
    // 1. 写旧表（同步，确保业务成功）
    writeOldTable(core, address, business);

    // 2. 写新表（根据配置异步/同步）
    if (stateManager.needWriteNewTable()) {
        if (Boolean.TRUE.equals(dualWrite.getAsync())) {
            asyncWriteNewTable(core, address, business);
        } else {
            syncWriteNewTable(core, address, business);
        }
    }
}

// 灰度读示例
public DemoDO getByCode(String code) {
    Integer fromNewPercent = grayRead.getFromNewPercent();
    boolean readFromNew = shouldReadFromNew(code, fromNewPercent);

    return readFromNew
        ? readFromNewTable(code)
        : readFromOldTable(code);
}
```

---

### 🎯 适用场景

| 场景 | 技能输出 |
|------|---------|
| **单表字段过多** | 垂直拆分方案（核心表+扩展表） |
| **数据量过大** | 水平拆分方案（分片策略设计） |
| **字段冗余严重** | 去冗余方案（关联查询设计） |
| **需要不停机迁移** | 双写迁移方案（灰度发布） |
| **分库分表需求** | 分片算法设计 + 中间件选型 |

---

### 🔧 技术栈

- **数据库**: MySQL 5.7+, MySQL 8.0+
- **Java框架**: Spring Boot, Spring Cloud
- **ORM**: MyBatis, MyBatis-Plus
- **中间件**: ShardingSphere, MyCAT, TiDB
- **迁移方式**: 停机迁移, 双写迁移, Binlog同步

---

### 📚 参考文档

技能内置以下参考文档：

- `references/vertical-split.md` - 垂直拆分最佳实践
  - 字段分组策略
  - 核心表与扩展表设计
  - 关联查询优化

- `references/horizontal-split.md` - 水平拆分策略详解
  - 分片键选择
  - 分片算法（范围/Hash/一致性Hash）
  - 分片数量规划

- `references/mysql-optimization.md` - MySQL性能优化指南
  - 索引优化策略
  - 查询优化技巧
  - 表结构设计规范

- `references/data-migration.md` - 数据迁移方案参考
  - 停机迁移方案
  - 双写迁移方案
  - Binlog同步方案

---

### 🤝 贡献指南

欢迎贡献代码和建议！

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

#### 贡献方向

- 📖 完善参考文档（references/）
- 🎨 优化技能交互流程
- 🔧 增加新的重构模式
- 🐛 修复Bug
- 🌍 添加多语言支持

---

### 📝 更新日志

#### v1.0.0 (2024-02-04)
- ✅ 初始版本发布
- ✅ 支持四阶段工作流
- ✅ 自动生成对比分析文档
- ✅ 输出完整实施资源
- ✅ 中文目录和文档支持
- ✅ 双写迁移方案支持

---

### ❓ 常见问题

**Q: 技能支持哪些数据库？**
A: 目前主要支持MySQL，后续会扩展到PostgreSQL、Oracle等。

**Q: 生成的脚本可以直接在生产使用吗？**
A: 建议先在测试环境验证，根据实际情况调整后再应用到生产。

**Q: 支持哪些ORM框架？**
A: 主要针对MyBatis和MyBatis-Plus，其他框架可以参考示例代码进行适配。

**Q: 如何自定义技能的行为？**
A: 可以修改SKILL.md文件中的工作流程和问题定义。

---

### 📄 许可证

本项目采用 [MIT License](LICENSE) 开源协议。

---

### 🔗 相关链接

- [Claude Code 官方文档](https://claude.ai/code)
- [技能开发指南](https://docs.claude.ai/code/skills)
- [Issue 跟踪](https://github.com/YOUR_USERNAME/java-database-refactoring-expert/issues)

---

### 📮 贡献与反馈

欢迎通过以下方式参与贡献：

- **GitHub Issues**: [提交问题或建议](https://github.com/YOUR_USERNAME/java-database-refactoring-expert/issues)
- **Pull Requests**: 欢迎提交 PR 改进技能
- **讨论**: 在 Issues 中交流使用经验和最佳实践

---

<div align="center">

**如果这个技能对你有帮助，请给个 ⭐️ Star！**

Made with ❤️ by Architecture Upgrade Team

</div>

---

## English Documentation

### 📖 Overview

**Java Database Refactoring Expert** is a professional AI skill for Claude Code designed to guide database architecture refactoring through a four-stage workflow. It collects system information interactively and automatically generates comprehensive diagnostic reports and implementation plans.

This skill helps when you face:
- 📊 **Too many fields** (100+) in a single table causing poor query performance
- 💾 **Large data volume** (10M+ rows) requiring database sharding
- 🔗 **Severe field redundancy** with duplicate data across tables
- 🚀 **System performance bottlenecks** requiring architecture upgrades
- 🔄 **Online migration** requirements without downtime

---

### ✨ Key Features

| Feature | Description |
|---------|-------------|
| 🎯 **Interactive Guidance** | Collects information via popup questions |
| 📊 **Automatic Analysis** | Analyzes table structures and identifies bottlenecks |
| 📋 **Comparison Docs** | Auto-generates field comparison analysis |
| 🗃️ **Complete Scripts** | Outputs DDL/migration/validation scripts |
| 💻 **Code Examples** | Provides complete Java code refactoring examples |
| 🔄 **Online Migration** | Supports dual-write and gray release strategies |
| 📁 **Chinese Support** | Generates Chinese directories and documents |

---

### 🚀 Quick Start

#### Installation

```bash
# Clone the repository
# 克隆仓库（请替换为实际的仓库地址）
git clone https://github.com/YOUR_USERNAME/java-database-refactoring-expert.git

# Copy to Claude Code skills directory
cp -r java-database-refactoring-expert ~/.claude/skills/

# Or use symbolic link
ln -s $(pwd)/java-database-refactoring-expert ~/.claude/skills/
```

#### Usage

Invoke the skill in Claude Code:

```
/user: I need to split the orders table, it has 50 million rows
/assistant: [Calls java-database-refactoring-expert skill]
```

The skill will collect information through interactive Q&A:

1. **Project Name** → Creates working directory
2. **System Info** → Business type, tech stack, data volume
3. **Pain Points** → Current specific problems
4. **Database Status** → Table structures, indexes, queries
5. **Business Constraints** → Consistency requirements, downtime tolerance

---

### 📋 Four-Stage Workflow

```
┌─────────────────────────────────────────────────────────────┐
│              Architecture Refactoring Workflow               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Stage 1: Information Collection (01-信息收集/)             │
│   ├── System Basic Information                              │
│   ├── Pain Points & Problems                                │
│   ├── Database Current Status                               │
│   └── Business Constraints                                  │
│                                                             │
│  Stage 2: Architecture Diagnosis (02-架构诊断/)             │
│   ├── Current Status Assessment  ← Performance Analysis     │
│   ├── Problem Classification      ← Priority Matrix        │
│   └── Splitting Necessity        ← Split Type Recommendation│
│                                                             │
│  Stage 3: Solution Design (03-方案设计/)                   │
│   ├── Vertical Splitting Plan  ← Field Grouping Strategy   │
│   ├── Horizontal Splitting Plan ← Sharding Algorithm       │
│   ├── Technology Selection     ← Middleware Selection      │
│   └── Data Consistency Plan    ← Consistency Guarantees    │
│                                                             │
│  Stage 4: Implementation Guide (04-实施指导/)              │
│   ├── Field Comparison Analysis     ← Auto-generated       │
│   ├── Dual-Write Migration Plan     ← Online migration     │
│   ├── Implementation Checklist                              │
│   ├── DDL Scripts                                             │
│   ├── Migration Scripts                                       │
│   ├── Validation Scripts                                     │
│   └── Code Examples/                                         │
│       └── Application Refactoring Examples                  │
│                                                             │
│  Deliverables: (05-交付物/)                                  │
│       ├── Architecture Analysis Report                       │
│       ├── Architecture Upgrade Plan                          │
│       ├── Implementation Plan                                │
│       └── Data Migration Plan                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 📊 Output Example

After analyzing a logistics system requirement table, the skill generates:

#### Table Structure Comparison

| Table | Fields | Description |
|-------|--------|-------------|
| Old `demo_business_table` | **170** | All fields in one table |
| New Core Table | **20** | Reduced by **88%** ⬇️ |
| Address Extension | 30 | Detailed address info |
| Business Extension | 40 | Business-related fields |

#### Generated Implementation Resources

```bash
04-实施指导/
├── 字段对比分析.md           # Detailed field mapping
├── 双写迁移方案.md           # 6-week online migration plan
├── 00-实施检查清单.md        # Pre/post-launch checklist
├── 01-创建新表DDL.sql        # DDL scripts
├── 02-数据迁移脚本.sql        # Migration scripts
├── 03-数据校验脚本.sql        # Data validation
└── 代码示例/
    └── 应用改造示例.java      # DAO/Service/Controller examples
```

---

### 🎯 Use Cases

| Scenario | Skill Output |
|----------|-------------|
| **Too Many Fields** | Vertical splitting (core + extension tables) |
| **Large Data Volume** | Horizontal splitting (sharding strategy) |
| **Field Redundancy** | Deduplication plan (join query design) |
| **No-Downtime Migration** | Dual-write migration (gray release) |
| **Sharding Needs** | Sharding algorithm + middleware selection |

---

### 🔧 Tech Stack

- **Database**: MySQL 5.7+, MySQL 8.0+
- **Java Frameworks**: Spring Boot, Spring Cloud
- **ORM**: MyBatis, MyBatis-Plus
- **Middleware**: ShardingSphere, MyCAT, TiDB
- **Migration Methods**: Downtime, Dual-Write, Binlog Sync

---

### 📚 Reference Documentation

Built-in reference documents:

- `references/vertical-split.md` - Vertical splitting best practices
- `references/horizontal-split.md` - Horizontal splitting strategies
- `references/mysql-optimization.md` - MySQL performance optimization
- `references/data-migration.md` - Data migration approaches

---

### 🤝 Contributing

Contributions are welcome!

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Open a Pull Request

---

### 📄 License

This project is licensed under the [MIT License](LICENSE).

---

### 🔗 Related Links

- [Claude Code Documentation](https://claude.ai/code)
- [Skill Development Guide](https://docs.claude.ai/code/skills)
- [Issue Tracker](https://github.com/your-org/java-database-refactoring-expert/issues)

---

<div align="center">

**If this skill helps you, please give it a ⭐️ Star!**

Made with ❤️ by Architecture Upgrade Team

</div>
