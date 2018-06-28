.. _Troubleshooting OSDs:

==============
 OSD 故障排除
==============

对 OSD 排障前，先检查一下监视器集群和网络。如果 ``ceph health``
或 ``ceph -s`` 返回的是健康状态，这意味着监视器达到了足够数量。
如果监视器数量不足、或者监视器状态错误，要先 `解决监视器问题`_ 。
检查您的网络，确保它在正常运行，因为网络对 OSD 的运行和性能有
显著影响。

.. _解决监视器问题: ../troubleshooting-mon


.. Obtaining Data About OSDs

收集 OSD 数据
=============

除了 `监控 OSD`_ 时（如 ``ceph osd tree`` ）收集到的数据外，
开始 OSD 排障的第一步最好先收集额外信息。


.. Ceph Logs

Ceph 日志
---------

如果没有更改默认路径，您可以在 ``/var/log/ceph`` 下找到日志::

	ls /var/log/ceph

如果看到的日志还不够详细，可以提高日志级别。请参考
`日志记录和调试`_ ，以确保在大量记录日志时不影响集群运行。


.. Admin Socket

管理套接字
----------

用管理套接字检索运行时信息。所有 Ceph 套接字列表::

	ls /var/run/ceph

然后，执行下例命令显示可用选项，把 ``{daemon-name}`` 换成实际的
守护进程（如 ``osd.0`` ）::

	ceph daemon osd.0 help

另外，您也可以指定一个 ``{socket-file}`` （如 ``/var/run/ceph`` 
下的文件）::

	ceph daemon {socket-file} help

除去其他一些功能，您可以用管理接口：

- 在运行时列出配置
- 列出历史操作
- 列出操作的优先队列状态
- 列出在进行的操作
- 列出性能计数器


.. Display Freespace

显示剩余空间
------------

可能是文件系统问题，用 ``df`` 显示文件系统剩余空间。::

	df -h

其它用法见 ``df --help`` 。


.. I/O Statistics

I/O 统计信息
------------

用 `iostat`_ 定位 I/O 相关问题。::

	iostat -x


.. Diagnostic Messages

诊断消息
--------

要查看诊断信息，配合 ``less`` 、 ``more`` 、 ``grep`` 或
``tail`` 使用 ``dmesg`` ，例如::

	dmesg | grep scsi


.. Stopping w/out Rebalancing

停止自动重均衡
==============

您可能需要周期性地维护集群的部分系统、或解决某个故障区域的问题（如一机
架）。如果您不想在停机维护 OSD 时让 CRUSH 自动重均衡，可以预先设置
``noout`` ::

	ceph osd set noout

在集群上设置 ``noout`` 后，您就可以停机维护故障区域内的 OSD 了。::

	stop ceph-osd id={num}

.. note:: 在解决故障区域域内的问题时，停机的 OSD 内归置组状态
   会变为 ``degraded`` 。

维护结束后，重启OSD。::

	start ceph-osd id={num}

最后，您需要解除 ``noout`` 标志。::

	ceph osd unset noout


.. _osd-not-running:

OSD 未运行
==========

通常情况下，简单地重启 ``ceph-osd`` 进程就可以重回集群并恢复。


.. An OSD Won't Start

OSD 无法启动
------------

如果您重启了集群，但其中一个 OSD 无法启动，依次检查：

- **配置文件：** 如果您新装的 OSD 不能启动，检查下配置文件，确
  保它合爻性（比如 ``host`` 而非 ``hostname`` 等）；

- **检查路径：** 检查您配置的路径，以及它们自己的数据和日志路
  径。如果您分离了 OSD 数据和日志、而配置文件和实际挂载点存在
  出入，启动 OSD 时就可能失败。如果您想把日志存储于一个块设备，
  应该为日志硬盘分区并为各 OSD 分别分配一分区。

