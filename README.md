# 区块查询服务（blockqs）

## 前言

将区块数据单独存储并提供必要的查询服务，可以对区块链系统中庞大存储负载进行分离。

借助于UTXO指纹的设计，当前区块所需的UTXO集合可以轻易验证。因为存在公共的区块查询服务，全节点不再必需，普通的节点就可以完成所需的验证工作。区块数据托付于单独的服务网络，或许让数据的缓存和共享也更有效率。

验证节点会存储UTXO集以及近期的区块数据，查询服务通常不会太繁忙，服务节点更多地服务于对漫长历史区块的检索。



## 区块存储

区块数据的存储按尽量分离的原则进行，即以交易数据为单元进行小文件存储。检索交易的路径结构分两个面向：简易的 `442检索`，以及基础性的输入项 `2+20+2` 来源检索。

- `4+4+2` 结构：区块高度4字节、交易在区块中的序位4字节、输出项在交易输出集中的偏移2字节。由 `GOTO/JUMP` 两个指令使用。
- `2+20+2` 结构：交易时间戳年度2字节、交易ID片段20字节、输出项偏移2字节（同上）。主要由UTXO和零散的交易输出项检索使用。

```go
// 2+20+2 检索
// TxID 使用查询键的20字节序列
2025/                           // 年度（交易）
    [TxID:0xff]/                // 一级：1字节16进制表达，前置 0x 以与日次区分
        [TxID:FF]/              // 二级：同上，但大写且无前置 0x
            [TxID:000]/         // 三级：同上1字节，但用10进制表达
                [TxID].data     // 交易数据
                [TxID].sig      // 签名数据
                [TxID].index    // 交易数据&签名的索引
                [TxID].meta     // 交易的元信息，包含交易所在区块、时间戳、哈希摘要（完整ID）等
                chksum.list     // 当前目录内各文件的校验和清单

// 442 检索
2025/                           // 年度（区块）
    001/                        // 日次：第1天
    365/                        // 日次：按实际历法计算（365|366）
        [height].head           // 交易ID&时间戳清单：按交易在区块中的顺序排列
        chksum.list             // 当日各清单文件的校验和

// 区块头链
// 作为区块的基础数据，数据节点应当有一个副本。
2025/                           // 年度（区块）
    blocks.head                 // 本年度区块头链数据
    blocks.chksum               // 上面头链数据的校验和存储

// 注：
// 上面三个年度目录结构可合并存储。易于区分。

// 附：UTXO集
// 在内存中构建的单独缓存，为当前（末端区块后）最新状态集。可选
// 此结构用于计算UTXO指纹。
2025/                           // 年度（交易）
    [TxID:0]/                   // 一级：交易ID（20字节检索段）首字节分组
        [TxID:1]/               // 二级：同上次字节分组
            [TxID:2]            // 三级：同上第三字节分组
                [TxID]:[0][...] // 末端：特定交易输出花费状态位集（会实时更新）
```

第二段的*年/日*分级用于 `GOTO/JUMP` 指令定位交易：由区块高度计算出年度和日次 => 由交易序位检索交易ID和交易时间戳 => 由 `2+20+2` 结构检索交易数据。

假设一个区块最多包含64k笔交易，`2+20+2` 结构下的三级分层里，末端目录平均会容纳343笔交易的数据，如果一笔交易4个文件，一个目录大概容纳1400个文件不到。这在普通终端上是可行的。

> **注：**
> 源于固定的出块时间，区块的时间戳可以仅通过*创始块时间*和*区块高度*计算出来。
> UTXO集只是当前最新状态缓存，各历史区块的UTXO集可由当前序列结合上一区块内容逆向逐块推导出来。


### 由交易查询所在区块

区块不收录未来的交易，因此如果知道一笔交易的ID（及其时间戳），想要查询它被收录入的区块，可以由交易的时间戳计算区块开始，向后（高度增加的方向）逐个区块查询。

又因为交易滞留在链外有过期时间，因此向后查询的区块也是有限的。通常，交易所在区块的信息会记录在它的元信息里（[TxID].meta）。



## 数据检索

