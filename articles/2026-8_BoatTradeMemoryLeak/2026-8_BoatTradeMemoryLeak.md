# Minecraft 假人 relogin 与地图展示内存泄漏分析

## 摘要

本文分析一个运行于 Fabric 26.1.2 服务端的环境长期内存泄漏问题。服务器连续运行数天后内存持续攀升，TPS 下降，需定期重启才能回落。通过堆转储（heap dump）与字节码级源码分析，定位到两个相互独立、又在对象层面互为放大的泄漏：一是假人 relogin 造成的玩家生命周期泄漏，旧化身因断开流程被跳过而永远无法离场；二是地图更新名单（`MapItemSavedData.carriedBy`）泄漏，挂框地图的帧广播持续登记全服玩家，而清理循环被 lithium 的 framed_maps 优化意外旁路，滚木假人条目只进不出。堆分析实测两张地图相关数据结构共保留约 1.06 GB 内存。修复以自研 mod `someshitleakfix`（4 个 Mixin）从数据入口与玩家生命周期两个方向兜底。

> 适用环境：Fabric 26.1.2 + Carpet + Carpet-Org-Addition（org）+ Syncmatica
> 修复方案：自研 mod `someshitleakfix`（4 个 Mixin，修复 2 个泄漏）

---

## 1 问题现象

### 1.1 服务器表现

- 内存占用持续爬升，TPS 逐渐下降，最终 OOM 或依赖定时重启维持；
- 假人频繁执行 `/player xxx relogin` 的机器区域，内存上涨尤为明显；
- 重启后内存回落，运行一段时间后再次上涨。

### 1.2 堆分析（MAT）实测数据

对运行数小时的服务器进行堆转储分析：

| 数据项                                  | 数值                     |
|--------------------------------------|------------------------|
| 主嫌疑地图 map 79（坐标 -1408, -3200，locked） | `carriedBy` 残留 1,644 条 |
| 全堆 `HoldingPlayer`                   | 10,070 个               |
| `SavedDataStorage` 保留堆               | 1.06 GB                |

其中 9,993 个 `HoldingPlayer` 引用的玩家对象已处于 `isRemoved()` 状态（滚木假人条目），77 个仍存活。滚木假人条目去重后对应 1,711 个假人对象（本服假人共 1,715 个，其中 1,710 个已移除、5 个仍在世界），即每个假人平均被约 6 张地图各登记一次。所有滚木假人条目引用的假人 `removalReason` 均为同一枚举常量（同一条移除路径，非null），说明它们曾被正常移出世界、满足 vanilla 清理条件，但从未被地图名单清除。

---

## 2 总体结论

堆中滞留的是同一批"滚木假人对象"，但它们被多条互不相同的强引用路径同时锚定，对应两个独立的泄漏根因：

| 泄漏                 | 根因                                                   | 强引用锚点                                                                |
|--------------------|------------------------------------------------------|----------------------------------------------------------------------|
| 泄漏一：relogin 生命周期泄漏 | org 的 `ReLoginTask` 在假人 `isRemoved()` 后提前返回，断开流程从未执行 | `PlayerList.players`、Syncmatica 的 `playerMap`、`PlayerList.save` 反复写档 |
| 泄漏二：地图更新名单泄漏       | 帧广播持续注册全服玩家，清理循环被 lithium 篡改后只检查当前玩家                 | `MapItemSavedData.carriedByPlayers` / `carriedBy`                    |

两条泄漏互为放大：泄漏一以约 4 次/秒的频率批量制造"新玩家对象实例"；泄漏二将每个上线假人逐张地图登记进持图人名单，且滚木假人条目无法清除。两者叠加使堆中对象数量呈单调累积。

---

## 3 泄漏一：假人 relogin 生命周期泄漏

### 3.1 机制

`ReLoginTask`（`boat.carpetorgaddition.periodic.task.schedule.ReLoginTask`）的下线流程按字节码还原如下：

```java
server.schedule(new TickTask(server.getTickCount(), () -> {
    if (fakePlayer.isRemoved()) {
        return;   // 缺陷：假人已被移出世界时直接返回，
                  // 后续的断开逻辑（connection.onDisconnect）永远执行不到
    }
    fakePlayer.connection.onDisconnect(new DisconnectionDetails(reason)); // 真正的断开
}));
```

