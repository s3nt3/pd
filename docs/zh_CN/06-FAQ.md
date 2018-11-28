常见问题
=======

Lightning 对 TiDB/TiKV/PD 的最低版本要求是多少？
--------------------------------------------

最低版本要求是 2.0.4。

Lightning 支持导入多个库吗？
-------------------------

支持。

Lightning 对下游的数据库账号权限要求是？
-----------------------------------

Lightning 需要以下权限：

* SELECT
* UPDATE
* ALTER
* CREATE
* DROP

另外，存储断点的数据库额外需要以下权限：

* INSERT
* DELETE

Lightning 在导数据过程中某个表报错了，会影响其他表吗？进程会马上退出吗?
------------------------------------------------------------

如果只是个别表报错，不会影响整体，报错的那个表会停止处理，继续处理其他的表。

如何校验导入的数据的正确性？
-----------------------

Lightning 默认会对导入数据计算校验和 (checksum)，如果校验和不一致就会停止导入该表。可以在日志看到相关的信息。

TiDB 亦支持从 mysql 命令行运行 `ADMIN CHECKSUM TABLE` 指令计算校验和。

```text
mysql> ADMIN CHECKSUM TABLE `schema`.`table`;
+---------+------------+---------------------+-----------+-------------+
| Db_name | Table_name | Checksum_crc64_xor  | Total_kvs | Total_bytes |
+---------+------------+---------------------+-----------+-------------+
| schema  | table      | 5505282386844578743 |         3 |          96 |
+---------+------------+---------------------+-----------+-------------+
1 row in set (0.01 sec)
```

Lightning 支持哪些格式的数据源？
----------------------------

到 v2.1.0 版本为止，只支持本地文档形式的数据源，支持 [mydumper](https://github.com/pingcap/mydumper) 格式。

我已经在下游创建好库和表了，Lightning 可以忽略建库建表操作吗？
----------------------------------------------------

可以。在配置文档中的 `[data-source]` 将 `no-schema` 设置为 `true` 即可。
`no-schema=true` 会默认下游已经创建好所需的数据库和表，如果没有创建，会报错。

有些不合法的数据，能否通过关掉严格 SQL 模式 (Strict SQL MOde) 来导入？
-------------------------------------------------------------

可以。Lightning 默认的 [`sql_mode`] 为 `"STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION"`。
这个设置不允许一些非法的数值，例如 `1970-00-00` 这样的日期。可以修改配置文件 `[tidb]` 下的 `sql-mode` 值。

```toml
...
[tidb]
sql-mode = ""
...
```

[`sql_mode`]: https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html

可以起一个 `tikv-importer`，同时有多个 `tidb-lightning` 进程导入数据吗？
-----------------------------------------------------------------

只要每个 Lightning 操作的表互不相同就可以。

如何正确关闭 `tikv-importer` 进程？
--------------------------------

如使用 TiDB Ansible 部署，在 Importer 的服务器上运行 `scripts/stop_importer.sh` 即可。

否则，可通过 `ps aux | grep tikv-importer` 获取进程ID，然后 `kill «pid»`。

如何正确关闭 `tidb-lightning` 进程？
---------------------------------

如使用 TiDB Ansible 部署，在 Lightning 的服务器上运行 `scripts/stop_lightning.sh` 即可。

如果 `tidb-lightning` 正在前台运行，可直接按 <kbd>Ctrl</kbd>+<kbd>C</kbd> 退出。

否则，可通过 `ps aux | grep tidb-lightning` 获取进程ID，然后 `kill -2 «pid»`。

进程在服务器上运行，进程莫名其妙地就退出了，是怎么回事呢？
-----------------------------------------------

这种情况可能是启动方式不正确，导致因为收到 SIGHUP 信号而退出，此时 `tidb-lightning.log` 通常有这幺一行日志：

```
2018/08/10 07:29:08.310 main.go:47: [info] Got signal hangup to exit.
```

不推荐直接在命令行中使用 `nohup` 启动进程，而应该把 `nohup` 这行命令放到一个脚本中运行。

为什么用过 Lightning 之后，TiDB 集群变得又慢又耗 CPU？
------------------------------------------------

如果 `tidb-lightning` 曾经异常退出，集群可能仍留在“导入模式” (import mode)，不适合在生产环境工作。
此时需要强制切换回“普通模式” (normal mode)：

```sh
tidb-lightning-ctl --switch-mode=normal
```