- **检查最大线程数：** 如果您的节点有很多 OSD ，也许就会触碰到
  默认的最大线程数限制（如通常是 32k 个），尤其是在恢复期间。
  您可以用 ``sysctl`` 增大线程数，把最大线程数更改为支持的最大
  值（即 4194303），看看是否有用。例如::

	sysctl -w kernel.pid_max=4194303

  如果增大最大线程数解决了这个问题，您可以把此配置
  ``kernel.pid_max`` 写入配置文件 ``/etc/sysctl.conf`` ，使之
  永久生效，例如::

	kernel.pid_max = 4194303

- **内核版本：** 确认您在用的内核版本以及所用的发行版。Ceph 
  默认依赖一些第三方工具，这些工具可能有缺陷或者与特定发布版和
  /或内核版本冲突（如 Google perftool）。检查下
  `操作系统推荐`_ 确保您已经解决了内核相关的问题。

- **段错误：** 如果有了段错误，提高日志级别（如果还没提高），
  再试试。如果重现了，联系 ceph-devel 并提供您的配置文件、显示
  器输出和日志文件内容。


.. An OSD Failed

OSD 故障
--------

``ceph-osd`` 挂掉时，监视器可通过活着的 ``ceph-osd`` 了解到此
情况，并通过 ``ceph health`` 命令报告::

	ceph health
	HEALTH_WARN 1/3 in osds are down

而且，有 ``ceph-osd`` 进程标记为 ``in`` 与 ``down`` 的时候，您
会得到警告，您可以用下面的命令得知哪个 ``ceph-osd`` 进程挂了::

	ceph health detail
	HEALTH_WARN 1/3 in osds are down
	osd.0 is down since epoch 23, last address 192.168.106.220:6800/11080

如果有个硬盘出错或其它错误使 ``ceph-osd`` 不能正常运行或重启，
一条错误信息将会出现在日志文件 ``/var/log/ceph/`` 里。

如果守护进程因心跳失败、或者底层文件系统无响应而停止，查看
``dmesg`` 获取硬盘或者内核错误。

如果是软件错误（失败的检验或其它意外错误），应该报告到
`ceph-devel`_ 邮件列表。


.. No Free Drive Space

无剩余硬盘空间
--------------

以免丢失数据, Ceph 不允许您向满的 OSD 写入数据。在运营着的集群
中，您应该能收到集群空间将满的警告。 ``mon osd full ratio`` 默
认为 ``0.95`` 、即达到 95% 时它将阻止客户端写入数据；
``mon osd backfillfull ratio`` 默认为 ``0.90`` 、即达到容量的
90% 时它会阻塞，防止回填启动； ``mon osd nearfull ratio`` 默认
为 ``0.85`` 、即达到容量的 85% 时它会产生健康警告。

集群用满的问题一般出现在测试过程中，为了检验 Ceph 在小型集群上
如何处理 OSD 故障。当某一节点存储的集群数据比例较高时，集群能
够很快掩盖将满和占满率。如果您在小型集群上测试 Ceph 如何应对
OSD 故障，应该保留足够的空闲空间，然后临时降低
``mon osd full ratio`` 、 ``mon osd backfillfull ratio`` 和 
``mon osd nearfull ratio`` 的值。

``ceph health`` 会显示将满的 ``ceph-osds`` ::

	ceph health
	HEALTH_WARN 1 nearfull osd(s)

或者::

	ceph health detail
	HEALTH_ERR 1 full osd(s); 1 backfillfull osd(s); 1 nearfull osd(s)
	osd.3 is full at 97%
	osd.4 is backfill full at 91%
	osd.2 is near full at 87%

处理满集群最好的方法就是增加新的 ``ceph-osd`` ，这允许集群把
数据重分布到新 OSD 里。

如果因满载而导致 OSD 不能启动，您可以试着删除那个 OSD 上的一些
归置组数据目录。

.. important:: 如果您准备从填满的 OSD 中删除某个归置组，注意
   **不要** 删除另一个 OSD 上的同归置组，否则
   **您会丢数据** 。 **必须** 在至少一个 OSD 上保留至少一份数据
   副本。

