# 区块查询服务

设计细节详见 [design](docs/design.md)。


## 安装

这是一个服务器程序，下载发行版，命令行执行即可。


### 命令行

-h, --help      显示帮助信息
-v, --version   显示版本信息
--config 文件   指定配置文件路径（默认：`./config.hjson`）


### 配置

用户可以通过配置文件（`config.hjson`）指定自己的服务端口和数据、日志的存储位置等。

当前仅支持单条区块链服务，因此收益地址在配置文件中指定。如果一台主机需要同时支持多条区块链，只能启动多个实例（注意配置修改）。



## 实时服务

类似于常见的Web服务器，通过http协议提供数据检索，实时高效。

- **脚本**：`年度 + 交易ID + 输出偏移` 检索。通用的交易输入源检索。
- **交易**：`年度 + 交易ID` 检索。对交易本身的完整检索。
- **UTXO**：`年度 + 交易ID + 输出偏移` 检索。查询目标是否未花费，可能由非校验节点的普通用户使用。

> **注：**
> 对于高实时性要求的校验组，通常有一个自己的UTXO缓存集。


### API接口

仅包含查询，且不支持大数据量的*整体区块*的获取。

小于*10MB*的附件可以直接获取，但整体区块和附件数据的上传由文档分享应用负责。

- `GET /transaction/{year}/{id}`
  检索交易数据。
  应确知交易收录入的区块年度，否则使用下面的查询模式（带 `?year=yyyy`）。

- `GET /transaction/{id}?year=yyyy`
  检索交易数据。
  可指定一个起始搜索年度（`yyyy`），会向后直到当前年度（或**10**年）的范围内查找。
  最终的真实年度会通过响应头中的 `X-Storaged-Year` 字段返回。
  下同。

- `GET /scripts/{year}/{id}/{offset}`
  检索输出脚本。
  逻辑同上，应已确知年度，以及输出脚本的序位。

- `GET /scripts/{id}/{offset}?year=yyyy`
  检索输出脚本。
  准确年度未知，`yyyy`为起始查询年度，范围同上。

- `GET /scripts/{year}/{id}`
  检索输出脚本集。
  应确知交易收录入的区块年度，否则使用下面的查询模式。

- `GET /scripts/{id}?year=yyyy`
  检索输出脚本集。
  准确年度未知，`yyyy`为起始查询年度，范围同上。

- `GET /annex/{id}`
  获取小型附件（<10MB）。
  大尺寸附件从数据网络查询，由文件分享App（`Archives`）传输。

- `GET /txids/{year}/{idpart}`
  获取交易ID组。
  `idpart` 为交易ID的前部片段，相同前段的交易ID即为同组。

- `HEAD /transaction/{year}/{id}`
  检查目标交易是否存在。
  需准确指定交易收录入的区块年度。如果存在，返回状态码200，否则为404错误。

- `HEAD /transaction/{id}?year=yyyy`
  检查目标交易是否存在。
  从指定的年度（`yyyy`）开始检查，会向后直到当前年度（或**100**年）的范围内查找。

- `HEAD /utxo/{year}/{id}/{offset}`
  检查目标输出是否未花费。
  这是一个存在性测试，只提供准确URL下的查询，不支持未知年度检索。

- `HEAD /annex/{id}`
  查询附件是否存在。

> **提示：**
> 如果知道交易的时间戳，从时间戳所在年度开始即可。
> 因为区块不收录未来的交易，所以年度取值在时间戳之前没有意义。**注**：主要针对年末的情况。



## 文档分享服务

通过数据网络的P2P文件分享方式工作，包括数据的获取和供给，主要针对完整区块（较大）和附件数据。

这一操作由单独的P2P文件分享App完成，两者可能共享相同的存储目录。

> **注：**
> 详见 [Archives](https://github.com/cxio/archives) 项目。
