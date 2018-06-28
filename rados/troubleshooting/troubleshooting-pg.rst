============
 归置组排障
============

归置组总不被清理
=================

当您创建的集群并且您的集群一直处于 ``active`` ， ``active+remapped`` 或 
``active+degraded`` 并且永远不会达到 ``active+clean`` 时，您的配置可能有问题。

您可能需要查看 `存储池、归置组和 CRUSH 配置参考`_ 中的设置并进行适当的调整。

一般来讲，您应该使用多个 OSD 运行集群，并且池大小大于1个对象副本。

单节点集群
----------------

Ceph 不再提供在单个节点运营的文档，因为您不会在单个节点上部署专为分布式计算而设计的系统。此外，
在包含 Ceph 守护进程的单个节点上挂载客户端内核模块可能会由于 Linux 内核本身的问题（除非您为客
户端使用虚拟机）导致死锁。尽管有这些的限制，您可以在单节点配置中尝试 Ceph。

如果您尝试在单个节点上创建集群，则在创建监视器与 OSD 之前，必须将 Ceph 配置文件中的
``osd crush chooseleaf type`` 设置的默认值从 ``1`` （表示 ``host`` 或 ``node`` ）更改
为``0``（表示osd ``osd`` ）。这告诉 Ceph OSD 可以与同一主机上的另一个 OSD 进行互联。如果您
尝试设置单节点集群，并且 ``osd crush chooseleaf type`` 大于 ``0`` ，取决于您的设置 Ceph
将尝试将 OSD 的 PG 与在另一个节点、机箱、机柜、机架、甚至另一个数据中心的 OSD 的 PG 互联。

.. tip:: 不要将内核客户端直接挂载在与 Ceph 存储集群相同的节点上，因为可能会产生内核冲突。 但
    是，您可以将内核客户端挂载在单个节点上的虚拟机（VM）中。

如果您使用单个硬盘创建 OSD，则必须首先手动为数据创建目录。 例如::

	ceph-deploy osd create --data {disk} {host}


OSD 比副本少
-------------

如果您已经将两个 OSD 调整到 ``up`` 和 ``in`` 的状态，但仍未看到 ``active + clean`` 的归
置组，可能是 ``osd pool default size`` 的值大于 ``2`` 。

有几种方法可以解决这种情况.如果您希望以两个副本 ``active + degraded`` 的状态运行集群，您可
以将 ``osd pool default min size`` 设为 ``2`` ，这样您就可以在 ``active + degraded`` 
的状态写入对象。您也可以将 ``osd pool default size`` 设置为 ``2`` ，这样您只有两个存储的副
本（原始和一个副本），在这种情况下，集群应达到 ``active + clean`` 状态。

.. note:: 您可以在运行时进行更改。 如果您在 Ceph 配置文件中进行更改，则可能需要重新启动集群。


池大小 = 1
-------------

如果将 ``osd pool default size`` 设置为 ``1`` ，对象将只有一个副本。OSD 依靠其他 OSD 来
获知它应该拥有哪个对象。如果只有一个副本，那么其他 OSD 无法告诉该 OSD 哪些对象它应该拥有。对于
该 OSD 对应的每个归置组（见 ``ceph pg dump`` ），您可以运行以下命令强制该 OSD 注意它需要的
归置组::

    ceph osd force-create-pg <pgid>
   

CRUSH 图错误
------------

另一个导致您的归置组不被清理的可能就是 CRUSH 图中的错误。

卡住的归置组
============

有失败时归置组会进入“degraded”（降级）或“peering”（连接建立中）状态，这事时有发
生，通常这些状态意味着正常的失败恢复正在进行。然而，如果一个归置组长时间处于某个这些
状态就意味着有更大的问题，因此监视器在归置组卡 （stuck） 在非最优状态时会警告。我们
具体检查：

* ``inactive`` （不活跃）——归置组长时间无活跃（即它不能提供读写服务了）；
  
* ``unclean`` （不干净）——归置组长时间不干净（例如它未能从前面的失败完全恢复）；

* ``stale`` （过时）——归置组状态没有被 ``ceph-osd`` 更新，表明存储这个归置组的所
  有节点可能都已下线。

您可以列出卡住的归置组::

	ceph pg dump_stuck stale
	ceph pg dump_stuck inactive
	ceph pg dump_stuck unclean

卡在 ``stale`` 状态的归置组通过修复 ``ceph-osd`` 进程通常可以修复；卡在 
``inactive`` 状态的归置组通常是互联问题（参见 :ref:`failures-osd-peering` ）；卡
在 ``unclean`` 状态的归置组通常是由于某些原因阻止了恢复的完成，例如未找到的对象（参
见 :ref:`failures-osd-unfound` ）。


