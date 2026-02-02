# Minecraft TNT 水流同步异常问题分析

大家好，这里是清辞。今天我们来深入分析一个 Minecraft 中 TNT 实体在水流中移动时的渲染异常问题。

---

## 🔍 问题现象

### 实验观察  
通过 `/data get entity @e[type=tnt]` 和 `/tick freeze` 命令结合 Carpet Mod 的 `/log tnt` 分析，发现：

1. **服务端实际状态**：TNT在水流末端已完全静止（Pos 数据无变化，与 Carpet 显示的最终爆点相同）  
2. **客户端视觉表现**：TNT 继续向前“滑动”，然后突然回位

### 三种典型情况分类

#### **情况一：TNT 竖直落下（水流末端方块完整）**  
```
✅ 渲染正常  
TNT → 水流 → 完整方块底部  
客户端显示与实际位置一致
```

#### **情况二：水流末端为侧面碰撞箱不完整方块**  
```
❌ 渲染异常  
TNT → 水流 → 不完整碰撞箱（如台阶、楼梯）  
表现：TNT“飘出”第一个碰撞箱 → 碰到最近的碰撞箱后不久“回弹”
```

#### **情况三：水流末端为侧面碰撞箱完整方块**  
```
❌ 渲染异常  
TNT → 水流 → 完整碰撞箱  
表现：TNT“飘出”该方块 → 一段时间后“回弹”到正确位置
```

---

## 🕵️ 初步分析  
基于以上观察，初步判断为**客户端渲染和预测问题**。

---

## 🔬 源码分析

### 1. 客户端渲染机制

查看 `net.minecraft.client.renderer.entity.EntityRender.java`：

```java
public void extractRenderState(final T entity, final S state, 
                               final float partialTicks) {
    state.x = Mth.lerp((double)partialTicks, entity.xOld, entity.getX());
    state.y = Mth.lerp((double)partialTicks, entity.yOld, entity.getY());
    state.z = Mth.lerp((double)partialTicks, entity.zOld, entity.getZ());
}
```

#### 关键机制解释：
- `Mth.lerp()`：线性插值函数  
- `entity.xOld`：上一游戏刻的位置  
- `entity.getX()`：当前游戏刻的位置  
- `partialTicks`：时间比例因子（0~1 表示插值，>1 表示外推）

#### 插值运算的物理意义：
```
渲染位置 = 上一刻位置 + (当前位置 - 上一刻位置) × 时间比例

当 partialTicks > 1 时：外推（extrapolation）
客户端用当前运动趋势预测未来位置
```

> **注**：Minecraft 的移动模型实际上是 `pos₂ = pos₁ + motion`，其中 motion 是速度向量的0.05倍。

### 2. 客户端数据包处理

查看 `net.minecraft.client.multiplayer.ClientPacketListener.java`：

#### 速度同步处理：
```java
public void handleSetEntityMotion(final ClientboundSetEntityMotionPacket packet) {
    Entity entity = this.level.getEntity(packet.getId());
    if (entity != null) {
        entity.lerpMotion(packet.getMovement()); // 平滑更新速度
    }
}
```

#### 位置纠正机制：
```java
public void handleEntityPositionSync(final ClientboundEntityPositionSyncPacket packet) {
    // 小误差平滑纠正
    if (this.level.isTickingEntity(entity) && !tooBigToInterpolate) {
        entity.moveOrInterpolateTo(pos, yRot, xRot);
    } else {
        // 大误差直接闪回
        entity.snapTo(pos, yRot, xRot);
    }
}
```

这个纠正机制完美解释了观察到的“回弹”现象：
- 小位置偏差 → 平滑移动纠正  
- 大位置偏差 → 直接闪回纠正

### 3. 服务端同步决策

查看 `net.minecraft.server.level.ServerEntity.java`：

#### 关键同步逻辑：
```java
// 位置变化阈值检测
boolean positionChanged = 
    positionCodec.delta(currentPosition).lengthSqr() >= 7.6293945E-6F;// 约0.00276格的平方
boolean pos = positionChanged || this.tickCount % 60 == 0;// 强制更新触发条件
...
if (positionChanged || this.tickCount % 60 == 0) {
    // 发送位置更新包
}
```