org 以 `isRemoved()`（实体是否已从世界移除）作为"是否继续执行断开"的前置判断。relogin 场景下旧化身的实体先被移出世界（`isRemoved()` 置位），随后进入该 lambda，判断直接返回，断开流程被跳过。结果是旧化身既不在世界中，也未被移出 `PlayerList.players`、连接未关闭，成为一个"半断开"对象常驻服务器。每 relogin 一次产生一个。

`PlayerList.save` 进一步放大问题：其保存逻辑不区分玩家状态，仍登记在 `PlayerList` 中的已移除假人会被无差别写入存档，并进入同步侧处理流程。

org relogin 的节奏：`interval` 控制在线时长（`remainingTick` 从 `interval` 倒数到 0 触发下线），下线到重新上线为硬编码的 3 tick。`interval=1` 时完整周期约 5 tick（约 4 次/秒），旧化身由此被批量快速制造。

### 3.2 Syncmatica 的玩家注册表：第二层强引用

Syncmatica（服务端 placement 同步 mod）在服务端持有全局通信管理器 `ServerCommunicationManager`，其中维护 `playerMap: Map<ExchangeTarget, ServerPlayer>`，以强引用登记每个已加入的玩家：

```java
// ServerCommunicationManager（字节码还原）
private final Map<ExchangeTarget, ServerPlayer> playerMap;

public void onPlayerJoin(ExchangeTarget target, ServerPlayer player) { playerMap.put(target, player); }
public void onPlayerLeave(ExchangeTarget target)                     { playerMap.remove(target); }
```

注册表生命周期由 mixin 注入 vanilla 流程维护：

- `MixinPlayerManager`：在玩家管理器（`PlayerList`）的加入/移除流程中回调 `syncmatica$eventOnPlayerJoin` / `syncmatica$eventOnPlayerLeave`；
- `MixinServerPlayNetworkHandler`：实现 `IServerPlay` 接口（`syncmatica$operateComms` / `syncmatica$getExchangeTarget`），在网络处理器建立与断开时回调 `syncmatica$onConnect` / `syncmatica$onDisconnected`；
- `syncmatica$onDisconnected` 的 lambda 最终调用 `comManager.onPlayerLeave(exTarget)`，即从 `playerMap` 移除条目。

移除路径完全依赖断开流程：`syncmatica$onDisconnected` 由 `connection.onDisconnect` 触发。泄漏一中 `ReLoginTask` 提前返回后 `onDisconnect` 从未被调用，Syncmatica 的 leave 钩子随之永不触发，`playerMap` 永久保留该假人的强引用。于是同一批僵尸玩家对象至少被 `PlayerList.players` 与 Syncmatica 的 `playerMap` 两条路径同时锚定。

### 3.3 影响

旧化身对象被 `PlayerList.players` 与 Syncmatica `playerMap` 持续强引用，GC 无法回收；`PlayerList.save` 在每次自动保存时还会将这批对象反复写入存档，使其活跃状态被不断固化。

### 3.4 修复

**修复一（ReLoginTaskMixin）**：以 `@Redirect` 将 `lambda$logoutPlayer$2` 中的 `isRemoved()` 调用替换为恒定返回 `false`，强制旧化身走完断开流程：

```java
@Mixin(ReLoginTask.class)
public abstract class ReLoginTaskMixin {

    @Redirect(
            method = "lambda$logoutPlayer$2",
            at = @At(value = "INVOKE",
                     target = "Lcarpet/patches/EntityPlayerMPFake;isRemoved()Z")
    )
    private static boolean someshitleakfix$forceDisconnect(EntityPlayerMPFake fakePlayer) {
        return false;   // 使 org 认为旧假人仍在世界，继续执行真正的断开逻辑
    }
}
```

**修复二（PlayerListMixin）**：在 `PlayerList.save` 的 HEAD 注入，已移除假人直接取消保存，避免僵尸数据写入存档并进入同步侧：

```java
@Mixin(PlayerList.class)
public abstract class PlayerListMixin {

    @Inject(method = "save", at = @At("HEAD"), cancellable = true)
    private void someshitleakfix$skipSavingRemovedFakePlayer(ServerPlayer player, CallbackInfo ci) {
        if (player instanceof EntityPlayerMPFake && player.isRemoved()) {
            ci.cancel();
        }
    }
}
```

