## 一、精度问题为什么在动画中出现

### 浮点精度的本质

`float32` 有效位约 7 位十进制。离原点越远，能表达的最小间隔越大：

| 世界坐标量级 | 可表达最小间隔（相邻 float32 之差） |
| :--- | :--- |
| 附近 1.0 | ~ $1.2 \times 10^{-7}$ |
| 附近 1,000 | ~ $1.2 \times 10^{-4}$ |
| 附近 100,000 | ~ 0.008 |
| 附近 1,000,000 | ~ 0.06 |
| 附近 10,000,000 | ~ 0.6 |

角色绑定时确实在原点 $(0,0,0)$ 附近，精度很高。但动画播放 / 演出摆放时角色被整体搬到场景里的远处——比如大世界里某个 POI 坐标 $(500,000, 0, 300,000)$ 或 cutscene 里某个剧情点的位置。此时：

* 骨骼的世界坐标数值进入“百万级”区间。
* 骨骼在每一帧的旋转要算 $\sin/\cos$ 再乘到位置上。
* 远离原点时，“骨骼相对绑定姿态的小角度扰动”（比如手指抖 1°）所产生的世界位置变化，数值上和“骨骼所在大坐标本身的量级”相差 6~7 个数量级，落进了 `float32` 精度的死区。

### 具体的数值表现

角色在原点时，手指末端世界位置可能是 $(5.123456, 170.234567, 2.345678)$。骨骼旋转 1° $\rightarrow$ 手指末端变成 $(5.087234, 170.251123, 2.366891)$ $\rightarrow$ 差异精确到毫米级，蒙皮顶点跟随的是正确的。

把同一个角色挪到 $(1,000,000, 0, 0)$：
* 手指末端世界位置变成 $(1,000,005.123, 170.234, 2.345)$。
* `float32` 在 100 万这个量级，小数点后只剩 2 位可信。
* 骨骼旋转 1° 想让手指位置变化 0.036 mm 级别 $\rightarrow$ 这个变化量比 `float32` 在当前量级的精度还小 $\rightarrow$ 计算结果要么无变化、要么跳一大步。
* **表现**：骨骼角度变了但顶点“不跟”，或者抖动、阶梯状跳变、JP (joint popping)、蒙皮麻花扭曲。

### 为什么只在“位移后”才暴露

绑定时模型在原点，`skinCluster.bindPreMatrix[i]` 被记录成“骨骼绑定世界矩阵的逆”——这是个数值量级小、精度高的矩阵。播放时骨骼被搬远，`matrix[i]` 变成大数值矩阵。蒙皮算：

$$vtx\_skinned = \sum weight[i] \times (matrix[i] \times bindPreMatrix[i]) \times vtx\_rest$$

$matrix[i] \times bindPreMatrix[i]$ 在数学上表示“骨骼相对于绑定姿态的变换”，理论上在“骨骼只是被整体平移”时应该等于纯平移矩阵。但两个量级悬殊的矩阵相乘，浮点精度大量丢失——大数值矩阵的每个元素有效位只剩两三位，乘上一个小数值矩阵，结果的低位全是噪声。

---

## 二、这套 "Inverse Node" 方案怎么解决

### 核心思想

不要让蒙皮计算去做“大 × 小”的精度杀手操作。把“整体平移”这部分变换从 $bindPreMatrix \times matrix$ 的乘积里挪出去，让蒙皮只处理“骨骼相对变化”，整体平移另外接到 transform 节点上。

### 具体做法

* **原本**：`bindPreMatrix[i]` 是静态数值，是“绑定那一刻的骨骼世界逆矩阵”。
* **改后**：给每根骨骼建一个辅助 transform `Inv_i`，让它始终等于骨骼当前世界矩阵（挂在跟骨骼同样被整体平移的地方，比如 Main 下）。
* 把 `Inv_i.worldInverseMatrix[0]` connect 到 `skinCluster.bindPreMatrix[i]`（动态连接，每帧计算）。

---

## 三、数学公式对比

### 符号约定

