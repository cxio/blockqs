# 区块查询服务

详细设计请参考 [design](conception/design.md)。


## 安装

这是一个服务器程序，下载发行版，命令行执行即可。


### 命令行

-h, --help      显示帮助信息
-v, --version   显示版本信息
--config 文件   指定配置文件路径（默认：`./config.jsonc`）


### 配置

用户可以通过配置文件（`config.jsonc`）指定自己的服务端口和数据、日志的存储位置等。

当前仅支持单条区块链服务，因此收益地址在配置文件中指定。如果一台主机需要同时支持多条区块链，只能启动多个实例（注意配置修改）。



## 实时服务

类似于常见的Web服务器，通过http协议提供数据检索，实时高效。


### API接口

主要用于实时查询，小于*10MB*的附件通常也可以直接获取（可选服务）。

- `GET /transaction/{year}/{id}`
  检索交易数据，应先确知交易的时间戳。

- `GET /scripts/{year}/{id}/{offset}`
  检索输出脚本。

- `GET /scripts/{year}/{id}`
  检索输出脚本集。

- `GET /txids/{year}/{idpart}`
  获取交易ID组。
  `idpart` 为交易ID的前部片段，相同前段的交易ID即为同组。

- `HEAD /transaction/{year}/{id}`
  检查目标交易是否存在。
  需准确指定交易收录入的区块年度。如果存在，返回状态码200。

- `HEAD /utxo/{year}/{id}/{offset}`
  检查目标输出是否未花费。

- `HEAD /annex/{Ax_id}`
  查询附件是否可用。如果返回200状态码，即可以请求获取。

（待补充……）