断开流程走完后，vanilla 的 `PlayerList.remove`、连接关闭以及 Syncmatica 的 leave 钩子都会正常执行，`playerMap` 与 `PlayerList.players` 中的僵尸条目被清除。

---

## 4 泄漏二：地图更新名单（carriedBy）泄漏

### 4.1 机制

挂框地图的帧广播位于 `ServerEntity.sendChanges`（按字节码还原）：

```java
// 每 tick 被调用；挂框地图分支
if (entity instanceof ItemFrame) {
    if (tickCount % 10 == 0) {                 // 每 10 tick（0.5 秒）
        ItemStack stack = frame.getItem();
        if (stack.getItem() instanceof MapItem) {
            MapItemSavedData data = MapItem.getSavedData(mapId, level);
            if (data != null) {
                for (ServerPlayer player : level.players()) {   // 遍历全服玩家，无距离限制
                    data.tickCarriedBy(player, stack, frame);   // 注册持图人
                    player.connection.send(data.getUpdatePacket(mapId, player));
                }
            }
        }
    }
}
```

`MapItemSavedData.tickCarriedBy` 同时承担注册与清理：

```java
public void tickCarriedBy(Player player, ItemStack stack, ItemFrame frame) {
    if (!carriedByPlayers.containsKey(player)) {
        carriedByPlayers.put(player, new HoldingPlayer(player)); // 注册
        carriedBy.add(...);
    }
    // 清理循环：遍历 carriedBy，移除已失效条目
    for (int i = 0; i < carriedBy.size(); i++) {
        HoldingPlayer hp = carriedBy.get(i);
        if (hp.player.isRemoved() || (frame == null && !inventory.contains(...))) {
            carriedByPlayers.remove(hp.player);
            carriedBy.remove(i);
        }
    }
}
```

vanilla 的清理设计是纯惰性的：玩家从世界移除时没有任何回调通知地图摘除其 `HoldingPlayer`，清理只在下一次 `tickCarriedBy` 被调用时顺带执行。就本场景而言，只要帧广播持续运行，清理循环应当被反复触发，`isRemoved()==true` 的僵尸条目应当被清除。

### 4.2 lithium 对清理循环的篡改（根因）

首先确认帧广播确实在持续运行。同一运行实例的两份堆转储（间隔 34 分钟，约 40,800 tick）中，7 个地图帧对应的 `ServerEntity.tickCount`（即 `sendChanges` 被调用次数）增长 7,394 至 41,052。`tickCount%10==0` 的帧地图广播分支应当周期性执行，`tickCarriedBy` 的清理循环也应当反复运行，但僵尸 `HoldingPlayer` 并未减少。该矛盾说明问题不在触发条件，而在清理循环本身被篡改。

检查 mod 层后发现，lithium（`lithium-fabric 0.24.2+mc26.1.2`）的 framed_maps 优化（`net.caffeinemc.mods.lithium.mixin.entity.framed_maps.MapItemSavedDataMixin`）通过 mixinextras 的 `@WrapOperation` 包装了 `tickCarriedBy` 清理循环内的 `List.size()` 与 `List.get(I)` 两个调用。其处理逻辑（字节码还原）为：

```java
// @WrapOperation(method = "tickCarriedBy(...)", at = @At(target = "Ljava/util/List;size()I"))
private int sizeOrOne(List<HoldingPlayer> instance, Operation<Integer> original,
                      ItemStack mapStack, ItemFrame placedInFrame) {
    if (placedInFrame != null) return 1;   // 帧广播路径：循环只执行 1 次
    return original.call(instance);        // 其余路径：保持原逻辑
}

// @WrapOperation(method = "tickCarriedBy(...)", at = @At(target = "Ljava/util/List;get(I)Ljava/lang/Object;"))
private <E> E getOrGetThePlayer(List<E> instance, int i, Operation<E> original,
                                Player player, ItemStack mapStack, ItemFrame placedInFrame) {
    if (placedInFrame != null) {
        return Objects.requireNonNull(carriedByPlayers.get(player)); // 只取当前广播玩家
    }
    return original.call(instance, i);
}
```

