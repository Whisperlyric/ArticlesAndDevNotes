# Minecraft 假人 relogin 与地图展示内存泄漏分析

大家好，这里是清辞。今天我们来深入分析一个 Minecraft 服务器上的内存泄漏问题——运营一段时间的服务器内存持续上涨、重启才回落，最终在堆分析里挖出了两个"只进不出"的玩家残留泄漏。

> 适用环境：Fabric 26.1.2 + Carpet + Carpet-Org-Addition + Syncmatica
> 修复方案：自研 mod `someshitleakfix`（4 个 Mixin，修复 2 个泄漏）

---

## 🔍 问题现象

### 服务器表现

服务器开服运行数天后：

1. **内存占用持续爬升**，TPS 逐渐下降，最终 OOM 或靠定时重启维持；
2. 假人频繁执行 `/player xxx relogin` 的机器区域，内存上涨尤为明显；
3. 重启后内存回落，但运行一段时间又会涨回去。

### 堆分析（MAT）实测数据

对服务器做堆转储（heap dump）分析，找到一张**主嫌疑地图**：

| 数据项 | 数值 |
|---|---|
| 主嫌疑地图 map 79（坐标 -1408, -3200，locked） | `carriedBy` 残留 **1,644 条** |
| 全堆 `HoldingPlayer` 对象 | **10,070 个** |
| `SavedDataStorage` 保留堆 | **1.06 GB** |

---

## 🕵️ 初步分析

堆里躺着两批"玩家残留"：

1. **假人 relogin 生命周期泄漏** —— 旧假人化身没有真正断开，赖在服务器上；
2. **地图"更新收件人名单"泄漏** —— 挂框地图每 0.5 秒把全服玩家记进名单，玩家下线后名字没人划掉。

两个问题本质相同：**玩家的对象被服务器某处数据结构强引用"钉住"，Java 的垃圾回收（GC）永远收不掉**，内存越堆越多，最终拖垮服务器。

---

## 🔬 源码分析

### 泄漏一：假人 relogin 生命周期泄漏

#### 1. org 的下线逻辑缺陷

查看 `boat.carpetorgaddition.periodic.task.schedule.ReLoginTask.java`：

```java
// ReLoginTask.logoutPlayer() 里调度的 lambda（按字节码还原）
server.schedule(new TickTask(server.getTickCount(), () -> {
    if (fakePlayer.isRemoved()) {
        return;   // ← 缺陷：假人已被标记 removed 时直接跳过，
                  //   下面的"真正断开连接"逻辑永远执行不到
    }
    fakePlayer.connection.onDisconnect(new DisconnectionDetails(reason)); // 真正的断开
}));
```

`isRemoved()` 是假人"是否已从世界移除"的标志。org 的下线流程依赖这个判断来决定是否继续执行断开，但 relogin 场景下旧化身恰恰处于"半标记"状态——检查把它拦下了，断开流程却从未真正发生。

**结果**：旧假人化身既没被移出世界，也没断开连接——一个"半死不活"的旧化身永远赖在服务器里。每 relogin 一次就多赖一个。

#### 2. 无差别存档

查看 `net.minecraft.server.players.PlayerList.java` 的 `save` 方法：

```java
public void save(ServerPlayer player) {
    // 无差别把玩家数据写入存档 / 同步给 Syncmatica，
    // 已移除的假人也照存不误 → "尸体"被写进存档并继续被引用
}
```

#### org relogin 的实际节奏

`interval` 控制的是"在线时长"——`remainingTick` 从 `interval` 倒数到 0 即触发下线；而下线到重新上线是硬编码的 3 tick（`canSpawn` 2→1→0 后执行登录），与 interval 无关。`interval=1` 时完整周期约 5 tick（≈4 次/秒，在线约 2 tick）。假人机器按这个频率反复 relogin，旧化身就是这样被一批批快速制造出来的。

---

### 泄漏二：地图"更新收件人名单"泄漏

#### 1. 挂框地图的广播注册

查看 `net.minecraft.server.level.ServerEntity.java` 的 `sendChanges`：

