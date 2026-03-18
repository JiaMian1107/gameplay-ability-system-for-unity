# EX-GAS 版本对比：1.0 vs 2.0

本文档对 EX Gameplay Ability System（EX-GAS）的 **1.0**（分支 `EX-GAS-1.0`，最终版本 1.1.8）与 **2.0**（分支 `EX-GAS-2.0`，版本 2.0.0）进行全面的对比分析，帮助用户了解两个版本的核心差异，从而做出合适的版本选择。

---

## 目录

- [整体概述](#整体概述)
- [依赖环境对比](#依赖环境对比)
- [架构对比](#架构对比)
- [配置工作流对比](#配置工作流对比)
- [编辑器工具对比](#编辑器工具对比)
- [核心模块对比](#核心模块对比)
  - [GameplayTag](#gameplaytag)
  - [Attribute 与 AttributeSet](#attribute-与-attributeset)
  - [GameplayEffect（GE）](#gameplayeffectge)
  - [GameplayCue](#gameplaycue)
  - [Ability](#ability)
  - [AbilitySystemComponent（ASC）](#abilitysystemcomponentasc)
  - [ModifierMagnitudeCalculation（MMC）](#modifiermagnitudecalculationmmc)
- [新增功能（仅 2.0）](#新增功能仅-20)
- [移除或弃用功能](#移除或弃用功能)
- [迁移建议](#迁移建议)

---

## 整体概述

| 对比项 | EX-GAS 1.0（v1.1.8） | EX-GAS 2.0（v2.0.0） |
|--------|----------------------|----------------------|
| 底层框架 | Unity MonoBehaviour（传统 OOP） | Unity DOTS / ECS（数据导向编程） |
| 配置方式 | ScriptableObject | Excel → Luban → JSON（推荐）或自定义 |
| 数据层与逻辑层 | 耦合 | 强分离 |
| Unity 最低版本 | 2022.3+ | 2022.3+（建议 Unity 6 以下） |
| 必须依赖 | 无强制，Odin Inspector 强烈推荐 | Unity Entities 1.2.3 |
| 可选依赖 | Odin Inspector 3.2+ | Odin Inspector 3.2+、Luban |
| 包版本号 | 1.1.8 | 2.0.0 |

> **重要提示**：2.0 是对底层框架的重大重构，**不兼容** 1.0 的配置资产和代码，无法直接升级迁移。

---

## 依赖环境对比

### EX-GAS 1.0

| 依赖 | 是否必须 | 说明 |
|------|----------|------|
| Unity 2022.3+ | ✅ 必须 | |
| Odin Inspector 3.2+ | ⚠️ 强烈推荐 | 大量编辑器界面依赖 Odin |
| 无其他第三方包 | — | 纯 Unity 运行时 |

### EX-GAS 2.0

| 依赖 | 是否必须 | 说明 |
|------|----------|------|
| Unity 2022.3+ | ✅ 必须 | |
| Unity Entities 1.2.3 | ✅ 必须 | DOTS 框架核心，建议精确匹配版本 |
| Odin Inspector 3.2+ | ⚠️ 可选 | 使用编辑器工作流时需要 |
| Luban | ⚠️ 可选 | 使用 Excel → JSON 配置工作流时需要 |
| Python（本地） | ⚠️ 可选 | 使用 WebEditor 编辑器工具时需要 |

---

## 架构对比

### EX-GAS 1.0：传统 OOP + MonoBehaviour

1.0 版本基于 Unity 的传统 MonoBehaviour 体系：
- **ASC（AbilitySystemComponent）** 是一个挂载在 `GameObject` 上的 `MonoBehaviour` 组件。
- 所有 GAS 对象（Ability、GameplayEffect、Attribute 等）均以 C# 对象/ScriptableObject 的形式存在于托管堆中。
- 通过事件系统进行模块间通信（`GASEvents`）。
- 配置以 ScriptableObject 资产文件形式保存在 `Assets` 目录下。

```
GameObject
  └── AbilitySystemComponent (MonoBehaviour)
        ├── AbilityContainer
        ├── AttributeSetContainer
        ├── GameplayEffectContainer
        └── GameplayTagAggregator
```

### EX-GAS 2.0：ECS / DOTS 架构

2.0 版本基于 Unity Entities（DOTS）重构：
- **AbilitySystemCell** 是 ECS 中的一个 Entity，存储所有 GAS 数据为 ECS Component。
- **AbilitySystemComponent（ASC）** 保留了 MonoBehaviour 接口，但内部持有并操作 `AbilitySystemCell`（ECS Entity），作为逻辑层的桥梁。
- GAS 的逻辑处理由 ECS System（`System/` 目录）驱动，实现并行化和 Burst 编译优化。
- 数据层（ECS Component）与逻辑层（ECS System）强分离。

```
GameObject
  └── AbilitySystemComponent (MonoBehaviour 桥接层)
        └── AbilitySystemCell (ECS Entity)
              ├── CAscBasicData (ECS Component)
              ├── CAttributeData (ECS Component)
              ├── BGameplayEffect (ECS Buffer)
              ├── BEAttrSet (ECS Buffer)
              └── ... (其他 ECS Components)

World (ECS World)
  ├── SAbilitySystem (ECS System)
  ├── SAttributeSystem (ECS System)
  ├── SGlobalTimer (ECS System)
  └── ... (其他 ECS Systems)
```

### 架构对比总结

| 方面 | 1.0 | 2.0 |
|------|-----|-----|
| 核心数据容器 | C# 对象 / ScriptableObject | ECS Component / Buffer |
| 逻辑驱动 | MonoBehaviour Update / 事件 | ECS System（支持 Burst / Jobs） |
| 内存布局 | 托管堆（GC 压力较大） | Native Memory（低 GC，高性能） |
| 并行能力 | 有限 | ECS JobSystem 支持多线程 |
| 扩展方式 | 继承 / 重写 | ECS Component 组合 + AbilityLogic |

---

## 配置工作流对比

### EX-GAS 1.0：ScriptableObject 工作流

所有配置通过 Unity 编辑器创建 ScriptableObject 资产文件完成：
- Tag 在 `ProjectSettings → EX-GAS` 中编辑，代码自动生成 `GTagLib`。
- Attribute / AttributeSet 在专属编辑器窗口中配置，自动生成对应 C# 类。
- GameplayEffect、Ability、GameplayCue、MMC 均以 `.asset` 文件形式存在。
- 适合小型项目或习惯 Unity 资产管理的团队。

### EX-GAS 2.0：Luban + Excel → JSON 工作流（推荐）

2.0 推荐使用基于 Luban 的配置表工作流：

```
Excel 配置表（.xlsx）
    ↓ Luban 导表工具
JSON 数据文件
    ↓ GASCenter 编辑器 / BeanUpdater
C# Bean 类 + Unity 运行时数据加载
```

**配置表文件列表**：

| 配置表文件 | 对应模块 |
|-----------|---------|
| `#exgas.gameplaytags.xlsx` | GameplayTag |
| `#exgas.attribute.xlsx` | Attribute |
| `#exgas.attributeSet.xlsx` | AttributeSet |
| `#exgas.gameplayeffect.xlsx` | GameplayEffect |
| `#exgas.ability.xlsx` | Ability |
| `#exgas.gameplaycue.xlsx` | GameplayCue |
| `#exgas.mmc.xlsx` | MMC |
| `#exgas.asc.xlsx` | ASC 预设 |
| `#exgas.timelineability.xlsx` | TimelineAbility |

**优点**：
- 策划可直接修改 Excel 表格，程序无需改代码。
- 数据与逻辑完全解耦。
- 支持批量配置和版本管理。
- `BeanUpdater.cs` 自动同步 Excel 字段与 C# Bean 类定义。

**备注**：如果已有自定义配置系统，2.0 的数据层可以完全替换，无需使用 Luban 工作流。

---

## 编辑器工具对比

### EX-GAS 1.0 编辑器

| 工具 | 位置 | 功能 |
|------|------|------|
| GAS Setting Provider | ProjectSettings → EX-GAS | 基础路径配置、子目录生成 |
| GameplayTag 编辑器 | ProjectSettings → EX-GAS → Tag | 管理 Tag 树结构，生成 GTagLib |
| Attribute 编辑器 | ProjectSettings → EX-GAS → Attribute | 管理 Attribute 列表，生成代码 |
| AttributeSet 编辑器 | ProjectSettings → EX-GAS → AttributeSet | 管理 AttributeSet，生成代码 |
| GameplayEffect Inspector | Asset Inspector | 通过 Odin 的 Inspector 配置 GE |
| Ability Inspector | Asset Inspector | 通过 Odin 的 Inspector 配置 Ability |
| Timeline Ability 编辑器 | 独立窗口 | 可视化时间轴技能编辑 |
| GAS Asset Aggregator | 独立窗口 | 汇总显示所有 GAS 资产 |
| GAS Runtime Watcher | 独立窗口 | 运行时监控 ASC 状态 |

### EX-GAS 2.0 编辑器（新增/变更）

| 工具 | 位置 | 功能 |
|------|------|------|
| **GASCenter Window** | EXTool → EX-GAS → GASCenter | **新增**：统一的中心管理窗口，集成所有配置入口 |
| **WebEditor（ASC）** | GASCenter 内部 | **新增**：Python 本地 HTTP 服务器驱动的 Web 编辑器 |
| **WebEditor（Attribute）** | GASCenter 内部 | **新增**：属性 Web 编辑器 |
| **WebEditor（AttributeSet）** | GASCenter 内部 | **新增**：属性集 Web 编辑器 |
| **WebEditor（Effect）** | GASCenter 内部 | **新增**：游戏效果 Web 编辑器 |
| **WebEditor（Tag）** | GASCenter 内部 | **新增**：标签 Web 编辑器 |
| **Luban 配置目录部署** | EXTool → EX-GAS | **新增**：一键从云端拉取 Luban 配置模板 |
| **BeanUpdater** | 编辑器代码 | **新增**：根据配置表自动同步/生成 Bean 参数类 |
| **CodeGenerator** | 编辑器代码 | **新增**：Tag/Attribute/AttributeSet 代码生成（数据表驱动） |
| Timeline Ability 编辑器 | 独立窗口 | 保留并增强 |
| GAS Runtime Watcher | 独立窗口 | 保留 |
| **DebugDrawTool** | 编辑器 | **新增**：可视化调试绘制工具 |

> 1.0 中基于 ScriptableObject 的 Tag/Attribute/AttributeSet/Effect Inspector 编辑器在 2.0 中已被数据表驱动的编辑器取代。

---

## 核心模块对比

### GameplayTag

| 特性 | 1.0 | 2.0 |
|------|-----|-----|
| 配置方式 | ProjectSettings 中的 Tag 树编辑器（ScriptableObject） | Excel 配置表（`#exgas.gameplaytags.xlsx`）或 WebEditor |
| 代码生成 | `GTagLibGenerator.cs` | `CodeGenerator.cs`（数据表驱动） |
| 运行时存储 | `GameplayTagAggregator`（C# 对象） | ECS Component（在 AbilitySystemCell 中） |
| Tag 容器 | `GameplayTagContainer`、`GameplayTagSet` | 保留，优化了底层存储 |

### Attribute 与 AttributeSet

| 特性 | 1.0 | 2.0 |
|------|-----|-----|
| 配置方式 | ProjectSettings → Attribute/AttributeSet 编辑器 | Excel 配置表或 WebEditor |
| Attribute 数据 | `AttributeValue`（C# struct） | `CAttributeData`（ECS Component） |
| 脏值标记 | 无显式脏标记 | `CAttributeIsDirty`（ECS Component，用于 System 按需计算） |
| AttributeSet | `AttributeSet`（ScriptableObject） | `AttrSetConfig`（配置类）+ `BEAttrSet`（ECS Buffer） |
| 聚合器 | `AttributeAggregator` | 保留，并结合 ECS System 处理 |
| 值钳制 | 1.1.6 版本添加 | 保留并优化 |
| Stacking 计算 | `AttrBasedWithStackModCalculation` | 保留 |

### GameplayEffect（GE）

| 特性 | 1.0 | 2.0 |
|------|-----|-----|
| 配置方式 | ScriptableObject `.asset` 文件 | Excel 配置表或 WebEditor |
| 运行时存储 | `GameplayEffectContainer`（C# List） | `BGameplayEffect`（ECS Buffer） |
| GE 数据类 | `GameplayEffectData`、`IGameplayEffectData` | 配置类（JSON Bean）+ ECS Component |
| Modifier 计算 | `GameplayEffectModifier`、各类 MMC ScriptableObject | `XParam` 参数系统 + MMC Bean |
| Stacking | `GameplayEffectStacking` | 保留，整合进 ECS System |
| Period Ticker | `GameplayEffectPeriodTicker` | 整合进 ECS Timer System（`SGlobalTimer`） |
| Granted Ability | `GrantedAbilityFromEffect` | `CGrantedByEffect`（ECS Component） |
| 冷却追踪 | `CooldownTimer` | 整合进 ECS System |

### GameplayCue

| 特性 | 1.0 | 2.0 |
|------|-----|-----|
| 配置方式 | ScriptableObject `.asset` 文件 | Excel 配置表或 WebEditor |
| 基类 | `GameplayCue`、`GameplayCueDurational`、`GameplayCueInstant` | `GameplayCueBase<T>`（泛型，绑定 XParam） |
| 内置实现 | `CueAnimation`、`CueAnimationOneShot`、`CuePlaySound`、`CueVFX`、`CueAnimationSpeedModifier` | 保留，重构为泛型+XParam 方式 |
| 参数传递 | `GameplayCueParameters` | `XParam` 参数系统（`[BeanField]` 标注字段自动序列化） |

### Ability

| 特性 | 1.0 | 2.0 |
|------|-----|-----|
| 配置方式 | ScriptableObject `.asset` 文件 | Excel 配置表 |
| 基类 | `AbstractAbility`（ScriptableObject）继承 | `AbstractAbility` + `AbilityLogicBase<T>` 组合 |
| 逻辑扩展方式 | 继承 `AbstractAbility`，重写方法 | **AbilityLogic** 系统：通过 ECS Component 组合模块化逻辑 |
| ECS 静态组件 | 无 | `BAbility`、`CAbilityBaseInfo`、`CAbilityCooldown`、`CAbilityCost`、`CAbilityActivationRequiredTags` 等 |
| ECS 动态组件 | 无 | `CAbilityActive`、`CAbilityInTryActivate`、`MCGrantedAbilityRuntime` 等 |
| Timeline Ability | `TimelineAbility`（ScriptableObject） | `ALTimeline`（AbilityLogic）+ `TimelineAbilityAsset` |
| 任务系统 | `AbilityTaskBase`、`InstantAbilityTask`、`OngoingAbilityTask` | 保留，新增常用内置任务（`TaskApplyEffects`、`TaskDoCooldown`、`TaskDoCost`、`TaskPlayCue`、`TaskDebug`） |
| TargetCatcher | `TargetCatcherBase`（及各实现） | 保留，新增 `CatchAreaBox3D` 和 `TargetCatcherHelper` |

### AbilitySystemComponent（ASC）

| 特性 | 1.0 | 2.0 |
|------|-----|-----|
| 本质 | `MonoBehaviour`（直接管理所有 GAS 逻辑） | `MonoBehaviour` 桥接层（内部持有 ECS Entity：AbilitySystemCell） |
| 配置预设 | `AbilitySystemComponentPreset`（ScriptableObject） | `AbilitySystemCellConfig`（配置类，JSON 驱动） |
| 数据存储 | C# 集合（List/Dict）在托管堆 | ECS Component / Buffer 在 Native Memory |
| 接口 | `IAbilitySystemComponent` | 保留接口，扩展 API |
| 底层实体 | 无 | `AbilitySystemCell`（ECS Entity 封装） |
| 基础数据控制器 | 无 | `BasicDataController`（控制 ASC 的 ECS 基础数据） |

### ModifierMagnitudeCalculation（MMC）

| 特性 | 1.0 | 2.0 |
|------|-----|-----|
| 定义方式 | 继承 `ModifierMagnitudeCalculation`（ScriptableObject） | 继承 `ModMagnitudeCalculationBase<T>`（泛型，绑定 XParam） |
| 参数传递 | 直接在 ScriptableObject Inspector 中配置字段 | `XParam` 参数系统（`[BeanField]` 标注） |
| 内置实现 | `ScalableFloatModCalculation`、`AttributeBasedModCalculation`、`SetByCallerFromNameModCalculation`、`SetByCallerFromTagModCalculation`、`StackModCalculation` | 保留并重构为 XParam 方式 |
| 配置来源 | ScriptableObject 引用 | JSON 配置（Excel 导表）+ BeanField 自动映射 |

---

## 新增功能（仅 2.0）

### 1. XParam 参数系统

2.0 的核心扩展机制。所有可扩展类（MMC、GameplayCue、AbilityLogic、AbilityTask、TargetCatcher）均通过 `XParam` 泛型 + `[BeanField]` 特性绑定参数：

```csharp
// 示例：带参数的 MMC
public class MMCDamageCalc : ModMagnitudeCalculationBase<XParamDamageCalc>
{
    public override float CalculateMagnitude(/* ... */)
    {
        return XParam.BaseDamage * XParam.DamageMultiplier;
    }
}

public class XParamDamageCalc : XParamBase
{
    [BeanField(nameof(SetBaseDamage), Comment = "基础伤害")]
    public float BaseDamage { get; private set; }

    [BeanField(nameof(SetDamageMultiplier), Comment = "伤害倍率")]
    public float DamageMultiplier { get; private set; }

    public void SetBaseDamage(float v) => BaseDamage = v;
    public void SetDamageMultiplier(float v) => DamageMultiplier = v;
}
```

### 2. AbilityLogic 系统

将 Ability 的逻辑拆分为可组合的 `AbilityLogic` 模块，替代 1.0 中纯继承的方式：

```
AbilityLogicBase<T>
  ├── ALApplyEffect    // 施加 GameplayEffect
  ├── ALDebugLog       // 调试日志
  ├── ALTimeline       // 时间轴技能逻辑
  └── 自定义 AL...
```

### 3. GASCenter Window（中心管理窗口）

统一的编辑器入口，支持：
- 导入 Luban 配置模板
- 管理所有 Excel 配置表
- 批量导入 JSON 数据
- 快速跳转各模块编辑器

### 4. WebEditor

基于 Python 本地 HTTP 服务器的 Web 端编辑器，提供更友好的配置界面（适用于不熟悉 Unity Editor 的策划）：
- `ASC Editor`
- `Attribute Editor`
- `AttributeSet Editor`
- `GameplayEffect Editor`
- `GameplayTag Editor`

### 5. BeanUpdater / CodeGenerator

- **BeanUpdater**：扫描项目中的 `XParam` 类，将 `[BeanField]` 字段同步回 Luban 配置表的 Bean 定义，保持代码与表结构一致。
- **CodeGenerator**：从数据表自动生成 Tag 枚举、Attribute 常量等 C# 代码，替代 1.0 中手动维护的代码生成器。

### 6. DebugDrawTool

运行时可视化调试工具，支持在 Scene 视图中绘制 GAS 运行数据（如 TargetCatcher 区域、ASC 状态等）。

### 7. GASResourceLoader

统一的 GAS 资源加载管理器，处理 JSON 配置加载、Bean 对象池、资产引用等。

### 8. Timeline Ability 新增 XParam 支持

Timeline 中的 Track 数据现在支持 `XParam` 参数绑定（`XParamTimeline`），可从配置表中读取技能时间轴参数。

---

## 移除或弃用功能

| 功能 | 1.0 | 2.0 状态 |
|------|-----|---------|
| `GameplayTagsAsset.cs` | 独立的 Tag 资产编辑器 | ❌ 移除，改为数据表 |
| `AttributeAsset.cs`（Editor） | Attribute ScriptableObject 编辑器 | ❌ 移除，改为数据表 |
| `AttributeSetConfigEditorWindow.cs` | AttributeSet 编辑窗口 | ❌ 移除，改为数据表 |
| `ModifierConfigEditor.cs` | Modifier Inspector 编辑器 | ❌ 移除，改为 XParam |
| `AbilityOverview.cs` | Ability 总览编辑器 | ❌ 移除 |
| `GASProjectSettings`（旧编辑器） | ProjectSettings 中的 GAS 设置 | ❌ 重构，移入 GASCenter |
| `GASTextDefine.cs` | 纯文本常量定义 | ❌ 替换为 `GASConstDefine.cs`（更丰富的常量） |
| `GTagLibGenerator.cs` | Tag 代码生成器（旧版） | ❌ 替换为 `CodeGenerator.cs` |
| `TagEditorUntil.cs` | Tag 编辑工具（旧版） | ❌ 移除 |

---

## 迁移建议

### 从 1.0 迁移到 2.0

> ⚠️ **警告**：1.0 到 2.0 是破坏性升级，**无法自动迁移**，需要重新配置所有 GAS 数据。

**迁移步骤参考**：

1. **备份项目**：在新分支或副本中操作。
2. **升级 Unity**：确保使用 Unity 2022.3+。
3. **安装 Entities 1.2.3**：通过 Package Manager 安装 `com.unity.entities@1.2.3`。
4. **导入 EX-GAS 2.0**：替换 `Assets/GAS` 文件夹。
5. **删除旧 ScriptableObject 配置资产**：清理 1.0 生成的 `.asset` 文件和 `ProjectSettings` 中的旧配置。
6. **部署 Luban 配置模板**：通过 GASCenter → 导入模板，一键部署 Excel 配置工程目录。
7. **重新配置数据**：在 Excel 中填写 Tag、Attribute、AttributeSet、Effect、Ability 等配置。
8. **重构自定义 Ability/MMC/Cue 代码**：按照 `AbilityLogicBase<T>` 和 XParam 方式重写。
9. **重新绑定 ASC 预设**：根据 2.0 的 `AbilitySystemCellConfig` 格式重新配置。

### 是否应该升级到 2.0？

| 场景 | 建议 |
|------|------|
| 新项目，使用 Unity 2022.3+ | ✅ 推荐使用 2.0 |
| 已有 1.0 项目，运行稳定 | ⚠️ 暂时保持 1.0，等待 2.0 更加成熟 |
| 需要高性能（大量单位、帧同步等） | ✅ 强烈推荐升级到 2.0（DOTS 性能优势明显） |
| 项目体量小，不需要复杂配置工作流 | 🔄 两个版本均可，1.0 更简单 |
| 策划需要独立编辑数据（非程序员） | ✅ 推荐 2.0（Excel + WebEditor 更友好） |
| 使用 Unity 2021 或更旧版本 | ❌ 请继续使用 1.0 |

---

## 参考资源

- [EX-GAS 1.0 分支](https://github.com/JiaMian1107/gameplay-ability-system-for-unity/tree/EX-GAS-1.0)
- [EX-GAS 2.0 分支](https://github.com/JiaMian1107/gameplay-ability-system-for-unity/tree/EX-GAS-2.0)
- [EX-GAS 1.0 CHANGELOG](https://github.com/JiaMian1107/gameplay-ability-system-for-unity/blob/EX-GAS-1.0/Assets/GAS/CHANGELOG.md)
- [EX-GAS 2.0 CHANGELOG](https://github.com/JiaMian1107/gameplay-ability-system-for-unity/blob/EX-GAS-2.0/Assets/GAS/CHANGELOG.md)
- [Luban 配置工具文档](https://www.datable.cn/docs/intro)
- [UE GAS 文档（中文）](https://github.com/BillEliot/GASDocumentation_Chinese)