.. _failures-osd-peering:

归置组下线——互联失败
========================

在某些情况下， ``ceph-osd`` 连接建立进程会遇到问题，使 PG 不能活跃、可用，例如 
``ceph health`` 也许显示::

	ceph health detail
	HEALTH_ERR 7 pgs degraded; 12 pgs down; 12 pgs peering; 1 pgs recovering; 6 pgs stuck unclean; 114/3300 degraded (3.455%); 1/3 in osds are down
	...
	pg 0.5 is down+peering
	pg 1.4 is down+peering
	...
	osd.1 is down since epoch 69, last address 192.168.106.220:6801/8651

可以查询到 PG 为何被标记为 ``down`` ::

	ceph pg 0.5 query

.. code-block:: javascript

 { "state": "down+peering",
   ...
   "recovery_state": [
        { "name": "Started\/Primary\/Peering\/GetInfo",
          "enter_time": "2012-03-06 14:40:16.169679",
          "requested_info_from": []},
        { "name": "Started\/Primary\/Peering",
          "enter_time": "2012-03-06 14:40:16.169659",
          "probing_osds": [
                0,
                1],
          "blocked": "peering is blocked due to down osds",
          "down_osds_we_would_probe": [
                1],
          "peering_blocked_by": [
                { "osd": 1,
                  "current_lost_at": 0,
                  "comment": "starting or marking this osd lost may let us proceed"}]},
        { "name": "Started",
          "enter_time": "2012-03-06 14:40:16.169513"}
    ]
 }

``recovery_state`` 段告诉我们连接建立因 ``ceph-osd`` 进程下线而被阻塞，本例是 
``osd.1`` 下线了，启动这个进程应该就可以恢复。

另外，如果 ``osd.1`` 是灾难性的失败（如硬盘损坏），我们可以告诉集群它丢失（ 
``lost`` ）了，让集群尽力完成副本拷贝。

.. important:: 集群不能保证其它数据副本一致且最新很危险！

让 Ceph 无论如何都继续::

	ceph osd lost 1

恢复将继续。


.. _failures-osd-unfound:

未找到的对象
============

某几种失败相组合可能导致 Ceph 抱怨有找不到（ ``unfound`` ）的对象::

	ceph health detail
	HEALTH_WARN 1 pgs degraded; 78/3778 unfound (2.065%)
	pg 2.4 is active+degraded, 78 unfound

这意味着存储集群知道一些对象（或者存在对象的较新副本）存在，却没有找到它们的副本。下
例展示了这种情况是如何发生的，一个 PG 的数据存储在 ceph-osd 1 和 2 上：

* 1 下线了；
* 2 独自处理一些写动作；
* 1 上线了；
* 1 和 2 重新互联， 1 上面丢失的对象加入队列准备恢复；
* 新对象还未拷贝完， 2 下线了。

这时， 1 知道这些对象存在，但是在线的 ``ceph-osd`` 都没有副本，这种情况下，读写这些
对象的 IO 就会被阻塞，集群只能指望节点早点恢复。这时我们假设用户希望先得到一个 IO 
错误。

首先，您应该确认哪些对象找不到了::

	ceph pg 2.4 list_missing [starting offset, in json]

.. code-block:: javascript

 { "offset": { "oid": "",
      "key": "",
      "snapid": 0,
      "hash": 0,
      "max": 0},
  "num_missing": 0,
  "num_unfound": 0,
  "objects": [
     { "oid": "object 1",
       "key": "",
       "hash": 0,
       "max": 0 },
     ...
  ],
  "more": 0}

如果在一次查询里列出的对象太多， ``more`` 这个字段将为真，并且您可以查询更
多对象。（命令行工具可能将一些信息隐藏，但这里没有）

其次，您可以找出哪些 OSD 上探测到、或可能包含数据::

	ceph pg 2.4 query