```java
// ServerEntity.sendChanges() —— 每 tick 被调用，挂框地图分支
if (entity instanceof ItemFrame) {
    if (tickCount % 10 == 0) {                       // 每 10 tick（0.5 秒）
        ItemStack stack = frame.getItem();
        if (stack.getItem() instanceof MapItem) {
            MapItemSavedData data = MapItem.getSavedData(mapId, level);
            if (data != null) {
                for (ServerPlayer player : level.players()) {      // ← 遍历全服玩家，无距离限制
                    data.tickCarriedBy(player, stack, frame);      // ① 写进听众名单
                    player.connection.send(data.getUpdatePacket(mapId, player)); // ② 发更新包
                }
            }
        }
    }
}
```

玩家背包里的地图同理：玩家每 tick 对自己背包/手持的每张地图调用 `tickCarriedBy(player, stack, null)`，让自己成为持有者。

#### 2. 惰性清理缺陷

查看 `net.minecraft.world.level.saveddata.maps.MapItemSavedData.java` 的 `tickCarriedBy`：

```java
public void tickCarriedBy(Player player, ItemStack stack, ItemFrame frame) {
    if (!carriedByPlayers.containsKey(player)) {
        carriedByPlayers.put(player, new HoldingPlayer(player)); // 加入听众名单
        carriedBy.add(...);
    }
    for (HoldingPlayer hp : carriedBy) {
        if (hp.player.isRemoved() || (frame == null && !inventory.contains(...))) {
            carriedByPlayers.remove(...);   // ← 缺陷：清理只在"下次广播"时才顺手做；
                                            //   玩家离开后 tickCarriedBy 不再被调用，
                                            //   这行代码永远等不到执行
        }
    }
}
```

**关键机制解释：**

- `carriedByPlayers` 的 key 是**玩家对象实例**，不是玩家名——同一玩家每次上线都是一个新的 `ServerPlayer` 对象，都会被记一次；
- 清理是"惰性"的，且只认 `isRemoved()`，靠"下次广播时顺手检查"完成；
- 玩家下线后不在 `level.players()` 里 → 广播循环不再为他触发 → 清理代码永远轮不到他；
- 假人旧化身 `isRemoved()` 是 `false`（泄漏一未修时的状态）→ **即使清理逻辑跑起来也清不掉**。

#### 为什么数量能堆到上千

| 玩家类型 | 下线后会发生什么 |
|---|---|
| 真实玩家 | 已不在 `level.players()` 里，广播不再触发；若挂框区域一直有其他人，`isRemoved=true` 的还能被顺手清掉；一旦区域没人，就永远无人清 |
| 假人旧化身 | `isRemoved()` 是 `false`，即使清理逻辑跑起来也清不掉 |

`map 79` 的 1,644 条残留 = 1,644 个曾被注册的玩家对象实例（假人 relogin 反复制造新实例贡献了绝大部分）；全服所有地图累计出 10,070 个 `HoldingPlayer`，每个都强引用一个完整的 `ServerPlayer`（背包、属性、连接、玩家数据全被拽住），于是堆里躺着 1.06 GB 的 `SavedDataStorage`。

---

## 🎯 根本原因

### 核心问题：

地图的**收件人名单只进不出**：

1. 挂框地图每 0.5 秒把全服在线玩家写进 `carriedByPlayers`；
2. 清理逻辑只在"广播时顺手做"，且只认 `isRemoved()`；
3. 玩家离开后广播不再为他触发 → 条目永久钉死；
4. 假人 relogin 以约 4 次/秒的频率批量制造"新玩家实例"，全部被写进名单 → 数量滚雪球。

### 时间线分析：

```
t=0:   假人 relogin，新化身上线
       挂框地图广播 → 新化身被写进 carriedByPlayers（新对象实例）
t=5t:  下一次 relogin，又一个新化身上线
       又被写进名单；旧化身因 isRemoved()=false 赖在服务器
...    反复循环
N天后: 名单累积上千条残留，每个残留强引用一个完整 ServerPlayer
       内存只进不出 → OOM / 重启才能释放
```