#### 重要常量：
```java
public static final int FORCED_POS_UPDATE_PERIOD = 60; // 强制更新周期
public static final int FORCED_TELEPORT_PERIOD = 400;
private static final double TOLERANCE_LEVEL_POSITION = (double)7.6293945E-6F;
```


#### 决策流程：
1. 检查位置变化是否超过阈值（约0.00276格）  
2. 未超过阈值且不是第 60 tick → 不发送位置更新包  
3. 实体静止时，服务端**停止发包**  
4. 每 60 tick 强制发送一次位置更新包  
5. 每 400 tick 强制发送一次完整位置同步包（防累积误差）

---

## 🆚 对比分析：为什么其他实体正常？

### 掉落物（ItemEntity）的不同处理：

查看 `net.minecraft.world.entity.item.ItemEntity.java` 中的构造与同步机制：

```java
public ItemEntity(final Level level, final double x, final double y, final double z, 
                  final ItemStack itemStack, final double deltaX, final double deltaY, 
                  final double deltaZ) {
    this(EntityType.ITEM, level);
    this.setPos(x, y, z);
    this.setDeltaMovement(deltaX, deltaY, deltaZ); // 初始速度设置
    this.setItem(itemStack);
}
```

#### ItemEntity.tick() 中的关键代码：
```java
if (!this.level().isClientSide()) {
    double value = this.getDeltaMovement()
                      .subtract(oldMovement).lengthSqr();
    if (value > 0.01) {           // 速度变化检测
        this.needsSync = true;    // 设置同步标志
    }
}
```

### TNT 的简化实现：

查看 `net.minecraft.world.entity.item.PrimedTnt.java` 中的移动处理：

```java
public PrimedTnt(final Level level, final double x, final double y, final double z, 
                 final @Nullable LivingEntity owner) {
    this(EntityType.TNT, level);
    this.setPos(x, y, z);
    double rot = level.random.nextDouble() * (double)((float)Math.PI * 2F);
    this.setDeltaMovement(-Math.sin(rot) * 0.02, (double)0.2F, -Math.cos(rot) * 0.02);
    this.setFuse(80);
    this.xo = x;
    this.yo = y;
    this.zo = z;
    this.owner = EntityReference.of(owner);
}
```

#### TNT 的缺失机制：
- `PrimedTnt` 类仅有约 200 行代码  
- **缺少 `needsSync` 标志设置**  
- **缺少速度变化检测**  
- **依赖被动同步，无主动触发**

---

## 🎯 根本原因

### 核心问题：
TNT 实体缺少**主动同步触发机制**，导致：
1. 服务端在水流末端停止发包  
2. 客户端继续外推运动  
3. 60 tick 后强制纠正产生视觉异常

### 验证实验：
将测试时间调整为 60 tick，观察到 TNT **刚好在此时归位**，验证了强制更新周期的存在。

---

## 🛠️ 解决方案

### 添加位置更新标签 ✅  
为 TNT 添加类似掉落物的同步机制  
- **实现**：在 `PrimedTnt.tick()` 中添加速度变化检测  
- **应用**：已在 Carpet HFUT Additions 中实现  

### 注意事项：
不推荐在珍珠炮等大量使用 TNT 的场景启用此修复，可能增加网络负载。

---

## 💎 总结

### 问题本质：
这是一个**客户端-服务端状态同步不一致**的问题，根源在于 TNT 实体的简化实现遗漏了关键的速度同步触发机制。

### 技术要点：
1. 客户端使用插值/外推进行平滑渲染  
2. 服务端对微小位置变化不发送更新包  
3. TNT 缺少主动同步触发，依赖 60 tick 强制更新  
4. 强制更新时的纠正产生视觉“回弹”  
5. 其他实体（如掉落物）通过 `needsSync` 机制避免此问题

---

## 📚 参考资料

- 视频文字版：GitHub 与 Bilibili 专栏  
- Minecraft 1.21.11 未混淆源码  
- Fabric Carpet Mod 测试数据 

---

**感谢观看！如果喜欢这个分析，请关注我获取更多技术内容。**


---



