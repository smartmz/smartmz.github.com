---
layout: post
title: "Pacemaker的CIB配置"
date: 2016-02-16 16:42
keywords: tfs,ha
description: Heartbeat v3资源管理层的重点
categories: [HA]
tags: [ha, pacemaker]
group: archive
icon: globe
---

<!-- more -->

　　HA的主要工作就是根据集群当前情况对资源进行分配调度，达到服务不中断的目的。这里涉及到两方面，一是资源本身，二是根据资源和PE的调度指令对资源实现调度。对应到具体的实现上，第一部分内容涉及Pacemaker的调度资源，就是RA，RA重点关注OCF类的；第二部分涉及Pacemaker如何进行资源调度，就是CIB配置。集群DC节点上的PE通过CIB配置决策资源在各个节点上的分配，然后由CRM通过节点上的LRM调用RA完成资源的启停。第一部分参见[《[HA]Pacemaker定制OCF类RA资源》](http://smartmz.github.io/2016/01/21/ha-pacemaker-ocf)，本文重点来说第二部分资源调度如何配置。   

　　在Heartbeat v3中，CRM主要由分离项目Pacemaker承担资源管理工作。（Pacemaker的[官方网站](http://clusterlabs.org/wiki/Main_Page))   
　　目前Pacemaker的版本1.1.14，2016年1月中旬发布，已经支持了CentOS7.包含的组件包括：

* stonithd：心跳系统，决定节点是否加入或脱离集群。【**节点级别**】
* lrmd：LRM，本地资源管理守护进程。它提供了一个通用的接口支持的资源类型。直接调用资源代理（脚本）。
* pengine：PE，策略引擎。根据当前状态和配置集群计算的下一个状态。产生一个过渡图，包含行动和依赖关系的列表。
* CIB：集群信息库。包含所有群集选项，节点，资源，他们彼此之间的关系和现状的定义。**自动同步更新**到所有群集节点。
* CRMD：集群资源管理守护进程。主要是消息代理的PEngine和LRM，还选举一个领导者（DC）统筹活动（包括启动/停止资源）的集群。【**资源级别**】
* OpenAIS：OpenAIS的消息和成员层。
* Heartbeat：心跳消息层，替代OpenAIS，系列文章中都使用Heartbeat.
* CCM：集群成员一致性管理模块。

### 内部架构

　　引用官方的一张图将[《[HA]Linux-HA基本原理》](http://smartmz.github.io/2016/01/15/ha-base-knowlege)中 *图1 HA的集群事务决策模型* 中的Resource Allocated Layer层进行拓展：

<center>![图1 Pacemaker内部原理图](http://ww3.sinaimg.cn/bmiddle/a8484315jw1f1140i776qj20m80gowh5.jpg)</center><br/><center><font size=2>图1 Pacemaker内部原理图</font></center>

　　CIB使用xml配置集群和当前集群中的所有资源，会在节点之间自动同步。PE通过CIB决策集群如何达到最佳状态（选出master）。  
　　master选举逻辑根据PE的决策指令在所有的节点中选择一个节点（CRMd）作为master，如果该节点的CRMd出错，会再选择一个。这个选举逻辑官方叫做 DC(Designated Controller)。具体的过程是：  
　　DC将PE的决策指令按照指定的优先级顺序传递给本节点的LRMd、通过CRMd传递给其他节点的LRMd，同时确定一个希望的（expected）master. 各个节点的LRMd调用TE执行具体的决策指令选出各自认为的master，将结果回传给DC，DC根据之前希望的master和其他节点已经回传的执行结果确定是继续等待优先级高的节点的结果，还是终断本次选举并通知PE再进行一轮选举。   
　　在一些情况下，为了保护共享数据或者完成数据恢复，DC可能决策某节点需要脱离集群，这需要STONITHd协助完成。    
　　在整个过程中重点需要搞清楚的有两点：CIB怎么配置，这是本文的重点；DC过程如何选择master，可以参考[《[HA]Pacemaker的分数机制和资源黏性》](http://smartmz.github.io/2016/02/18/ha-pacemaker-score-and-stickiness)    
### Pacemaker支持的集群模式
　　[《[HA]Linux-HA基本原理》](http://smartmz.github.io/2016/01/15/ha-base-knowlege)中提到的几种集群形式Pacemaker都可以支持，在TFS集群的搭建过程中主要使用主备模式，也是相关系列文章关注的模式。    
　　Pacemaker官方介绍了其他的模式，可以参考[这里](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_types_of_pacemaker_clusters.html)和[这里](http://clusterlabs.org/wiki/Main_Page)。这些模式实际上是通过CIB的资源分数、节点属性设置实现的。一定要注意，HA集群不仅仅支持两个节点、一个资源，完全可以支持多节点、多资源。
### CIB配置
　　CIB描述了集群全部信息供DC使用。
#### CIB内容
　　配置文件cib.xml默认在`/var/lib/heartbeat/crm/cib.xml`，最简单的CIB包括以下几项。

```xml
<cib crm_feature_set="3.0.7" validate-with="pacemaker-1.2" admin_epoch="1" epoch="0" num_updates="0">
  <configuration>
    <crm_config/>
    <nodes/>
    <resources/>
    <constraints/>
  </configuration>
  <status/>
</cib>
```

* **cib** CIB配置文件根节点 →[参考][cib-link]
	* **configuration** 集群配置信息的根节点   
		* **crm_config** 集群全局配置   
			* **cluster_property-set** 属性配置 →[参考][cluster-property-set-link]
		* **nodes** 集群各个节点的配置 →[参考][nodes-link]
			* **instance_attributes** Node参数配置   
		* **resources** 服务资源，TFS集群搭建HA使用RA资源 →[参考][resources-link]
			* **group** 可选，将多个资源合并为一组同时执行操作 →[参考][group-link]
				* **primitive** 配置资源及其属性，包括资源的分数(Score) →[参考][primitive-link]
				* **instance_attributes** OCF_RA所需参数配置，`crm ra meta OCF_RA_XXX`命令查询 →[参考][instance-attributes-link]
				* **meta_attributes** 配置资源的属性 →[参考][meta-attributes-link]
				* **operations** 操作配置，操作必须是OCF RA支持的操作 →[参考][operations-link]
		* **constraints** 资源在不同节点上的切换规则   
			* **rsc_location** 资源定位约束，确定不同节点上资源切换优先级（Score），也即针对同一资源，节点的优先级 →[参考][rsc_location-link]
				* **rule** 直接使用rsc-loaction可以实现优先级配置，使用rule可以使得配置项更加灵活易改，它能够精确配置节点的分数。rule不仅仅可以[应用于location][rule-location-link]，还可以[应用于meta_attributes][rule-meta-attributes-link] →[参考][rule-link]
			* **rsc_order** 可选，配置多资源的启动/停止顺序，多个配置之间是AND关系；可以使用[Ordered Set](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-sets-ordering.html#_ordered_set)的方式完成更多的顺序方式 →[参考][rsc-order-link]
			* **rsc_colocation** 可选，配置多资源的依赖；也可以使用[Colocation Set](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-sets-colocation.html)完成更多方式 →[参考][rsc-colocation-link]
		* **rsc_defaults** 全局资源默认配置。如果在<resources/>中对资源进行了配置，以<resources/>中的为准，否则以这里配置的为准   
	* **status** 集群每个节点的历史资源。集群根据该节点的内容构建完整的状态。注意该节点主要由LRMd在内存中操作，不会存盘，也不需要人为编辑，使用`#cibadmin --query|-Q`命令可以查看该节点的内容。   

> 配置中的score、stickiness等容易混淆，在配置时只要简单记住score用在constraints中，stickiness用在resource中即可。有关详细内容请参考[《[HA]Pacemaker的分数机制和资源黏性》](http://smartmz.github.io/2016/02/18/ha-pacemaker-score-and-stickiness)。

##### CIB例子
　　下面通过TFS搭建HA时的CIB配置来说明一些关键的地方。

```xml
<cib epoch="37" admin_epoch="0" validate-with="pacemaker-1.0" crm_feature_set="3.0.1" have-quorum="1" dc-uuid="7508b0ea-78aa-4011-917a-fe5f5d7082cd" num_updates="184">
  <configuration>
    <crm_config>
      <cluster_property_set id="cib-bootstrap-options">
        <!--配置了4个参数-->
        <nvpair id="cib-bootstrap-options-dc-version" name="dc-version" value="1.0.12-unknown"/>
        <nvpair id="cib-bootstrap-options-cluster-infrastructure" name="cluster-infrastructure" value="Heartbeat"/>
        <nvpair id="cib-bootstrap-options-symmetric-cluster" name="symmetric-cluster" value="true"/>
        <nvpair id="cib-bootstrap-options-stonith-enabled" name="stonith-enabled" value="false"/>
      </cluster_property_set>
    </crm_config>
    <nodes>
      <!--两个节点-->
      <node id="268a3f70-2d62-4003-a323-0cd946fdc100" uname="yf04501" type="normal">
        <instance_attributes id="nodes-268a3f70-2d62-4003-a323-0cd946fdc100">
          <nvpair id="nodes-268a3f70-2d62-4003-a323-0cd946fdc100-standby" name="standby" value="off"/>
        </instance_attributes>
      </node>
      <node id="7508b0ea-78aa-4011-917a-fe5f5d7082cd" uname="yf04401" type="normal">
        <instance_attributes id="nodes-7508b0ea-78aa-4011-917a-fe5f5d7082cd">
          <nvpair id="nodes-7508b0ea-78aa-4011-917a-fe5f5d7082cd-standby" name="standby" value="off"/>
        </instance_attributes>
      </node>
    </nodes>
    <resources>
      <!--使用一个group将两个资源划分为一组，同时启动、停止、监控-->
      <group id="ns-group">
        <!--资源1：ocf类型，名字为ip-alias，RA使用IPaddr2，调度VIP-->
        <primitive class="ocf" id="ip-alias" provider="heartbeat" type="IPaddr2">
          <instance_attributes id="ip-alias-instance_attributes">
            <!--给RA传参，RA需要的参数可以通过 # crm ra meta XXX 命令查询-->
            <nvpair id="ip-alias-instance_attributes-ip" name="ip" value="172.16.138.252"/>
            <nvpair id="ip-alias-instance_attributes-nic" name="nic" value="eth1:1"/>
          </instance_attributes>
          <meta_attributes id="ip-alias-meta_attributes">
            <!--设置集群中节点资源的允许状态为 允许运行 的-->
            <nvpair id="ip-alias-meta_attributes-target-role" name="target-role" value="Started"/>
          </meta_attributes>
          <operations>
            <!--添加操作，只配置了monitor，每2s执行一次-->
            <op id="ip-alias-monitor-2s" interval="2s" name="monitor"/>
          </operations>
        </primitive>
        <!--资源2：ocf类型，名字为ip-alias，RA使用NameServer，调度TFS NS进程-->
        <primitive class="ocf" id="tfs-name-server" provider="heartbeat" type="NameServer">
          <instance_attributes id="tfs-name-server-instance_attributes">
            <nvpair id="tfs-name-server-instance_attributes-basedir" name="basedir" value="/data/tfs"/>
            <nvpair id="tfs-name-server-instance_attributes-nsip" name="nsip" value="localhost"/>
            <nvpair id="tfs-name-server-instance_attributes-nsport" name="nsport" value="10000"/>
            <nvpair id="tfs-name-server-instance_attributes-user" name="user" value="root"/>
          </instance_attributes>
          <meta_attributes id="tfs-name-server-meta_attributes">
            <!--设置集群中节点资源的允许状态为 允许运行 的；设定资源启动成功的初始分数为 正无穷；设定失败后设为 负无穷-->
            <nvpair id="tfs-name-server-meta_attributes-target-role" name="target-role" value="Started"/>
            <nvpair id="tfs-name-server-meta_attributes-resource-stickiness" name="resource-stickiness" value="INFINITY"/>
            <!--resource-failure-stickiness已经弃用，改用migration-threshold-->
            <nvpair id="tfs-name-server-meta_attributes-resource-failure-stickiness" name="resource-failure-stickiness" value="-INFINITY"/>
          </meta_attributes>
          <operations>
            <!--配置NameServer操作，monitor每2s执行一次,start和stop不会循环执行-->
            <op id="tfs-name-nameserver-monitor-2s" interval="2s" name="monitor"/>
            <op id="tfs-name-nameserver-start" interval="0s" name="start" timeout="30s"/>
            <op id="tfs-name-nameserver-stop" interval="0s" name="stop" timeout="30s"/>
          </operations>
        </primitive>
      </group>
    </resources>
    <constraints>
      <!--针对tfs-name-server这一资源设置不同节点的分数-->
      <rsc_location id="cli-prefer-tfs-name-server" rsc="tfs-name-server">
        <!--添加规则，如果#uname为yf04401，则该节点的分数设置为 正无穷-->
        <rule id="cli-prefer-rule-tfs-name-server" score="INFINITY" boolean-op="and">
          <expression id="cli-prefer-expr-tfs-name-server" attribute="#uname" operation="eq" value="yf04401" type="string"/>
        </rule>
      </rsc_location>
    </constraints>
    <!--添加全局默认资源配置，初始分数配置为100，在resource节点中，tfs-name-server资源覆盖了该配置，ip-alias资源使用该配置-->
    <rsc_defaults>
      <meta_attributes id="rsc_defaults-options">
        <nvpair id="rsc_defaults-options-resource-stickiness" name="resource-stickiness" value="100"/>
      </meta_attributes>
    </rsc_defaults>
  </configuration>
  <status>
    ......
  </status>
</cib>
```
#### 如何操作CIB
　　官方给了三条**规定**：
　　1. 不要手动操作cib.xml
　　2. 遵循第1条
　　3. 如果不遵循第1条，集群有权拒绝使用手动更改的CIB   
　　鉴于此，我们需要通过 `cibadmin` 更新CIB。所有的更新及时生效并同步到所有集群节点。
##### 查询（Query）
```sh
#cibadmin --query|-Q
#cibadmin --query|-Q --obj_type {Xml_Node}
```
　　第一条命令查询内存中整个CIB配置，第二条查询CIB的一个XML节。这两条命令可以结合修改命令对内存中的CIB配置进行修改：

```sh
# cibadmin --query > tmp.xml
# vi tmp.xml
# cibadmin --replace --xml-file tmp.xml
```
　　结合第二条命令可以只对指定的XML节进行修改，降低修改出错的风险。
##### 删除（Delete）
```sh
cibadmin --delete|-D --crm_xml '{XML_Tag}'
```
　　一些情况下需要删除部分CIB配置，就可以使用上面的命令，比如

```sh
# cibadmin -Q | grep stonith
<nvpair id="cib-bootstrap-options-stonith-enabled" name="stonith-enabled" value="false"/>
# cibadmin --delete --crm_xml '<nvpair id="cib-bootstrap-options-stonith-enabled">'
```
　　注意 {XML_Tag}需要指明Tag的id.
##### 更新（Update）
```sh
# cibadmin --replace --xml-file tmp.xml
```
　　更新CIB配置和查询命令配合使用，参考上面查询命令的使用。大多数的CIB配置更新其实不需要直接修改xml文件，而使用其他命令代替cibadmin，比如：

```sh
# crm_attribute --name stonith-enabled --update 1/0 开启/关闭Stonith
# crm_attribute --type crm_config --attr-name symmetric-cluster --attr-value true/false 是否可以在集群的任何节点上迁移服务
# crm_attribute --type rsc_defaults --name resource-stickiness --update 100 设置全局默认资源配置
```
> 官方提供了沙箱操作来测试更新后的CIB，可以参考[Making Configuration Changes in a Sandbox](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-config-sandboxes.html)，如果需要，请自行学习，文中不再详述。

### CRM shell
　　启动Pacemaker服务后，更新CIB配置即可让集群运转起来。这里介绍一个工具 crmsh，该工具可以查看当前集群中的资源运行情况，包括查看目前资源的master节点，配置是否正确，集群系统支持的资源类型等，当然也可以对CIB资源进行直接操作配置。   
　　从pacemaker 1.1.8开始，Crmsh发展成一个独立项目，pacemaker中不再提供，[官方网站](https://savannah.nongnu.org/forum/forum.php?forum_id=7672)提供下载。下载RPM包结合yum解决依赖安装完成。   
　　在终端直接输入`crm`就进入CRM Shell控制台。按两下`TAB`键提示当前支持的命令，`help`可以查看帮助信息，`help CMD`可以查看具体命令的帮助信息。
##### 查看配置
```sh
# crm
crm(live)#
crm(live)# configure
crm(live)configure# show
node $id="3b8aa51e-88c4-4e3a-8fdb-2311ca791307" tc102
node $id="9b08998a-46c9-43f8-b0a9-18c7cb3acf5f" tc101
primitive ip-alias ocf:heartbeat:IPaddr2 \
	params ip="10.73.14.105" nic="eth1:1" \
	op monitor interval="2s" \
	meta target-role="Started"
primitive tfs-name-server ocf:heartbeat:NameServer \
	params basedir="/data0/tfs" nsip="localhost" nsport="10000" user="root" \
	op monitor interval="2s" \
	op start interval="0s" timeout="30s" \
	op stop interval="0s" timeout="30s" \
	meta target-role="Started" resource-stickiness="INFINITY" resource-failure-stickiness="-INFINITY"
group ns-group ip-alias tfs-name-server
property $id="cib-bootstrap-options" \
	dc-version="1.0.12-unknown" \
	cluster-infrastructure="Heartbeat" \
	symmetric-cluster="true" \
	no-quorum-policy="ignore" \
	stonith-enabled="0"
rsc_defaults $id="rsc_defaults-options" \
	resource-stickiness="100"
``` 
　　结果展示了CIB配置，比起Xml更加简洁明了。
##### 检查配置
```sh
接上
crm(live)configure# verify
ERROR: tfs-name-server: attribute resource-failure-stickiness does not exist
WARNING: tfs-name-server: specified timeout 30s for start is smaller than the advised 90
WARNING: tfs-name-server: specified timeout 30s for stop is smaller than the advised 180
crm(live)configure#
```
　　提示我们tfs-name-server资源没有`resource-failure-stickiness`属性。这是因为在Pacemaker 1.0以后，`resource-failure-stickiness`属性被`migration-threshold`属性替代了。
##### 修改配置
```sh
接上
crm(live)configure# edit
```
　　进入vim编辑模式，编辑内容为之前crm show的内容，而不是xml内容，将 `resource-failure-stickiness="-INFINITY"` 修改为 `migration-threshold="-INFINITY"`，保存退出。crm会自动检测编辑后的配置，如果还有ERROR，会询问是否继续编辑，如果选择否，之前的编辑不会应用。再次调用`verify`检查配置，已经没有ERROR提示了。   
　　修改全部完成后提交一下：

```sh
接上
crm(live)configure# commit
``` 
　　Crm Shell其他的功能可以参考它的帮助文档。

[rule-location-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_using_rules_to_control_resource_options.html
[rule-meta-attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_using_rules_to_control_cluster_options.html
[rule-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch08.html
[primitive-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/primitive-resource.html  
[instance-attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_resource_instance_attributes.html
[meta-attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-options.html#_resource_meta_attributes
[operations-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_resource_operations.html
[cib-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch03.html#_cib_properties
[cluster-property-set-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_cluster_options.html
[nodes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch04.html
[instance_attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-node-attributes.html
[resources-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch05.html
[group-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch10.html#group-resources
[rsc-location-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_deciding_which_nodes_a_resource_can_run_on.html#_location_properties
[rsc-order-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-ordering.html#_ordering_properties
[rsc-colocation-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-colocation.html#_colocation_properties

