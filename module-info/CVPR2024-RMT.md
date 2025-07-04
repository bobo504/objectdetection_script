# RMT Block模块详细分析 https://arxiv.org/pdf/2309.11523

## 1. 背景

### Vision Transformer的局限性
传统的Vision Transformer (ViT)存在两个核心问题：
- **缺乏显式空间先验**：Self-Attention机制本身不具备空间位置感知能力[1]
- **二次计算复杂度**：全局信息建模时Self-Attention的计算成本随token数量二次增长[1][2]

### 现有解决方案的不足
现有方法如Swin Transformer使用窗口操作、NAT改变感受野形状等，虽然能部分解决问题，但都会破坏空间先验信息的完整性[2][6]。

### RetNet的启发
RetNet在NLP领域使用基于距离的时间衰减矩阵为一维单向文本数据提供显式时间先验，这为视觉领域的改进提供了灵感[2][3]。

## 2. 模块原理

### RMT Block整体架构
根据图3所示，RMT Block包含以下核心组件[7]：
- **Layer Normalization (LN)**
- **Manhattan Self-Attention (MaSA)**
- **Depth-wise Convolution (DWConv 3×3)**
- **Feed-Forward Network (FFN)**

### Manhattan Self-Attention (MaSA)核心原理

#### 空间衰减矩阵设计
MaSA的核心是基于曼哈顿距离的二维双向空间衰减矩阵：
```
D²d_nm = γ^(|xn-xm|+|yn-ym|)
```
其中：
- `(xn, yn)`表示第n个token的二维坐标
- `γ`是衰减参数，控制距离衰减的强度
- 距离越远的token，注意力权重衰减越大[5]

#### MaSA计算公式
```
MaSA(X) = (Softmax(QK^T) ⊙ D²d)V
```
这里`⊙`表示逐元素相乘，空间衰减矩阵直接调制注意力权重[5]。

#### 注意力分解机制
为了降低计算复杂度，MaSA采用沿图像两个轴的分解形式：
```
AttnH = Softmax(QHK^T_H) ⊙ DH
AttnW = Softmax(QWK^T_W) ⊙ DW
MaSA(X) = AttnH(AttnWV)^T
```
其中：
- `DH_nm = γ^|yn-ym|`表示垂直方向距离
- `DW_nm = γ^|xn-xm|`表示水平方向距离[6][7]

### 局部上下文增强 (LCE)
为了进一步增强局部表达能力，RMT Block集成了局部上下文增强模块：
```
Xout = MaSA(X) + LCE(V)
```
LCE使用5×5深度卷积来增强局部特征[7]。

### 多头注意力的衰减参数设计
不同注意力头使用不同的γ值来控制感受野，使模型能够感知多尺度信息。对于第i个头：
```
γi = 1 - 2^(-a - (b-a)i/N)
```
其中a、b控制感受野范围，N是头的总数[19]。

## 3. 解决了什么问题

### 问题1：显式空间先验缺失
**解决方案**：通过曼哈顿距离的空间衰减矩阵，为每个token提供了明确的空间位置感知能力。
- 近距离token获得更高注意力权重
- 远距离token注意力权重按距离衰减
- 提供了比传统位置编码更丰富的空间先验信息[3][5]

### 问题2：二次计算复杂度
**解决方案**：通过注意力分解将复杂度从O(N²)降低到O(N)。
- 分别计算水平和垂直方向的注意力
- 保持了与原始MaSA相同的感受野形状
- 不破坏空间衰减矩阵的完整性[6][7]

### 问题3：全局与局部信息平衡
**解决方案**：通过分阶段使用不同形式的MaSA实现最优平衡。
- 前三个阶段使用分解的MaSA处理大量token
- 最后阶段使用完整MaSA进行精细建模
- LCE模块补充局部特征表达[7]

### 实验验证效果
消融实验证明了各组件的有效性：
- **MaSA vs Vanilla Attention**：分类准确率提升0.8%，检测AP提升2.5%[15]
- **分解形式的效率**：在保持性能的同时显著降低FLOPs[16]
- **多任务优越性**：在图像分类、目标检测、实例分割和语义分割任务上都取得了最先进的结果[8][10][13][14]

通过这些创新设计，RMT Block成功地将RetNet的时间建模能力扩展到空间域，为视觉Transformer提供了一个既高效又具有强空间感知能力的核心模块。