* $B_i$ = 骨骼 $i$ 绑定时的世界矩阵（固定）
* $M_i(t)$ = 骨骼 $i$ 在时刻 $t$ 的当前世界矩阵
* $T$ = 整体搬运矩阵（把角色从原点搬到场景位置）
* $L_i(t)$ = 骨骼 $i$ 相对于绑定姿态的“纯动画变换”（角度变化等），在原点附近、量级小
* $v$ = 顶点在绑定姿态下的世界位置（原点附近，小数值）

在“整体搬运 + 动画”的情况下：
$$M_i(t) = T \cdot L_i(t) \cdot B_i$$
即“先回到绑定姿态、再做相对动画、再整体搬到场景位置”。

---

### 方案 A：传统静态 bindPreMatrix（老的、会出精度问题）

$bindPreMatrix[i] = B_i^{-1}$（绑定时一次性算好，固定不变，数值小）

蒙皮计算：
$$v' = \sum_i w_i \cdot M_i(t) \cdot B_i^{-1} \cdot v = \sum_i w_i \cdot \big(T \cdot L_i(t) \cdot B_i\big) \cdot B_i^{-1} \cdot v = \sum_i w_i \cdot T \cdot L_i(t) \cdot v$$

代数上完全正确，但浮点计算不按代数顺序简化：
$$v' = \sum_i w_i \cdot \underbrace{M_i(t)}_{\text{量级大，百万级}} \cdot \underbrace{B_i^{-1}}_{\text{量级小}} \cdot \underbrace{v}_{\text{量级小}}$$

问题出在 $M_i(t) \cdot B_i^{-1}$ 这一步：两个量级悬殊的矩阵相乘，$M_i(t)$ 的大数值把 $B_i^{-1}$ 的低位有效数字“吃掉”——原本数学上该化简成 $T \cdot L_i(t)$（小数值），浮点实际算出来是带噪声的 $T \cdot L_i(t)$，噪声大小和 $T$ 的量级成正比。然后再乘 $v$（小数值），把噪声放大到顶点位置上 $\rightarrow$ 蒙皮抖动、麻花、炸裂。

---

### 方案 B：Inverse Node 动态 bindPreMatrix（工具加的）

让 $bindPreMatrix[i]$ 每帧动态等于骨骼当前世界逆矩阵：
$$\text{bindPreMatrix}_i(t) = M_i(t)^{-1}$$
（通过 `Inv_i.worldInverseMatrix[0]` $\rightarrow$ `skinCluster.bindPreMatrix[i]` 的 connection 实现）

此时蒙皮计算：
$$v' = \sum_i w_i \cdot M_i(t) \cdot M_i(t)^{-1} \cdot v = v$$
纯数学上是“不动”——但这不是目标。目标是让相对动画还能起作用，所以 `Inv_i` 不能真的等于 $M_i(t)$，而是等于“骨骼去掉相对动画后的那部分”。

实际做法：`Inv_i` 只跟着 $T$（整体搬运）走，不跟着 $L_i(t)$（相对动画）走。把 `Inv_i` 挂在 Main 下且不受动画影响：
$$\text{Inv}_i = T \cdot B_i \quad \Rightarrow \quad \text{bindPreMatrix}_i(t) = (T \cdot B_i)^{-1} = B_i^{-1} \cdot T^{-1}$$

蒙皮计算：
$$v' = \sum_i w_i \cdot M_i(t) \cdot (T \cdot B_i)^{-1} \cdot v = \sum_i w_i \cdot \big(T \cdot L_i(t) \cdot B_i\big) \cdot \big(B_i^{-1} \cdot T^{-1}\big) \cdot v$$

代数化简：
$$= \sum_i w_i \cdot T \cdot L_i(t) \cdot T^{-1} \cdot v$$
这是一个漂亮的共轭形式。但到这里还没解决精度——关键在浮点计算顺序。

### 真正的精度红利来自执行顺序

Maya 里蒙皮计算是 $matrix \times bindPreMatrix$ 这步作为整体先算，才乘 $v$：
$$C_i(t) = M_i(t) \cdot \text{bindPreMatrix}_i(t)$$