.. code-block:: javascript

   "recovery_state": [
        { "name": "Started\/Primary\/Active",
          "enter_time": "2012-03-06 15:15:46.713212",
          "might_have_unfound": [
                { "osd": 1,
                  "status": "osd is down"}]},

本例中，集群知道 ``osd.1`` 可能有数据，但它下线了（ ``down`` ）。所有可能的状态有：

* 已经探测到了
* 在查询
* OSD 下线了
* 尚未查询

有时候集群要花一些时间来查询可能的位置。

还有一种可能性，对象存在于其它位置却未被列出，例如，集群里的一个 ``ceph-osd`` 停止
并被剔出，然后集群完全恢复了；后来的失败、恢复后导致未找到的对象，它也不会觉得早已分离
的 ceph-osd 上仍可能包含这些对象。（这种情况不太可能发生）。

如果所有位置都查询过了仍有对象丢失，那就得放弃丢失的对象了。这种可能是罕见的失败组合
导致的，集群在写入完成前，未能得知写入是否已执行。以下命令把未找到的（ unfound ）对
象标记为丢失（ lost ）。 ::

	ceph pg 2.5 mark_unfound_lost revert|delete

上述最后一个参数告诉集群应如何处理丢失的对象。

delete 选项将完全删除它们。

revert 选项（纠删码存储池不可用）会回滚到前一个版本或者（如果它是新对象的话）删除
它。要慎用，它可能迷惑那些期望对象存在的应用程序。


无根归置组
==========

所有拥有归置组拷贝的 OSD 可能都失败，在这种情况下，那一部分的对象存储不可用，监视器就不
会收到那些归置组的状态更新。为检测这种情况，监视器把任何主 OSD 失败的归置组标记
为 ``stale`` （过时），例如::

	ceph health
	HEALTH_WARN 24 pgs stale; 3/300 in osds are down

您能找出哪些归置组为 ``stale`` 、和存储这些归置组的最后的 OSD ，命令如下::

	ceph health detail
	HEALTH_WARN 24 pgs stale; 3/300 in osds are down
	...
	pg 2.5 is stuck stale+active+remapped, last acting [2,0]
	...
	osd.10 is down since epoch 23, last address 192.168.106.220:6800/11080
	osd.11 is down since epoch 13, last address 192.168.106.220:6803/11539
	osd.12 is down since epoch 24, last address 192.168.106.220:6806/11861

如果想使归置组 2.5 重新上线，上面的输出告诉我们它最后由 ``osd.0`` 和 
``osd.2`` 管理，重启这些 ``ceph-osd`` 将恢复它（可能还有其它的很多 PG ）。


只有几个 OSD 接收数据
=====================

如果您的集群有很多节点，但只有其中几个接收数据， `检查`_ 下存储池里的归置组数量。
因为归置组是映射到多个 OSD 的，少量的归置组将不能分布于整个集群。试着创建个新存
储池，其归置组数量是 OSD 数量的若干倍。详情见 `归置组`_ ，存储池的默认归置组数量
没多大用，您可以参考 `这里`_ 更改它。


不能写入数据
============

如果您的集群已启动，但一些 OSD 没启动，导致不能写入数据，确认下运行的 OSD 数量满足
归置组要求的最低 OSD 数。如果不能满足， Ceph 就不会允许您写入数据，因为 Ceph 不能保
证复制能如愿进行。详情参见 `存储池、归置组和 CRUSH 配置参考`_ 里的 
``osd pool default min size`` 。


.. _PGs Inconsistent:

归置组不一致
============

如果您看到状态变成了 ``active + clean + inconsistent`` ，这可能
是洗刷时遇到了错误。与往常一样，我们可以这样找出不一致的归置组::

    $ ceph health detail
    HEALTH_ERR 1 pgs inconsistent; 2 scrub errors
    pg 0.6 is active+clean+inconsistent, acting [0,1,2]
    2 scrub errors

或者这样，如果您喜欢程序化的输出::

    $ rados list-inconsistent-pg rbd
    ["0.6"]

一致的状态只有一种，然而在最坏的情况下，我们可能会遇到多个对象
产生了各种各样的不一致。假设在 PG ``0.6`` 里的一个名为 ``foo``
的对象被截断了，我们将会看到::

    $ rados list-inconsistent-obj 0.6 --format=json-pretty

.. code-block:: javascript

    {
        "epoch": 14,
        "inconsistents": [
            {
                "object": {
                    "name": "foo",
                    "nspace": "",
                    "locator": "",
                    "snap": "head",
                    "version": 1
                },
                "errors": [
                    "data_digest_mismatch",
                    "size_mismatch"
                ],
                "union_shard_errors": [
                    "data_digest_mismatch_info",
                    "size_mismatch_info"
                ],
                "selected_object_info": "0:602f83fe:::foo:head(16'1 client.4110.0:1 dirty|data_digest|omap_digest s 968 uv 1 dd e978e67f od ffffffff alloc_hint [0 0 0])",
                "shards": [
                    {
                        "osd": 0,
                        "errors": [],
                        "size": 968,
                        "omap_digest": "0xffffffff",
                        "data_digest": "0xe978e67f"
                    },
                    {
                        "osd": 1,
                        "errors": [],
                        "size": 968,
                        "omap_digest": "0xffffffff",
                        "data_digest": "0xe978e67f"
                    },
                    {
                        "osd": 2,
                        "errors": [
                            "data_digest_mismatch_info",
                            "size_mismatch_info"
                        ],
                        "size": 0,
                        "omap_digest": "0xffffffff",
                        "data_digest": "0xffffffff"
                    }
                ]
            }
        ]
    }

此时，我们可以从输出里看到：

* 唯一不一致的对象名为 ``foo`` ，并且它的 head 不一致。
* 不一致分为两类：

  * ``errors``: 这些错误表明不一致性出现在分片之间，但是没说明
    哪个（或哪些）分片有问题。如果 `shards` 阵列中有 ``errors``
    字段，且不为空，它会指出问题所在。

    * ``data_digest_mismatch``: OSD.2 内读取到的副本的数字摘要
      与 OSD.0 和 OSD.1 的不一样。
    * ``size_mismatch``: OSD.2 内读取到的副本的大小是 0 ，而
      OSD.0 和 OSD.1 说是 968 。
  * ``union_shard_errors``: ``shards`` 阵列中、所有与分片相关
    的 ``errors`` 的并集。 ``errors`` 是个拥有该错误的分片的集合，
    如 ``read_error`` 。以 ``oi`` 结尾的 ``errors`` 表明它是与 ``selected_object_info`` 的对照结果。从 ``shards`` 阵列里
    可以查到哪个分片有什么样的错误。

    * ``data_digest_mismatch_info``: 存储在 object-info （对象信
      息）里的数字签名不是 ``0xffffffff`` （这个是根据 OSD.2 
      上的分片计算出来的）。
    * ``size_mismatch_oi``: object-info 内存储的大小与 OSD.2 
      上的对象大小 0 不同。

您可以用下列命令修复不一致的归置组::

	ceph pg repair {placement-group-ID}

此命令会用 `权威的` 副本覆盖 `有问题的` 。根据既定规则，多
数情况下 Ceph 都能从若干副本中选择正确的，但是也会有例外。比
如，存储的数字摘要可能正好丢了，而计算出的数字摘要选择权威副本
时被忽略，总之，用此命令时小心为好。

如果一个分片的 ``errors`` 里出现了 ``read_error`` ，很可能是硬
盘错误引起的不一致，您最好先查验那个 OSD 所用的硬盘。

如果您周期性遇到时钟偏移引起的 ``active + clean + inconsistent``
状态，最好在监视器主机上配置互联的 `NTP`_ 服务。配置细节
可参考 `网络时间协议`_ 和 Ceph `时钟选项`_ 。


.. _Erasure Coded PGs are not active+clean:

纠删编码的归置组不是 active+clean
=================================

CRUSH 找不到足够多的 OSD 映射到某个 PG 时，它会显示为 ``2147483647`` ，意思
是 ITEM_NONE 或 ``no OSD found`` ，例如::

	[2,1,6,0,5,8,2147483647,7,4]

OSD 不够多
----------

如果 Ceph 集群仅有 8 个 OSD ，但是纠删码存储池需要 9 个，就会显示上面的错
误。这时候，您仍然可以另外创建需要较少 OSD 的纠删码存储池::

	ceph osd erasure-code-profile set myprofile k=5 m=3
	ceph osd pool create erasurepool 16 16 erasure myprofile

或者新增一个 OSD ，这个 PG 会自动使用它。

CRUSH 条件不能满足
------------------

即使集群拥有足够多的 OSD ， CRUSH 规则集的强制要求仍有可能无法满足。假如有 
10 个 OSD 分布于两个主机上，且 CRUSH 规则集要求相同归置组不得使用位于同一主
机的两个 OSD ，这样映射就会失败，因为只能找到两个 OSD ，您可以从规则集里查看
必要条件::

    $ ceph osd crush rule ls
    [
        "replicated_rule",
        "erasurepool"]
    $ ceph osd crush rule dump erasurepool
    { "rule_id": 1,
      "rule_name": "erasurepool",
      "ruleset": 1,
      "type": 3,
      "min_size": 3,
      "max_size": 20,
      "steps": [
            { "op": "take",
              "item": -1,
              "item_name": "default"},
            { "op": "chooseleaf_indep",
              "num": 0,
              "type": "host"},
            { "op": "emit"}]}

可以这样解决此问题，创建新存储池，其内的 PG 允许多个 OSD 位于同一主机，命令
如下::

	ceph osd erasure-code-profile set myprofile ruleset-failure-domain=osd
	ceph osd pool create erasurepool 16 16 erasure myprofile

CRUSH 过早中止
--------------

假设集群仅拥有足以映射到 PG 的 OSD （比如有 9 个 OSD 和一个纠删码存储池的集群，
每个 PG 需要 9 个 OSD ）， CRUSH 仍然有可能在找到映射前就中止了。可以这样解决：

* 降低纠删存储池内 PG 的要求，让它使用较少的 OSD （需创建另一个存储池，因为
  纠删码配置不支持动态修改）。

* 向集群添加更多 OSD （无需修改纠删存储池，它会自动回到清洁状态）。

* 通过手工打造的 CRUSH 规则集，让它多试几次以找到合适的映射。把 
  ``set_choose_tries`` 设置得高于默认值即可。

您从集群中提取出 crushmap 之后，应该先用 ``crushtool`` 校验一下是否有问题，
这样您的试验就无需触及 Ceph 集群，只要在一个本地文件上测试即可::

    $ ceph osd crush rule dump erasurepool
    { "rule_name": "erasurepool",
      "ruleset": 1,
      "type": 3,
      "min_size": 3,
      "max_size": 20,
      "steps": [
            { "op": "take",
              "item": -1,
              "item_name": "default"},
            { "op": "chooseleaf_indep",
              "num": 0,
              "type": "host"},
            { "op": "emit"}]}
    $ ceph osd getcrushmap > crush.map
    got crush map from osdmap epoch 13
    $ crushtool -i crush.map --test --show-bad-mappings \
       --rule 1 \
       --num-rep 9 \
       --min-x 1 --max-x $((1024 * 1024))
    bad mapping rule 8 x 43 num_rep 9 result [3,2,7,1,2147483647,8,5,6,0]
    bad mapping rule 8 x 79 num_rep 9 result [6,0,2,1,4,7,2147483647,5,8]
    bad mapping rule 8 x 173 num_rep 9 result [0,4,6,8,2,1,3,7,2147483647]

其中 ``--num-rep`` 是纠删码 crush 规则集所需的 OSD 数量， ``--rule`` 是 
``ceph osd crush rule dump`` 命令结果中 ``ruleset`` 字段的值。此测试会尝试
映射一百万个值（即 ``[--min-x,--max-x]`` 所指定的范围），且必须至少显示一
个坏映射；如果它没有任何输出，说明所有映射都成功了，您可以就此停下：问题的
根源不在这里。

反编译 crush 图后，您可以手动编辑其规则集::

    $ crushtool --decompile crush.map > crush.txt

并把下面这行加进规则集::

    step set_choose_tries 100

然后 ``crush.txt`` 文件内的这部分大致如此::

     rule erasurepool {
             ruleset 1
             type erasure
             min_size 3
             max_size 20
             step set_chooseleaf_tries 5
             step set_choose_tries 100
             step take default
             step chooseleaf indep 0 type host
             step emit
     }

然后编译、并再次测试::

    $ crushtool --compile crush.txt -o better-crush.map

所有映射都成功时，用 ``crushtool`` 的 ``--show-choose-tries`` 选项能看到成
功映射的尝试次数直方图::

    $ crushtool -i better-crush.map --test --show-bad-mappings \
       --show-choose-tries \
       --rule 1 \
       --num-rep 9 \
       --min-x 1 --max-x $((1024 * 1024))
    ...
    11:        42
    12:        44
    13:        54
    14:        45
    15:        35
    16:        34
    17:        30
    18:        25
    19:        19
    20:        22
    21:        20
    22:        17
    23:        13
    24:        16
    25:        13
    26:        11
    27:        11
    28:        13
    29:        11
    30:        10
    31:         6
    32:         5
    33:        10
    34:         3
    35:         7
    36:         5
    37:         2
    38:         5
    39:         5
    40:         2
    41:         5
    42:         4
    43:         1
    44:         2
    45:         2
    46:         3
    47:         1
    48:         0
    ...
    102:         0
    103:         1
    104:         0
    ...

有 42 个归置组需 11 次重试、 44 个归置组需 12 次重试，以此类推。这样，重试
的最高次数就是防止坏映射的最低值，也就是 ``set_choose_tries`` 的取值（即上
面输出中的 103 ，因为任意归置组成功映射的重试次数都没有超过 103 ）。


.. _检查: ../../operations/placement-groups#get-the-number-of-placement-groups
.. _这里: ../../configuration/pool-pg-config-ref
.. _归置组: ../../operations/placement-groups
.. _存储池、归置组和 CRUSH 配置参考: ../../configuration/pool-pg-config-ref
.. _NTP: http://en.wikipedia.org/wiki/Network_Time_Protocol
.. _网络时间协议: http://www.ntp.org/
.. _时钟选项: ../../configuration/mon-config-ref/#clock