- **输出 <= 交易年度+交易ID+输出偏移（2202）**：主要用于存证（不在UTXO内）的检索。
- **输出 <= 区块高度+交易序位+输出偏移（442）**：或者通过442结构查询零散的管理者脚本（`GOTO`、`JUMP`）。
- **交易 <= 交易年度+交易ID**：检索完整的交易数据，可能用于第三方App（如存证验证/发展），从交易的局部信息来检索。
- **区块 <= 区块高度**：对一个完整区块的数据检索，可能源于初始上线节点或第三方应用的某种需求（如钱包App发掘账户关联交易）。
- **区块 <= 区块ID**：对一个区块的完整获取，可能源于其它数据节点的同步或第三方应用的需求。
- **区块头链 <= 区块链名称**：主要由初始上线节点下载此基准数据，用于其它下载和验证的基准。如局部链段或UTXO集等。
- **区块头链 <= 创始块ID**：同上原因和用途。一个区块链的客户端应当硬编码其所属链的创始块ID。
- **UTXO <= 交易ID+输出偏移**：查询目标是否已花费。数据节点保有一份当前最新UTXO集的副本，但这可能只是附加服务（因为实时性不强）。
- **UTXO集 <= 当前时间戳**：获取最新UTXO集。主要为新上线的校验组初始化UTXO缓存（校验组也可以向同类节点请求）。
- **区块过滤器 <= 区块高度**：获取目标高度区块的过滤器（BIP_157/158: Compact Block Filters）。主要用于账户所有者核实&查找自身账户地址相关的交易。

> **提示：**
> 区块头链数据从数据网络节点获取，但严格的UTXO集应从同类兄弟节点处同步（实时性更好）。
> 因为区块头内包含UTXO指纹，所以从兄弟节点传递过来的UTXO数据很容易被验证。



## UTXO指纹

UTXO是区块链所有未花费输出的集合，为了对当前UTXO集可以进行验证，我们添加了UTXO指纹设计。


### UTXO指纹结构图

<img src="docs/utxohash-1180x700.png" width="1180" alt="UTXO指纹结构" />

这是一个与上面区块存储保持同样结构的四层分级（`2+20+2`）树容器，末端的交易数据文件中存储着交易的输出指引（未花费标记）。

末端目录内所有的输出指引合并计算哈希值，上级汇总计算当前目录内子哈希的哈希值，逐级向上得到根哈希，即为UTXO指纹。

这其实是一个宽成员的哈希校验树，总共四层的分级可减少每次输出指引改变带来的重新计算的数据量。顶层为年度，虽然是一个无限增长的序列，但粒度足够大，可接受。


### 意义

UTXO指纹会对区块链末端产生合法性约束，实际上，它有些像全链交易历史的当前总结。正因如此，一个刚刚上线的节点可以请求并不太多的数据量（区块头链、末端11个区块、以及当前UTXO集合），就可以大致确定目标主链是否合法。然后再同步其它区块进行完整校验（如果必要的话）。

这可以极大地降低新节点上线入网的门槛，提升区块链系统的整体效率。


### 附：UTXO指纹的循环递进约束

> #### 当前区块与当前UTXO集合
> 当前区块是指当前正在验证交易数据，即将创建的区块。当前UTXO集合是当前区块所依据的UTXO集合，它尚未减去当前区块所收录交易的花费。
> 当前UTXO集去除掉当前区块收录交易的花费，加上新的输出和当前区块的Coinbase铸币，就成为当前区块成块之后的UTXO结果集。

当前区块的UTXO指纹从当前UTXO集合计算而来，该集合是上一区块完成之后的UTXO结果集（而不是本区块完成之后的结果集）。

这样的设计可以为UTXO指纹计算留出足够的时间，而同时也获得了一种循环递进的约束：攻击者无法在剔除或加入一笔交易的同时，维持影响仅限于本区块（假如哈希碰撞被悄悄实现的话）。这相当于区块链的哈希链式锁定不止限于区块ID本身，也包含UTXO指纹的链式约束。这算是一种双锁定吧。


#### 推导流程示例

- 假设当前区块为101号，当前UTXO集即为第100号区块的UTXO结果集。
- 当前UTXO集减去第100号区块的新输出和Coinbase铸币、加上第100号区块的输入花费，可得到第100号区块的当前UTXO集。
- 计算这个集合的指纹，它应当与第100号区块记录的UTXO指纹相同。这样就验证了（101号区块的）当前UTXO集合的合法性。
- 如果再用第100号区块的当前UTXO集减去第99号区块的新输出和Coinbase铸币，以及同样的输入处理，就可以验证第100号区块的当前UTXO集。
- 如此循环递进，我们就可以从一个最新的UTXO集合逆向验证区块链至任意历史位置。

另外，UTXO指纹表达的是上一区块的UTXO结果集，这使得指纹的约束是链式的，攻击者无法通过单个区块的交易ID塑造来匹配UTXO指纹。