该优化的本意是：帧广播路径只为当前玩家注册/取包，不需要遍历整张地图的全部持图人，因此将循环缩为只检查当前玩家。其副作用是：帧广播路径下的清理循环从此只检查"当前正在广播的玩家"，其他离线玩家遗留的僵尸条目永远不会被遍历到。由于这两处 `@WrapOperation` 仅命中 `placedInFrame != null` 的调用点，携带地图的玩家背包路径（`frame == null`）仍走原逻辑，不受影响。

与此同时，僵尸假人从不携带地图，`MapItem.inventoryTick` → `tickCarriedBy(player, stack, null)`（`frame == null`）这条独立清理通道对它们永不生效。两条触发清理的通道——帧广播与持图者背包 tick——对僵尸假人全部失效，其条目遂被永久滞留。

另有一个次要因素：vanilla 清理循环按索引就地 `remove` 后未回退 `i`，相邻僵尸条目会被跳过，需多轮调用才能清完。该缺陷仅拖慢清理速度，不是滞留主因。

### 4.3 影响

僵尸条目数量最终由"假人数量 × 地图数量"决定，单调累积：

| 玩家类型 | 清理可行性                                                                                      |
|------|--------------------------------------------------------------------------------------------|
| 真实玩家 | 若清理循环未被篡改，帧广播持续时 `isRemoved` 条目可被清掉；lithium 篡改后同样滞留。vanilla 层面亦有同类问题记录（见参考资料 Paper #14088） |
| 假人   | relogin 批量制造对象实例 + 从不持图（`frame == null` 通道失效）+ 帧路径清理被 lithium 旁路，条目单调累积                    |

实测：1,711 个僵尸假人 × 每张活跃地图各登记一次，形成 9,993 个 `HoldingPlayer`；每个 `HoldingPlayer` 强引用一个完整 `ServerPlayer`（背包、属性、连接、玩家数据全部被拽住），叠加 `SavedDataStorage` 中地图数据本体，构成 1.06 GB 保留堆。

堆中另 77 个 `HoldingPlayer` 引用仍存活的 5 名玩家（4 假 1 真）。其中存活假人若最终不走正常移除流程，其条目同样无法被 vanilla 清理条件（`isRemoved()==true`）命中——这是清理条件的固有盲区，数量级恒定，不构成主要增长源。

### 4.4 修复

**修复三（ServerEntityMixin）**：帧广播注册时跳过假人，从入口阻断假人进名单：

```java
@Mixin(ServerEntity.class)
public abstract class ServerEntityMixin {

    @Redirect(
            method = "sendChanges",
            at = @At(value = "INVOKE",
                     target = "Lnet/minecraft/world/level/saveddata/maps/MapItemSavedData;tickCarriedBy(Lnet/minecraft/world/entity/player/Player;Lnet/minecraft/world/item/ItemStack;Lnet/minecraft/world/entity/decoration/ItemFrame;)V")
    )
    private void someshitleakfix$skipFakePlayerRegistration(
            MapItemSavedData data, Player player, ItemStack stack, ItemFrame frame) {
        if (!(player instanceof EntityPlayerMPFake)) {
            data.tickCarriedBy(player, stack, frame);   // 只有真实玩家才注册
        }
    }
}
```

**修复四（PlayerListMapPurgeMixin）**：玩家真正断开（`PlayerList.remove`）时，遍历所有维度的地图数据，将该玩家的注册条目从 `carriedByPlayers` / `carriedBy` 中清除，并移除其装饰标记：

```java
@Mixin(PlayerList.class)
public abstract class PlayerListMapPurgeMixin {

    @Inject(method = "remove", at = @At("HEAD"))
    private void someshitleakfix$purgeMapRegistrations(ServerPlayer player, CallbackInfo ci) {
        MinecraftServer server = player.level().getServer();
        String playerName = player.getPlainTextName();
        for (ServerLevel level : server.getAllLevels()) {                    // 遍历所有维度
            SavedDataStorage storage = level.getDataStorage();
            for (Optional<SavedData> optional : ((SavedDataStorageAccessor) (Object) storage).getCache().values()) {
                if (optional.isPresent() && optional.get() instanceof MapItemSavedData data) {
                    Map<Player, Object> carriedByPlayers = ((MapItemSavedDataAccessor) (Object) data).getCarriedByPlayers();
                    Object holder = carriedByPlayers.get(player);
                    if (holder != null) {                                    // 该玩家在这张地图的名单里
                        carriedByPlayers.remove(player);                     // 从"玩家→持有者"映射移除
                        ((MapItemSavedDataAccessor) (Object) data).getCarriedBy().remove(holder); // 从名单列表移除
                        ((MapItemSavedDataAccessor) (Object) data).callRemoveDecoration(playerName); // 清掉玩家标记
                    }
                }
            }
        }
    }
}
```

