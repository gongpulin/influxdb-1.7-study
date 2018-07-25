# System Monitoring
_This functionality should be considered experimental and is subject to change._
此功能应视为实验性的，可能会有所变化

_System Monitoring_ means all statistical and diagnostic information made availabe to the user of InfluxDB system, about the system itself. Its purpose is to assist with troubleshooting and performance analysis of the database itself.
系统监控_表示所有统计和诊断信息都可供InfluxDB系统用户使用，关于系统本身。 其目的是协助对数据库本身进行故障排除和性能分析

## Statistics vs. Diagnostics
A distinction is made between _statistics_ and _diagnostics_ for the purposes of monitoring. Generally a statistical quality is something that is being counted, and for which it makes sense to store persistently for historical analysis. Diagnostic information is not necessarily numerical, and may not make sense to store.
出于监控的目的，_statistics_和_diagnostics_之间存在区别。 通常，统计质量是被计算的东西，并且为了历史分析而持久存储是有意义的。 诊断信息不一定是数字，并且可能没有意义存储。
An example of statistical information would be the number of points received over UDP, or the number of queries executed. Examples of diagnostic information would be a list of current Graphite TCP connections, the version of InfluxDB, or the uptime of the process.
统计信息的一个示例是通过UDP接收的点数或执行的查询数。 诊断信息的示例包括当前Graphite TCP连接的列表，InfluxDB的版本或进程的正常运行时间。
## System Statistics
`SHOW STATS [FOR <module>]` displays statisics about subsystems within the running `influxd` process. Statistics include points received, points indexed, bytes written to disk, TCP connections handled etc. These statistics are all zero when the InfluxDB process starts. If _module_ is specified, it must be single-quoted. For example `SHOW STATS FOR 'httpd'`.
SHOW STATS [FOR <module>]`显示正在运行的`Influxd`进程中的子系统的统计信息。 统计信息包括接收的点，索引的点，写入磁盘的字节，处理的TCP连接等。当InfluxDB进程启动时，这些统计信息都为零。 如果指定了_module_，则必须单引号。 例如`SHOW STATS FOR'htt“。

All statistics are written, by default, by each node to a "monitor" database within the InfluxDB system, allowing analysis of aggregated statistical data using the standard InfluxQL language. This allows users to track the performance of their system. Importantly, this allows cluster-level statistics to be viewed, since by querying the monitor database, statistics from all nodes may be queried. This can be a very powerful approach for troubleshooting your InfluxDB system and understanding its behaviour.
默认情况下，所有统计信息都由每个节点写入InfluxDB系统中的“监控”数据库，允许使用标准的InfluxQL语言分析汇总的统计数据。 这允许用户跟踪他们的系统的性能。 重要的是，这允许查看群集级统计信息，因为通过查询监视器数据库，可以查询来自所有节点的统计信息。 这可以是一个非常强大的方法，用于对InfluxDB系统进行故障排除并了解其行为。

## System Diagnostics
`SHOW DIAGNOSTICS [FOR <module>]` displays various diagnostic information about the `influxd` process. This information is not stored persistently within the InfluxDB system. If _module_ is specified, it must be single-quoted. For example `SHOW STATS FOR 'build'`.
`SHOW DIAGNOSTICS [FOR <module>]`显示有关`Influxd`过程的各种诊断信息。 此信息不会持久存储在InfluxDB系统中。 如果指定了_module_，则必须单引号。 例如`SHOW STATS FOR'build'`。