同类问题已被上游确认是 vanilla 通用 bug（真实玩家也会泄漏）：[PaperMC/Paper #14088](https://github.com/PaperMC/Paper/issues/14088)。

---

## 🛠️ 解决方案

针对两个泄漏，采用"**堵源头 + 根治残留**"双保险策略，实现为 4 个 Mixin。

### 修复一：强制假人断开 + 跳过尸体存档

#### ① ReLoginTaskMixin —— 强制让断开流程走完

```java
@Mixin(ReLoginTask.class)
public abstract class ReLoginTaskMixin {

    @Redirect(
            method = "lambda$logoutPlayer$2",
            at = @At(value = "INVOKE", target = "Lcarpet/patches/EntityPlayerMPFake;isRemoved()Z")
    )
    private static boolean someshitleakfix$forceDisconnect(EntityPlayerMPFake fakePlayer) {
        return false;   // 把 isRemoved() 的结果"篡改"成 false，
                        // 让 org 认为旧假人还在线，从而继续执行真正的断开逻辑
    }
}
```

`@Redirect` 的含义：把目标方法里 `fakePlayer.isRemoved()` 这个调用整体替换成我们自己的方法，返回值固定为 `false`——旧假人终于能被正常送出门了。

#### ② PlayerListMixin —— 已移除的假人不写入存档

```java
@Mixin(PlayerList.class)
public abstract class PlayerListMixin {

    @Inject(method = "save", at = @At("HEAD"), cancellable = true)
    private void someshitleakfix$skipSavingRemovedFakePlayer(ServerPlayer player, CallbackInfo ci) {
        if (player instanceof EntityPlayerMPFake && player.isRemoved()) {
            ci.cancel();   // "尸体"不存档，不再被 Syncmatica 引用
        }
    }
}
```

### 修复二：假人不注册 + 断开即清

#### ③ ServerEntityMixin —— 假人根本不进名单

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

#### ④ PlayerListMapPurgeMixin —— 真实玩家断开立即清名单

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

`PlayerList.remove` 只在玩家**真正断线**（或被踢）时调用，维度切换不会经过它，因此不会误清仍在线的玩家。`MapItemSavedDataAccessor` / `SavedDataStorageAccessor` 是 Mixin 访问器，用来读取 vanilla 的私有字段（`carriedByPlayers`、`carriedBy`、地图数据缓存）；`HoldingPlayer` 是私有嵌套类，用 `Object` 承载引用避免直接依赖。

一进一出，双保险：**假人从不注册**（堵源头）+ **真实玩家断开即清**（根治残留）。

### 性能评估

| 项目 | 数值 |
|---|---|
| 本服 31 张地图，每次断开全量扫描 | ≈ 0.01–0.05 ms |
| Paper #14088 在 25,000 张地图下实测 | 10.9 ms/次 |
| 成本决定因素 | 地图数，而非玩家数 |

断开清理的成本由地图数决定：本服 31 张地图规模下完全无感。

---

## 💎 总结

### 问题本质：

这是两个"**玩家对象被强引用钉住、GC 收不掉**"的内存泄漏：假人 relogin 旧化身不断线 + 地图收件人名单只进不出。假人 relogin 以约 4 次/秒的频率批量制造新玩家实例，全部被写进地图名单，最终在堆里留下 1.06 GB 的残留。

### 技术要点：

1. 挂框地图每 10 tick 对全服玩家注册进 `carriedByPlayers`，注册按对象实例而非玩家名；
2. `tickCarriedBy` 的清理是惰性的且只认 `isRemoved()`，玩家离开后广播不再触发，清理永不执行；
3. org relogin 的 `isRemoved()` 提前返回导致旧化身不断线（vanilla `PlayerList.remove` 也不会被调用）；
4. 修复采用"假人不注册（堵源头）+ 断开即清（根治残留）"双保险；
5. 断开清理成本由地图数决定，本服 31 张地图规模下 ≈ 0.01–0.05 ms，无性能顾虑；
6. 同类问题已被上游确认为 vanilla 通用 bug（Paper #14088，真实玩家也会泄漏）。

---

## 📚 参考资料

- 完整修复总结（本地）：`泄漏修复总结.md`
- 修复 mod 源码：`someshitleakfix`（4 个 Mixin + 2 个 Accessor）
- [PaperMC/Paper #14088 —— MapItemSavedData.carriedByPlayers retains disconnected ServerPlayer references indefinitely](https://github.com/PaperMC/Paper/issues/14088)
- [Mojang MC-274519 / MC-277780 —— 锁定地图每次自动保存全部重写导致冻结/崩溃（24w33a / 25w02a 修复）](https://bugs.mojang.com/browse/MC-274519)
- Fabric 26.1.2 服务端 + Carpet + Carpet-Org-Addition 1.45.1 源码
- Eclipse MAT 堆分析数据

---

**感谢观看！如果喜欢这个分析，请关注我获取更多技术内容。**

---