**方案 A：**
$$C_i = \underbrace{M_i(t)}_{\sim 10^6} \cdot \underbrace{B_i^{-1}}_{\sim 1}$$
两个悬殊量级相乘 $\rightarrow$ 低位精度丢失。

**方案 B：**
$$C_i = \underbrace{M_i(t)}_{\sim 10^6} \cdot \underbrace{(T \cdot B_i)^{-1}}_{\sim 10^{-6}, \text{因为是大矩阵的逆}}$$
注意 $(T \cdot B_i)^{-1}$ 的平移部分是 $-T \cdot B_i$ 的量级（百万级的负数）。两个量级差不多但符号相反的大矩阵相乘——这正是浮点里的“灾难性消去”场景，看上去应该更糟？

但是：Maya 在内部做 $M \cdot bindPreMatrix$ 时，如果两个矩阵都是动态输入，它可以在更高精度（通常是 `double`）里先相乘再转回 `float`。更关键的是——因为 `Inv_i` 和 $M_i(t)$ 共享同一个父节点层级（都在 Main 下），DG 计算它们的 `worldMatrix` 时，是在相同的参考系下递归累乘的，$T$ 这部分在两者里是同一组完全一致的浮点位，相乘时 $T \cdot T^{-1}$ 这部分在数值上精确消去，不产生噪声。

换句话说：
* **方案 A** 里 `bindPreMatrix` 是绑定时一次性存下的静态 `float32` 值，和运行时的 $M_i(t)$ 在不同时间、不同状态下算出来的，浮点表示不匹配，乘起来 $T \cdot T^{-1}$ 不会精确消去。
* **方案 B** 里 `bindPreMatrix` 是运行时从同一个 Main 层级实时算出来的，和 $M_i(t)$ 里的 $T$ 部分用的是同一组浮点位，乘起来能精确消去大量级部分，只留下小量级的 $L_i(t)$ 变化量。

### 最终有效计算

方案 B 在浮点下的实际效果：
$$C_i = T \cdot L_i(t) \cdot \cancel{B_i \cdot B_i^{-1}} \cdot T^{-1} = T \cdot L_i(t) \cdot T^{-1}$$

再乘 $v$：
$$v' = \sum_i w_i \cdot T \cdot L_i(t) \cdot T^{-1} \cdot v$$

关键：$L_i(t) \cdot T^{-1} \cdot v$ 这一步把 $v$ 先从小量级空间搬到大量级空间再做小幅扰动——这里有精度损失。但工具的 setup 还做了一件事：把 `<model>_SKIN_INVERSE_NODE.worldMatrix` $\rightarrow$ `skinCluster.geomMatrix`，让 mesh 侧也在相同量级下做补偿。最终的整体效果是：蒙皮的所有关键运算都在“小量级坐标系”里完成，只在最后一步用 $T$ 把结果搬到场景位置。

这就相当于在蒙皮计算的内部建立了一个“角色局部坐标系”，计算在局部坐标系里做（原点附近、精度高），输出时才 apply 整体搬运 $T$。

---

## 四、类比理解

一个更直观的类比：方案 A 像是让人在远离原点 100 万米的地方用厘米刻度的尺子量长度——尺子精度够，但位置数字太大，读数时小数点后的厘米全被大位吃掉。方案 B 像是让人先回到原点量好（“你往左挪 1 厘米”），再一起平移到 100 万米外——测量在高精度区间做，位移和测量解耦。

---

## 五、为什么对角色同样有用（不止场景绑定）

角色做动画播放时确实常在原点附近，问题不大。但：
* 演出/cutscene 场景把角色摆到大世界具体坐标（剧情点、关卡位置），这是 RM42 cutscene pipeline 的主要使用场景。
* 多角色合批时每个角色被放在各自位置。
* 角色整体 scale（大人变小人、夸张表演）时，$T$ 里含非 1 的 scale，精度退化更敏感。

所以这套 `PreMatrix` 机制对角色绑定一样必要——不是“绑定姿态不在 (0,0,0)”，而是运行期的整体变换 $T$ 让精度出问题。