## Standard expvar support
All statistical information is available at HTTP API endpoint `/debug/vars`, in [expvar](https://golang.org/pkg/expvar/) format, allowing external systems to monitor an InfluxDB node. By default, the full path to this endpoint is `http://localhost:8086/debug/vars`.
所有统计信息都以HTTP API端点`/ debug / vars`提供，采用[expvar]（https://golang.org/pkg/expvar/）格式，允许外部系统监控InfluxDB节点。 默认情况下，此端点的完整路径为“http：// localhost：8086 / debug / vars”。

## Configuration
The `monitor` module allows the following configuration:

 * Whether to write statistical and diagnostic information to an InfluxDB system. This is enabled by default.
 * The name of the database to where this information should be written. Defaults to `_internal`. The information is written to the default retention policy for the given database.
 * The name of the retention policy, along with full configuration control of the retention policy, if the default retention policy is not suitable.
 * The rate at which this information should be written. The default rate is once every 10 seconds.
`monitor`模块允许以下配置：

  *是否将统计和诊断信息写入InfluxDB系统。 默认情况下启用此功能。
  *应写入此信息的数据库的名称。 默认为`_internal`。 该信息将写入给定数据库的默认保留策略。
  *如果默认保留策略不合适，则保留策略的名称以及保留策略的完全配置控制。
  *应写入此信息的速率。 默认速率是每10秒一次。

# Design and Implementation

A new module named `monitor` supports all basic statistics and diagnostic functionality. This includes:

 * Allowing other modules to register statistics and diagnostics information, allowing it to be accessed on demand by the `monitor` module.
 * Serving the statistics and diagnostic information to the user, in response to commands such as `SHOW DIAGNOSTICS`.
 * Expose standard Go runtime information such as garbage collection statistics.
 * Make all collected expvar data via HTTP, for collection by 3rd-party tools.
 * Writing the statistical information to the "monitor" database, for query purposes.
名为`monitor`的新模块支持所有基本统计和诊断功能。 这包括：

  *允许其他模块注册统计和诊断信息，允许`monitor`模块按需访问。
  *响应“SHOW DIAGNOSTICS”等命令，向用户提供统计数据和诊断信息。
  *公开标准Go运行时信息，例如垃圾收集统计信息。
  *通过HTTP生成所有收集的expvar数据，以便通过第三方工具进行收集。
  *将统计信息写入“监视器”数据库，以供查询。

## Registering statistics and diagnostics注册统计和诊断

To export statistical information with the `monitor` system, a service should implement the `monitor.Reporter` interface. Services added to the Server will be automatically added to the list of statistics returned. Any service that is not added to the `Services` slice will need to modify the `Server`'s `Statistics(map[string]string)` method to aggregate the call to the service's `Statistics(map[string]string)` method so they are combined into a single response. The `Statistics(map[string]string)` method should return a statistics slice with the passed in tags included. The statistics should be kept inside of an internal structure and should be accessed in a thread-safe way. It is common to create a struct for holding the statistics and using `sync/atomic` instead of locking. If using `sync/atomic`, be sure to align the values in the struct so it works properly on `i386`.

To register diagnostic information, `monitor.RegisterDiagnosticsClient` is called, passing a `influxdb.monitor.DiagsClient` object to `monitor`. Implementing the `influxdb.monitor.DiagsClient` interface requires that your component have function returning diagnostic information in specific form, so that it can be displayed by the `monitor` system.

Statistical information is reset to its initial state when a server is restarted.
要使用`monitor`系统导出统计信息，服务应该实现`monitor.Reporter`接口。添加到服务器的服务将自动添加到返回的统计信息列表中。任何未添加到`Services`切片的服务都需要修改`Server`的`Statistics（map [string] string）`方法来聚合对服务的`Statistics（map [string] string）的调用。方法使它们组合成一个单一的响应。 `Statistics（map [string] string）`方法应该返回一个包含传入标签的统计片。统计信息应保存在内部结构中，并应以线程安全的方式访问。通常创建一个结构来保存统计信息并使用`sync / atomic`而不是lock。如果使用`sync / atomic`，请确保对齐结构中的值，以便它在`i386`上正常工作。

要注册诊断信息，调用`monitor.RegisterDiagnosticsClient`，将`influxdb.monitor.DiagsClient`对象传递给`monitor`。实现`Influxdb.monitor.DiagsClient`接口要求组件具有以特定形式返回诊断信息的功能，以便可以通过`monitor`系统显示它。

重新启动服务器时，统计信息将重置为其初始状态。