详情见 `监视器配置参考`_ 。


.. OSDs are Slow/Unresponsive

OSD 慢或无响应
==============

一个反复出现的问题是系统或无响应。在深入性能问题前，您应该先确
保不是其他故障。例如，确保您的网络运行正常、且 OSD 在运行，还
要检查 OSD 是否被恢复流量拖住了。

.. tip:: 较新版本的 Ceph 能更好地处理恢复，可防止恢复进程耗尽
   系统资源而导致 ``up`` 且 ``in`` 的 OSD 不可用或响应慢。


.. Networking Issues

网络问题
--------

Ceph 是一个分布式存储系统，所以它依赖于网络来互联 OSD 们、复制对
象、恢复错误、和检查心跳。网络问题会导致 OSD 延时和状态抖动，详情
参见 `状态抖动的 OSD`_ 。

确保 Ceph 进程和 Ceph 依赖的进程连接了、和/或在监听。::

	netstat -a | grep ceph
	netstat -l | grep ceph
	sudo netstat -p | grep ceph

检查网络统计信息。::

	netstat -s


.. Drive Configuration:

硬盘配置
----------

一个存储硬盘应该只用于一个 OSD 。如果有其它进程共享硬盘，
顺序读和顺序写吞吐量会成为瓶颈，这包括日志记录、操作系统、监视
器、其它 OSD 和非 Ceph 进程。

Ceph 在日志记录 *完成之后* 才会确认写操作，所以使用 ``XFS``
或 ``ext4`` 文件系统时高速的 SSD 对降低响应延时很有帮助。相
反， ``btrfs`` 文件系统可以同时读写（注意，尽管如此，我们推荐
生产部署不要使用 ``btrfs`` ）。

.. note:: 给硬盘分区并不能改变总吞吐量或顺序读写限制。把日
   志分离到单独的分区可能有帮助，但最好使用另外一块硬盘。


.. _Bad Sectors / Fragmented Disk:

坏扇区和硬盘碎片
------------------

检修下硬盘是否有坏扇区和碎片。这会导致总吞吐量急剧下降。


.. Co-resident Monitors/OSDs

监视器和 OSD 同驻
-----------------

监视器通常是轻量级进程，但它们会频繁调用 ``fsync()`` ，这会妨碍其它工作量，特别是
监视器和 OSD 共享硬盘时。另外，如果您在 OSD 主机上同时运行监视器，您可能会遇到与
这些相关的性能问题：

- 运行较老的内核（低于3.0）
- v0.48 版运行在老的 ``glibc`` 之上
- 运行的内核不支持 ``syncfs(2)`` 系统调用

在这些情况下，同一主机上的多个 OSD 会相互拖慢对方。它们经常导致爆发式写入。


.. Co-resident Processes

进程同驻
--------

共存于同一套硬件、并向 Ceph 写入数据的进程（像基于云的解决方案、虚拟机和其他应用程
序）会导致 OSD 延时大增。一般来说，我们建议用一个主机运行 Ceph 、其它主机运行其它进程，实
践证明把 Ceph 和其他应用程序分开可提高性能、并简化故障排除和维护。


.. Logging Levels

日志记录级别
------------

如果您为追踪某问题提高过日志级别、但结束后忘了调回去，这个 OSD 将向硬盘写入大量日
志。如果您想始终保持高日志级别，可以考虑在默认日志路径挂载硬盘，即 
``/var/log/ceph/$cluster-$name.log`` 。


.. Recovery Throttling

恢复节流
--------

根据您的配置，Ceph 可以降低恢复速度来维持性能，否则它会不顾 OSD 性能而加快恢复速
度。检查下 OSD 是否正在恢复。


.. Kernel Version

内核版本
--------

检查下您在用的内核版本。较老的内核也许没有移植能提高 Ceph 性能的功能。


.. Kernel Issues with SyncFS

