# 修复 Prompt：客户端事件订阅类缺失 `Dist.CLIENT` 导致专用服务器崩溃

## 背景

Minecraft 1.21.1 / NeoForge 21.1.233 模组 Emendatus Enigmatica（`emendatusenigmatica`）在**专用服务器（DEDICATED_SERVER）**启动时崩溃。

崩溃关键信息：

```
java.lang.RuntimeException: Attempted to load class net/minecraft/client/renderer/entity/LivingEntityRenderer for invalid dist DEDICATED_SERVER
    at net.neoforged.fml.javafmlmod.AutomaticEventSubscriber.inject(...)
Mod loading issue for: emendatusenigmatica
```

## 根因

FML 的 `AutomaticEventSubscriber` 在专用服务器上会加载**所有** `@EventBusSubscriber` 标注的类。当被加载的类在方法签名或方法体中引用了 Mojang 客户端专属类（带 `@OnlyIn(Dist.CLIENT)`，如各类 `Renderer`、`Model`、`BlockColor`、`ItemColor`），`RuntimeDistCleaner` 检测到在 `DEDICATED_SERVER` 上加载客户端类，立即抛异常，导致模组加载失败。

`events` 包内共 6 个订阅类，只有 `ShieldTextureEvent` 正确写了 `value = Dist.CLIENT`，其余 5 个均缺失。这些类在语义上全部是纯客户端逻辑（渲染层、颜色处理器、客户端扩展），本就不应在专用服务器上注册。

## 任务

为以下 5 个事件订阅类的 `@EventBusSubscriber` 注解补上 `value = Dist.CLIENT`，与已正确的 `ShieldTextureEvent`（`@EventBusSubscriber(modid = Reference.MOD_ID, value = Dist.CLIENT)`）保持一致。

修改前请逐个确认该文件已 import `net.neoforged.api.distmarker.Dist`；若未 import 则补充该 import。

| 文件 | 行号 | 当前内容 | 改为 |
|------|------|----------|------|
| `src/main/java/com/ridanisaurus/emendatusenigmatica/events/ArmorTextureEvent.java` | 48 | `@EventBusSubscriber(modid = Reference.MOD_ID)` | `@EventBusSubscriber(modid = Reference.MOD_ID, value = Dist.CLIENT)` |
| `src/main/java/com/ridanisaurus/emendatusenigmatica/events/PatreonRewardEvent.java` | 36 | `@EventBusSubscriber(modid = Reference.MOD_ID)` | `@EventBusSubscriber(modid = Reference.MOD_ID, value = Dist.CLIENT)` |
| `src/main/java/com/ridanisaurus/emendatusenigmatica/events/BlockColorEvent.java` | 38 | `@EventBusSubscriber(modid = Reference.MOD_ID)` | `@EventBusSubscriber(modid = Reference.MOD_ID, value = Dist.CLIENT)` |
| `src/main/java/com/ridanisaurus/emendatusenigmatica/events/ItemColorEvent.java` | 43 | `@EventBusSubscriber(modid = Reference.MOD_ID)` | `@EventBusSubscriber(modid = Reference.MOD_ID, value = Dist.CLIENT)` |
| `src/main/java/com/ridanisaurus/emendatusenigmatica/events/ClientExtensionsEvent.java` | 11 | `@EventBusSubscriber(modid = Reference.MOD_ID)` | `@EventBusSubscriber(modid = Reference.MOD_ID, value = Dist.CLIENT)` |

### 风险优先级（仅供参考，5 处建议全部修复）

1. **必修（当前崩溃源头）** — `ArmorTextureEvent`：方法签名 `addRenderLayer(LivingEntityRenderer<T, M> render, ...)` 直接引用 Mojang 客户端类。
2. **极可能是下一个崩溃点** — `PatreonRewardEvent`：方法体引用 `PlayerRenderer`。
3. **高风险** — `BlockColorEvent` / `ItemColorEvent`：`new BlockColorHandler()` / `new ItemColorHandler()`，handler 实现 Mojang 客户端接口 `BlockColor` / `ItemColor`。
4. **高风险** — `ClientExtensionsEvent`：注册的 `ShieldClientExtension` 引用 Mojang 客户端类 `BlockEntityWithoutLevelRenderer`。

## 约束

- 只修改上述 5 个文件的注解（以及必要的 `Dist` import），不要改动其他逻辑。
- 不要触碰 git status 中正在开发的 `EmendatusEnigmatica.java`、`EEPluginLoader.java`、`ModelLoader.java`，它们与本次崩溃无关。
- 保持每个文件原有的代码风格、缩进与 license 头。