`PlayerList.remove` 仅在玩家真正断线或被踢时调用，维度切换不会经过它，因此不会误清仍在线的玩家。`MapItemSavedDataAccessor` / `SavedDataStorageAccessor` 为 Mixin 访问器，用于读取 vanilla 私有字段（`carriedByPlayers`、`carriedBy`、地图数据缓存）；`HoldingPlayer` 是私有嵌套类，用 `Object` 承载引用避免直接依赖。

修复三与修复四一进一出：假人从不注册（堵源头）+ 真实玩家断开即清（根治残留）。

---

## 5 两个泄漏的耦合与验证方法

### 5.1 耦合关系

泄漏一与泄漏二在对象层面相互放大：relogin 以约 4 次/秒的频率持续制造新的玩家对象实例（泄漏一）；这些实例上线瞬间即被帧广播逐张地图登记进 `carriedBy`（泄漏二）；下线后对象进入 `isRemoved()` 状态，本可被清理，但泄漏一使其无法走完断开流程（`PlayerList.players` / Syncmatica `playerMap` 锚定），泄漏二又因 lithium 旁路而无法清除地图条目（`carriedBy` 锚定）。同一批对象被三个数据结构同时强引用，两个泄漏必须并列修复。

### 5.2 验证方法

帧广播是否"冻结"不能由单份 dump 的 `tickCount` 当前值判断，必须对比同一实例两个时点 dump 的取值。本调查中 7 个帧在 34 分钟内 `tickCount` 增长 7,394–41,052，直接证伪了"广播冻结导致清理不触发"的假设，进而把根因收敛到"清理循环被 mod 篡改"这一方向。

---

## 6 总结

1. 服务器存在两个并列的玩家对象泄漏，根因分别在 org 的 relogin 生命周期与地图更新名单的清理路径；
2. 泄漏一：`ReLoginTask` 以 `isRemoved()` 提前返回，断开流程（含 `PlayerList.remove`、Syncmatica `onPlayerLeave`）永不执行，旧化身被 `PlayerList.players` 与 Syncmatica `playerMap` 强引用；
3. 泄漏二：帧广播持续注册全服玩家，vanilla 的惰性清理本身依赖下一次 `tickCarriedBy`，但 lithium framed_maps 的 `@WrapOperation` 将清理循环缩为只检查当前广播玩家，僵尸条目被旁路；假人不持图使另一条清理通道也不适用；
4. 修复从入口与生命周期两个方向兜底：假人不注册、假人不写档、断开强制走完、断开即清地图条目；
5. 断开清理成本由地图数决定，本服 31 张地图规模下约 0.01–0.05 ms，无性能顾虑。

---

## 7 参考资料

- 修复 mod 源码：`someshitleakfix`（4 个 Mixin + 2 个 Accessor）
- [PaperMC/Paper #14088 —— MapItemSavedData.carriedByPlayers retains disconnected ServerPlayer references indefinitely](https://github.com/PaperMC/Paper/issues/14088)
- [Mojang MC-274519 / MC-277780 —— 锁定地图每次自动保存全部重写导致冻结/崩溃（24w33a / 25w02a 修复）](https://bugs.mojang.com/browse/MC-274519)
- lithium-fabric 0.24.2+mc26.1.2：`MapItemSavedDataMixin`（framed_maps，字节码还原）
- syncmatica-fabric 26.1.1：`ServerCommunicationManager` / `MixinPlayerManager` / `MixinServerPlayNetworkHandler` / `IServerPlay`（字节码还原）
- Fabric 26.1.2 服务端 + Carpet + Carpet-Org-Addition 1.45.1 源码
- Eclipse MAT 堆分析数据
- VisualVM 堆分析数据

---

## 8 特别鸣谢

Ryan100c
github (https://github.com/hotpad100c)
bilibili (https://space.bilibili.com/3546649519458603)