内核与 SyncFS 问题
------------------

试试在一主机上只运行一个 OSD ，看看能否提升性能。老内核未必支持有 ``syncfs(2)`` 系
统调用的 ``glibc`` 。


.. Filesystem Issues:

文件系统问题
------------

当前，我们推荐基于 xfs 部署集群。

我们推荐不使用 btrfs 或 ext4。btrfs 有很多诱人的功能，
但文件系统内的缺陷可能导致性能问题。我们不建议使用 ext4 ，
因为其 xattr 尺寸限制会破坏我们对长对象名的支持（RGW 所需）。

详情见 `文件系统推荐`_ 。

.. _文件系统推荐: ../configuration/filesystem-recommendations


.. Insufficient RAM

内存不足
--------

我们建议为每 OSD 进程规划 1GB 内存。您也许注意到了，通常情况下 OSD 仅会用一小部分
（如 100-200MB）。您也许想用这些空闲内存运行一些其他应用，如虚拟机等等，然而当 OSD 
进入恢复状态时，其内存利用率激增，如果没有可用内存，此 OSD 的性能将差的多。


.. Old Requests or Slow Requests

Old Requests 或 Slow Requests
------------------------------

如果某 ``ceph-osd`` 守护进程对一请求响应很慢，它会生成日志消息来报告请求耗费的时间
过长。默认警告阀值是 30 秒，用 ``osd op complaint time`` 选项来配置。这种情况发生
时，集群日志系统会收到这些消息。

很老的版本报告 "old requests" ::

	osd.0 192.168.106.220:6800/18813 312 : [WRN] old request osd_op(client.5099.0:790 fatty_26485_object789 [write 0~4096] 2.5e54f643) v4 received at 2012-03-06 15:42:56.054801 currently waiting for sub ops

较新版本的 Ceph 报告 "slow requests" ::

	{date} {osd.num} [WRN] 1 slow requests, 1 included below; oldest blocked for > 30.005692 secs
	{date} {osd.num}  [WRN] slow request 30.005692 seconds old, received at {date-time}: osd_op(client.4240.0:8 benchmark_data_ceph-1_39426_object7 [write 0~4194304] 0.69848840) v4 currently waiting for subops from [610]


可能原因有：

- 坏硬盘（查看 ``dmesg`` 输出）；
- 内核文件系统缺陷（查看 ``dmesg`` 输出）；
- 集群过载（检查系统负载、 iostat 等等）；
- ``ceph-osd`` 守护进程缺陷。

可能的解决方法：

- 从 Ceph 主机去除 VM 云解决方案；
- 升级内核；
- 升级 Ceph；
- 重启 OSD。

.. Debugging Slow Requests

调试 Slow Requests
------------------

如果您执行 ``ceph daemon osd.<id> dump_historic_ops`` 或 ``dump_ops_in_flight``
， 您将看到一系列操作和每个操作经过的事件列表。以下是简要的介绍。

来自信使层的事件:

- header_read: 当信使开始读取消息
- throttled: 当信使试图请求内存节流空间来把消息存入内存
- all_read: 当信使结束读取消息
- dispatched: 当信使把消息发送给 OSD
- Initiated: 与 header_read 相同，因为历史原因

来自准备操作中的 OSD：

- queued_for_pg: 操作已经被放入其 PG 的队列
- reached_pg: PG 开始处理操作
- waiting for \*: 操作需要等待其他的工作先完成（新的 OSDMap、等待其目标对象被
  擦洗、等待 PG 互联，全部由消息而定）
- started: 操作被接受为 OSD 应实际执行的操作（拒绝执行的理由：安全/权限检查失败、
  过期的本地状态等），并正在被执行
- waiting for subops from: 操作被发到副本 OSD。

来自 FileStore 的事件：

- commit_queued_for_journal_write: 操作已经交给 FileStore
- write_thread_in_journal_buffer: 操作已经在日志的缓冲中并等待被写入（下一次硬盘写入）
- journaled_completion_queued: 操作已经被记录在硬盘上并且其回调在调用队列中

来自已经将数据交给本地硬盘的 OSD：

- op_commit: 操作已经被主 OSD 提交（即，写入日志）
- op_applied: 操作已经被主机 write() 到底层文件系统（即，应用到内存但还未写入硬盘）
- sub_op_applied: 与 op_applied 相同，但关于副本的子操作
- sub_op_committed: 与 op_commited 相同，但关于副本的子操作（仅为 EC 池）
- sub_op_commit_rec/sub_op_apply_rec from <X>: 当主机收到来自副本 <X> 的以上两种事件
- commit_sent: 已经向客户端发送回复 (或对于子操作，主 OSD)

这些事件很多看起来有重复，但是越过了内部代码的重要边界（如跨过锁向新进程传递信息）。

.. Flapping OSDs

状态抖动的 OSD
==============

我们建议同时部署公网（前端）和集群网（后端），这样能更好地满足对象复制的容量需求。另
一个优点是您可以运营一个不连接互联网的集群，以此避免拒绝服务攻击。OSD 们互联和检查心跳
时会优选集群网（后端），详情见 `监视器与 OSD 的交互`_ 。

然而，如果集群网（后端）故障、或出现了明显的延时，同时公网（前端）
却运行良好，OSD 现在不能很好地处理这种情况。这时 OSD 们会向监视器
报告邻居 ``down`` 了、同时报告自己是 ``up`` 的，我们把这种情形称为
状态抖动（flapping）。

.. note:: 译者：社区同仁讨论认为，这是随时间延续，不断地在 ``up``、
	``down`` 状态之间反复的情形，状态变动的时间间隔有规律或无
	规律，运动方向为“上下”，非“左右”、亦非“前后”，也可理解为打
	摆子、状态翻转

如果有东西导致 OSD 状态抖动（反复地被标记为 ``down`` ，然后又 
``up`` ），您可以强制监视器停止::

	ceph osd set noup      # prevent OSDs from getting marked up
	ceph osd set nodown    # prevent OSDs from getting marked down

这些标记记录在 osdmap 数据结构里::

	ceph osd dump | grep flags
	flags no-up,no-down

下列命令可清除标记::

	ceph osd unset noup
	ceph osd unset nodown

``mon osd down out interval`` is).
还支持其它两个标记 ``noin`` 和 ``noout`` ，它们分别可防止正在启动的 OSD 被标记为 
``in`` 、或被误标记为 ``out`` （不管 ``mon osd down out interval`` 的值是什么）。

.. note:: ``noup`` 、 ``noout`` 和 ``nodown`` 从某种意义上说是临时的，一旦标记清
   除了，它们被阻塞的动作短时间内就会发生；相反， ``noin`` 标记阻止 OSD 启动后进入
   集群，但其它守护进程都维持原样。


.. _iostat: http://en.wikipedia.org/wiki/Iostat
.. _Ceph 日志记录和调试: ../../configuration/ceph-conf#ceph-logging-and-debugging
.. _日志记录和调试: ../log-and-debug
.. _调试和日志记录: ../debug
.. _监视器与 OSD 的交互: ../../configuration/mon-osd-interaction
.. _监视器配置参考: ../../configuration/mon-config-ref
.. _监控 OSD: ../../operations/monitoring-osd-pg
.. _订阅 ceph-devel 邮件列表: mailto:majordomo@vger.kernel.org?body=subscribe+ceph-devel
.. _退订 ceph-devel 邮件列表: mailto:majordomo@vger.kernel.org?body=unsubscribe+ceph-devel
.. _订阅 ceph-users 邮件列表: mailto:ceph-users-join@lists.ceph.com
.. _退订 ceph-users 邮件列表: mailto:ceph-users-leave@lists.ceph.com
.. _操作系统推荐: ../../../start/os-recommendations
.. _ceph-devel: ceph-devel@vger.kernel.org
