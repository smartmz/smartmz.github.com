---
layout: post
title: "聊聊有关数据的一些基本概念和常见误区"
date: 2017-06-14 00:00
keywords: software
description: 数据相关
categories: [data]
tags: [data]
group: data
icon: th-large
---

<!-- more -->

大数据一直是大家谈论的热点，但是对其中的一些基本概念，例如：数据源、数据元、元数据等等，大家是否觉得容易溷淆、不易区分清楚呢？本期作者将结合自己的实际经验及体会，将这些容易溷淆的概念为大家逐一阐述。

近期团队的大数据公众号受到了越来越多朋友们的关注，而大数据本身作为一个行业热点一直也是大家日常工作乃至茶馀饭后讨论的重点话题。过去一段时间以来，笔者也和国内许多金融机构（银行、农合机构、基金公司、资产管理公司、保险公司等）交流和探讨数据相关的工作事项。

通过交流，笔者发现一个特别有意思的现象，那就是朋友们聊起大数据来都是一副畅所欲言的架势，侃侃而谈、各抒己见，毕竟数据和自己日常的工作和生活是分不开的，而且能看出来都是饱受了数据的困扰（笑）。期间所引用的一些措辞和描述都不尽相同，虽然大概都能明白是什麽意思，但是也引起了一些困扰，特别是对于管理层以及业务人员，甚至也包括一些专业数据和技术人员。

藉此机会，笔者想抛开一些官方能查到的正式定义和术语解释，结合自身的实际经验和体会，把一些关键的数据基本概念以及大家容易溷淆的常见误区进行释疑，希望能够对朋友们有所帮助。以下是笔者整理的十大基础概念组合，由于内容比较多，我们会分上、下两篇分别来聊一聊！

（一） 数据、信息、大数据

既然说到数据，那麽首先就从最基本的概念入手吧，那就是数据、信息和大数据，这三者之间到底是什麽含义和关係。

我们把这三个词分成两组来解释，先说数据和信息。数据，顾名思义，是数字化的凭据，比如1234、ABCD等，以二进位来处理，所有的数据都可以用0和1来记录，形成数据化的凭据。说它是凭据（或单据），就是因为数据本身没有任何含义，就是一份记录，只是人们把想表达的意思记录下来所形成的凭据。而信息则不同，信息所体现的正是人们想表达的这层含义。举个例子，大家的电话号码，139XXXXXXXX，单就这11位数字而言就是一组数据，没有应用场景的话其本身不体现任何有价值的信息，只有在打电话的时候，人们用到这组数字拨号，才体现出它的价值，即某个联繫人的电话号码。因此，数据是信息的载体，信息多是複杂的、不规范的，但是数据可以是简单的、规范的，简单到能够用标准化的二进位语言0和1来表示。所以，在一般情况下，计算机处理的是标准化的数据，而人脑处理的是複杂的信息。

马云提出的从信息技术（IT）转变为数据技术（DT）的发展战略，反映的就是去繁从简的理念，从本身没有含义的数据出发，挖掘出有意义的信息，并支持业务经营和管理决策。简单总结一下，就是人先闭嘴（「先」字很重要，不要误会了），让数据说话！

再来说数据和大数据。大数据这个词可以说最近几年被用滥了，全世界都在谈论大数据，每家企业都说应用大数据（笑）。相对数据而言，大数据到底大在哪裡？我们这麽来理解，首先是体现在数据量上，大数据是海量数据，传统的技术处理不了，好比大家日常用单机Excel处理超过多少万条数据就死机了是一个道理，当然海量可远不止这个量级。至于海量具体是多少，有没有标准的说法？笔者想说，你能想的多远、多广，这个海量就有多大（笑）。其次就是数据的范围和类型，不仅仅是机构内部的数据，还有好多外部的数据；不仅仅是结构化的数据，还有非结构化的数据。说到结构化，此前笔者也提到了，一般情况下计算机处理的是标准化的数据，这就是结构化数据。但是，还有好多文本、视频、语音等并非标准化的，这就是非结构化数据。当然，通过一定的技术手段，还是可以把非结构化数据转化为结构化数据并进行处理，看来计算机的二进位仍然是王道啊。

说明一下，大家也千万不要被大数据的「大」所迷惑了，非得追求大数据应用，非要用海量的、外部的、非结构化的数据等。邓小平说过，管它黑猫白猫，能抓住老鼠的就是好猫。翻译过来，管它是大数据还是一般数据，能支持经营管理决策的就是好数据。说到这裡，笔者不禁又想起金庸小说天龙八部里的桥段，鸠摩智向大理天龙寺一众高僧展示各种眩目的武功，并表示希望以此交换六脉神剑秘籍，在众高僧被鸠顾问忽悠的眼红、心痒及矛盾纠结的时候，住持一句话就点醒众人：就你们这样连本派一阳指（都不说六脉神剑）都没练到位的，还有脸想学其他门派的武功！大数据应用也是一样的道理，先把企业自身积累的数据好好用起来，修炼自身内功，把以数据为驱动的管理理念和应用模式建立起来后，再叠代进行大数据的应用提升吧。

（二） 数据源、数据元、元数据

标题中的这三个术语算是业内比较容易把人弄晕的，相信很多朋友都有亲身经历过，更别提后来不知道哪位同仁又造出来一个「源数据」，这四个词的关係就更乱了（笑），各种YUAN，应该怎麽才能把它们说的圆啊。

先说数据源，字面上理解就很容易明白，这指的是数据的来源，比如数据来源的信息系统、数据来源的表格等。举个实际例子大家就更容易理解了，「合约信息的数据源是核心系统」，核心系统就是合约信息的数据源了。数据源这个词在企业使用的频率很高，那是因为数据源不一致（或不唯一）是企业数据质量低、数据打架的重要原因，所以统一数据源是企业的基础性数据工作。

而「源数据」这个词是后来不知道谁造出来的，可以认为这是一个口语化的用词，实际不应该作为一个正式的术语。它的产生笔者认为和「数据源」是有很大关係的，即源数据就是来自于特定数据源的原始数据，这麽解释还是很绕吧（笑）。还是沿用上面的例子，我们说「合约信息的数据源是核心系统」，那麽来自于核心系统的合约信息就是源数据，或者称为原始数据，是后续数据加工处理的源头数据，这样大家就容易理解了吧。数据源的主语是在「源」上，指的是来源，其本身并非数据；而源数据的主语是「数据」，来自特定源的数据。当然，在实际应用中，笔者不建议用这个口语化的词，因为确实比较不容易理解并引起溷淆，建议大家直接说具体的数据即可。

好，再来说说数据元。数据元的「元」指的是元素，即数据元素，你可以简单理解为数据项。比如「贷款馀额」就是一个具体的数据项，把它抽象起来就形成一个数据元素。那麽为什麽要进行抽象并形成数据元？其目的是为了对这些数据元素进行标准化和规范化，以便统一使用。财政部XBRL准则中引用的就是数据元（数据元素）的概念，把一项项数据进行抽象、定义和规范，形成基础元素，以便在财务报表中组合使用。在其他外部监管机构发布的各类标准中，数据元也是基本的要素，形成了数据元目录，并提供统一和标准化的定义，作为行业标准。

元数据中的「元」含义则和数据元的「元」不一样了，指的不是元素，而是，怎麽说，笔者暂时还想不到一个比较合适的词来解释，因为这个「元」太高、大、上了，和「元始天尊」、「天元」中的「元」是一个意思，而且你还不好解释为「原始」，否则就成原始天尊，变成了原始人，这级别和地位一下就拉下来了（笑）。可能「本质」、「本源」这些词更能用来解释，但似乎也不是那麽准确。所以，笔者试着这麽来解释吧，「元」是很高大上的，元数据就是数据中的数据，是最大的！那麽如何理解数据中的数据？那就是，用来解释、定义数据的数据，我们称之为元数据。如果大家还是被绕晕的话，那我们举个例子吧，例如上面说到把「贷款馀额」抽象为一个数据元，那麽「贷款馀额」的业务定义、统计口径和计算规则、管理属性、和其他数据的关联关係等描述性的数据就是「贷款馀额」这个数据元的元数据了。这麽看来，元数据的重要性就显而易见了，连数据标准都属于元数据的范畴了，元数据管理也就成为了保障、提升数据质量的重要手段。

（三） 数据治理、数据管理、数据管控

既然前面已经都提到了元数据以及元数据管理，那麽接下来就来聊聊和数据管理相关的几个概念。

数据治理、数据管理、数据管控是目前最容易被互相替代使用，且不太影响其表达含义的三个词。在实际使用中，大家确实也常在各种场景下「随机」使用这三个词，不过最近数据治理被使用的频率相对比较高一些。本着精益求精的精神，笔者还是试着来解释这三者之间细微的区别吧！

首先来说数据治理，相信大家看到这个词后会很快联想到「公司治理」。确实，数据治理本身属于一种公司治理活动，而且区别于一般的管理和管控活动，数据治理强调的是从企业的高级管理层及组织架构与职责入手，建立企业级的数据治理体系，自上而下推动数据相关工作在全企业范围的开展。可以说，数据治理是数据工作的顶层架构设计。

相对的，数据管理则更多的偏重于管理流程方面，涵盖了不同领域的数据管理流程和内容，包括数据需求管理、数据认责管理、数据标准管理、元数据管理、数据安全管理、数据质量管理、数据评价管理等各个领域，这是数据工作的核心内容。

而数据管控就更偏执行层面了，其重点在于如何执行和落地实施，涉及到具体的管控措施和手段。

因此，数据治理、数据管理和数据管控体现了自上而下的管理层级，治理的重点在于管理架构和体系，管理重点在于流程和机制，管控重点在于具体措施和手段。这三者之间是相辅相成的，缺一不可。前面提到了最近业界比较经常用到「数据治理」，启动的专项工作也多以数据治理来命名，这主要是因为在过去几年中，许多金融机构实际上都已经开展了一系列数据管理和数据管控的具体工作，但是主要都是以信息科技部门牵头，配合信息系统建设为主要目的。这种自下而上的推进方式，其实际成效往往不是特别显着，很难解决企业在业务经营和管理上存在的实际用数困难。这也是为什麽现阶段大多数金融机构都逐渐意识到在企业战略层面推动数据治理的重要性和必要性，并启动数据治理相关项目。对于数据治理工作具体如何开展，有何切实、有效的实施策略，这属于一个专项课题，笔者此前也发布过有关数据治理的一篇文章，各位朋友有兴趣的话可以查阅公众号的历史文章目录。

（四） 数据标准、数据规范、数据字典

聊完数据治理，接下来很自然的就要谈到数据治理的核心内容之一，那就是数据标准。

数据标准相信各位业内人士都经常接触到，包括外部的金标、银标等行业标准以及企业内部的标准，都属于数据标准的范畴。然而，在很多场景下，特别是早些年企业所制定的所谓数据标准，其实往往是更偏向于数据字典的概念，其内容还没有达到数据标准的要求。因此，接下来我们就先来说明一下数据标准和数据字典的区别。

首先来看数据标准应该包括的完整内容，即业务属性标准、管理属性标准、技术属性标准。业务属性标准指的是数据元的业务相关属性，包括名称、业务定义、统计规则和逻辑等，这都是需要数据的业务归属部门负责进行定义的；管理属性标准指的是数据元的管理过程属性，包括归属部门、使用部门、管理部门、加工系统、存储系统、应用系统以及数据的生命周期关係等内容，需要业务部门和技术部门共同确定；技术属性标准则偏重于数据元的技术规范，包括数据格式、编码规则、代码取值、库表栏位名称等，一般由信息科技部门进行定义。

数据标准是企业级的标准化语言，既统一规范了部门间沟通的业务语言，又规范了系统间交互的技术语言。相对于数据标准，数据字典更偏重于某个或某类系统的技术属性标准，解决的主要是系统层面的开发和交互语言。

虽然在实际应用中，数据字典也可以是企业级的，可以统一规范企业所有信息系统的数据字典（往往比较难），但一般情况下只作为单一系统的数据字典，即多个系统有多套数据字典。究其主要原因，是由于数据字典对应的是系统的实际资料库表设计，但是往往很少有企业能够在企业级实现所有信息系统均按照同样的数据字典进行库表设计，这裡有历史遗留的原因，也有外购成熟软体（难以调整）的原因。

而数据标准就不一样了，它更关注于不同信息系统之间进行数据交互或数据整合时，需要遵循的统一标准。不同信息系统在进行数据交互时，如果相应数据字典的规则互不一致，则要按照统一的标准进行数据映射和转换，例如A系统的客户性别栏位名是「Customer_sex」，取值为1（男）和2（女）；B系统的客户性别栏位名为「Client_gender」，取值为M（男）和F（女），两个系统的数据栏位名和取值均不一致。那麽在两个系统在交互或进行企业级数据整合时，就需要按照统一的标准命名进行数据项的映射，并按照统一的代码取值进行转换方可实现。

此外，数据标准还是业务部门之间的标准化语言，例如X部门的统计报表中「贷款馀额」与Y部门统计报表中的「各项贷款馀额」实际的业务含义和规则如是一致的，那麽就应统一命名，避免管理层和部门之间使用报表数据时产生歧义。

因此，从内容范围上来说，数据字典必须涵盖系统的所有数据项，但数据标准主要针对跨部门、跨系统的共享数据项。

讲完数据标准和数据字典的区别后，相信大家就理解为什麽笔者提到以往很多企业做的数据标准实际都还是停留在数据字典的层面，那是因为一方面当时做的标准不是企业级的，没有得到所有业务部门的认可，而更多是科技部门主导的信息系统级的标准。另外就是标准的内容也多偏技术属性标准，业务属性和管理属性标准普遍缺失。再有就是标准的应用主要为了指导信息系统开发和建设，而不是为了规范业务部门用数。

当然，笔者在与很多金融机构的同事聊起这个问题时，大多数科技部门的同事都表示当时也是没办法的，科技部门自身很难（不论是行政管理还是业务能力上）推动企业级数据标准的建设，而且聘请的外部机构往往是系统实施厂商，项目过程中又没有和管理层及业务部门充分沟通，主要靠科技部门和系统厂商一起闭关修炼做出一套标准来，那自然而然就做成了偏系统和技术层面的标准了。而笔者与业务部门同事沟通时，却发现许多业务人员都表示自己不知道企业原来还有这样的数据标准。在了解完情况后，业务人员表示「噢，那是科技部做的吧，我们不太清楚」。是啊，连业务部门都没有推广并使用的数据标准，还能称之为企业级的数据标准吗？但是呢，最终的项目成果还是很「显着」的，为什麽呢，因为标准在系统上落地实施了！这也是目前笔者认为业界存在的一个重大误区，即检验数据项目成败的关键不在于业务应用和管理上是否有成效，而是在于系统上是否落地实施？！还是回到数据治理的本质，我们不是为了治理数据而开展数据治理工作，最终还是为了服务于企业的业务经营和管理用数需求，如果过分关注（当然还是需要关注）技术和系统层面的实施，而忽略了业务应用和管理，那麽真是本末倒置了。

再来讲数据规范。可以这麽说，数据规范是一顶大帽子，既包括前面说到的数据标准、数据字典，也包括现在很多金融机构在做的业务术语规范、指标体系规范、数据模型规范等等。说到这裡，大家不禁会问，那指标体系、数据模型又是什麽呀？这裡笔者先卖个关子，请看下文分解（笑）。


---


聊聊有关数据的一些基本概念和常见误区（下）

上期文章为大家介绍了四组有关数据的基本概念和常见误区，大家是否对这四组概念有了更加清晰的认识呢？如果觉得意犹未尽的话，就请继续阅读本期后六组概念的分享吧~

在上一期文章中，笔者分享了四组有关数据的基本概念和常见误区，接下来还有六组要讲，任重而道远哪！那麽我们閒话少说，直接进入主题。如果说前四组比较偏纯概念的话，那麽我们接下来就从实实在在的系统平台开始吧。

（五） 数据集市、数据仓库、应用资料库、数据工厂

笔者此前在一家金融机构交流数据集市的时候，了解到许多同事对数据集市以及相关的几个概念理解的不是特别准确，这也给项目工作目标和计划推进造成了一定的困扰。业务人员都纷纷提出来，为什麽要建立数据集市，我们业务部门才不关心科技部是通过数据仓库、数据集市还是什麽其他资料库来实现，我们要的就是简简单单的能够满足日常业务用数需求的一个平台！

是的，笔者百分之百支持业务用户的这个观点。对于用户而言，只要能够满足其用数需求，通过什麽方式那是技术的问题。不过呢，如果我们打个比方，那麽大家就能看到原来数据集市和业务用户还是有着一定关係的（笑）。

首先，我们把企业看成一个家庭，把用户看成家庭成员，把数据看成一种商品（以食品为例吧）。

在家裡，孩子想喝饮料、吃水果等，可以直接去冰箱裡拿。妈妈煮饭炒菜煲汤，也去冰箱裡拿蔬菜、肉类等原材料。用户什麽时候想吃东西了，能够很方便的从冰箱裡直接取来食用，或进行加工后再食用。那麽，这个冰箱就像是应用资料库（或叫应用系统的资料库），冰箱裡的食品就是数据。应用资料库能够快速的解决业务用户日常的用数需求，想要直接拿就行了，快速便捷。

但是，冰箱本身存储的东西是有限的，如果冰箱裡的东西没了怎麽办？那麽当然是重新去超市採购喽，不过这个活不一定要所有的人来干，在家裡买菜这样的活一般还是由妈妈负责，这样看起来妈妈是管理员（当然同时也是用户），爸爸和孩子才是真正的业务用户啊！（笑）

好了，妈妈去超市买菜去了，这裡的超市就可以比做是数据集市。超市里分海鲜区、蔬果区、肉类区、饮料区等等，东西应有尽有，且按照客户的需要进行分门别类管理，便于大家进入相应的区域採购。那麽数据集市也是如此，按照业务应用需求，把数据进行分类，形成不同的主题区，比如风险主题集市存储了风险管理所需的数据，管会主题集市存储了管理会计所需的数据，监管主题集市存储了外部监管统计信息披露所需的数据等等，便于管理和使用。

妈妈买完东西后，再把食品存放到家裡的冰箱裡，以供所有家庭成员取用。同理，业务部门的管理员会定期从主题集市中获取所需的数据，并返给应用资料库支持日常用数，只不过往往这个动作不是人工来执行，而是定製好后通过系统接口自动批量处理。这样，数据集市就完成了对应用资料库的供数了！

噢，那麽大家自然而然就想到了，超市的货仓一定就是数据仓库啦！是的，这个货仓可以存储的东西非常多，虽然也是按照不同的商品分类进行管理（好比数据仓库的分区），但是用户一般不会直接到仓库里买东西，所以和用户来讲是相对隔离的。能接触仓库的就是库管员和送货员，这些人往往都是技术背景的，不属于业务用户的范畴。当然，事无绝对，不排除有些商家採取批发直销的模式，让用户直接去仓库里取货（国外有好多批发超市），不过这种情况毕竟较少了。所以，业务用户直接从数据仓库取数的情况也不是完全不存在的。

通过以上的生活实例，相信大家很容易能看出来数据集市与数据仓库、应用资料库之间的区别和联繫了。所以，数据集市对业务部门来说还是很有意义的，因为至少同时作为管理员和用户的妈妈还是需要到超市里採购东西的，而且不排除作为真正业务用户的爸爸和孩子也愿意去逛逛超市，直接选择自己想要的产品（笑）。大家想想，如果家裡附近没有超市，那是一件多麽不方便的事啊，这就是建立数据集市的意义所在！

最近业界还出现一个很新、很火的概念，叫数据工厂，很多人不理解它的含义和定位。没关係，还是拿上面的例子来解释。大家都知道，去超市买完鱼后回来得自己去鳞、取鳔等，一系列动作特别繁琐且不便，现在的年轻人一般都懒得做。好了，如果说有这样的数据工厂，能够按照用户的需求进行数据的定製加工，帮你把这一系列麻烦事做了，甚至还可以再进一步定製化的把鱼给清炖或红烧了，那麽用户就可以直接获得加工好的数据产品，何乐而不为？不过，这个加工厂如何选址、按什麽工序加工、如何配送那是科技部的事，用户可以不用管了，提出对最终产品的要求即可。

（六） 数据平台、大数据平台、元数据平台、数据服务平台

数据平台是一个很大的概念，但是这个概念在日常工作中往往被用的窄了。比如，很多业务用户认为数据平台就是报表平台、或者数据仓库，这和大家关注点、出发点不同是有很大关係的。

这麽说吧，数据和信息是对等的，平台和系统是对等的，那麽数据平台和信息系统也就是对等的。大家都知道，信息系统是多麽大的概念啊，那麽数据平台也是如此。有些金融机构说我们计划要建个数据平台，想问问专家有什麽建议。这就好比说我想建个信息系统，请你告诉应该怎麽做一样让人难以具体回答，因为还没说清楚要建什麽样的信息系统。所以，咱们首先还是要清楚数据平台到底包括哪些范围和类型。

一般来说，数据平台从大面上可以分为：数据整合平台、数据管理平台、以及数据应用平台。当然，这只是第一层级的划分，不过目前业界的数据相关平台都可以归到这三大类中。

数据整合平台指的是把数据从源系统进行收集和整合、加工处理、存储和共享的平台。数据整合平台的首要目标是形成企业级统一、共享的整合数据，为后续的数据应用提供基础。当然，整合平台也不仅仅是简单整合，还包括一定的数据加工。上面提到的数据集市、数据仓库、数据工厂都属于数据整合平台的范围。

那麽大家自然就会问，那麽大数据平台呢，是否属于数据整合平台的范围？严格意义上不完全是，就看这大数据平台是广义的还是狭义的了。应该这麽讲，广义的大数据平台和数据平台是在一个层级的，它也包括了大数据的整合平台、管理平台和应用平台。不过目前业界所说的大数据平台一般还是偏狭义的，即只包括大数据的整合平台。也就是说，狭义的大数据平台指的就是把海量的大数据进行整合和存储的平台。

区别于传统的数据整合平台，大数据平台架构由于其处理的数据类型和性质而与传统平台存在较大不同。在此前已经和大家聊过大数据的特点，那麽大数据平台比较特别的地方就在于它可以高效处理海量的历史数据、外部数据、特别是非结构化数据，一般是基于Hadoop分布式架构，具有高效性、高容错性和高扩展性的特点。而传统的数据整合平台（如数据仓库）多为关係型资料库，仅比较擅长处理结构化的数据。

再来说数据管理平台，数据管理平台的目标是实现数据全生命周期的管理，确保前面提到的数据治理、管理和管控措施的系统落地，保障数据质量。元数据平台就属于数据管理平台的范畴，通过元数据的管理最终实现对数据的管理。

数据管理平台的用户主要是数据管理部门和技术部门，当然也包括数据的业务用户。不过对于业务用户而言，数据管理平台最重要的意义主要体现在，一是可以查询并了解企业级的数据规范，包括数据的业务定义、统计规则等规范语言，二是在发生数据变化后，能够进行上下游的提示。例如，数据源发生变化后影响到后端的报表应用，报表用户能够及时获取变化信息，评估对自身用数的影响并制定应对措施。这个点其实非常重要，笔者在与很多业务部门同事沟通的过程中，许多人都提出了对此类上下游数据问题的不满，好比财务部的同事会抱怨前台部门修改数据前、后都没有及时和财务部进行沟通，最后报表都已经对外报送了，但是前台数据修改后就又导致业财不一致了。这类问题可通过数据管理平台解决。

最后再来说说数据应用平台，这也是业务用户最关心的、直接使用的用数平台。企业的报表平台、统计报送平台、指标系统、管理驾驶舱系统、决策支持平台、以及高阶的数据分析和挖掘工具等都属于数据应用平台的范畴。简单一句话概括，就是支持最终业务用户日常用数的应用交付平台。不论是数据整合平台、还是大数据平台，最终都是服务于业务用户的用数需求，虽然在应用模式上有所区别，有初阶的固定报表、数据模板应用，也有中阶的指标多维灵活查询、决策驾驶舱应用，还有高阶的大数据分析和挖掘应用，都需要通过数据应用平台进行实现数据服务的最终交付。

很多同事还会问，那麽数据服务平台又是什麽？和数据应用平台有何区别？以笔者个人的观点，数据服务和应用都属于一个范畴，即服务于用户的视角，实在没必要进行如此独立的平台区分，许多金融机构也是只建设统一的数据应用平台。如果非要区分两者间的区别的话，那麽笔者认为，数据应用更偏重于最终的数据结果交付，比如说用户通过应用介面获得想要的报表或指标；而数据服务更偏重于过程的调度，比如进行用户用数偏好分析以便选择用户可能需要的数据进行主动推送等。

（七） 基础数据、主数据

上面讲完有关係统和平台的两组概念后，接下来又要回到比较抽象的几组概念了，不过大家不用担心，笔者会儘量用简单的语言快速说明，大家别打瞌睡（笑）。

先来聊聊基础数据。基础数据是相对于衍生数据而言的，是从业务前端直接产生和採集的，未进行过加工计算的基础性数据。例如，业务交易产生的交易流水数据、採集的客户基础数据、签署的合约数据等，都属于基础数据的范畴。基础数据是企业开展日常业务经营所直接产生的，为中、后台的风险管理、财务核算、管理分析和统计应用等提供了数据基础。

基础数据的缺失和质量不高是目前业界碰到的最大难题。许多企业在开展数据统计和分析时，都或多或少遇到由于基础数据缺失而导致难以进行准确的分析和预测，无法更好的支持经营管理与决策，这个困难直接体现了企业的业务前台和中、后台管理之间的先天矛盾。

从前台的角度来讲，其主要的任务在于提高业绩、提升客户体验。基础数据的採集和维护一般都需要前台业务部门人员来执行，而这项工作却往往没有纳入其绩效考核的内容，因此往往引不起前台业务人员的重视。另一方面，在业务交易过程中进行数据的採集和维护也是需要一定的时间成本，如果在客户办理业务时过分强调信息的高质量採集，也会影响业务的办理效率和客户体验。而中、后台人员为了满足其管理目的，则希望前台能够儘可能多的採集所需的基础数据。看起来，这还真是一个难以调和的矛盾啊。笔者此前就曾遇到这样一种情况，企业的后台部门为了获取统计报送所需数据，提出了前台业务系统改造的数据需求，但是前台业务部门以影响业务效率为由拒绝接受，导致了相关工作难以顺利推进。相信许多中、后台管理人员都有类似的体会吧！

所以，为了更好的协调这个先天的矛盾，我们还是要站在前台业务的视角来解决问题，而不是一味的互相推卸和指责。首先要找到一个契合点，即哪些数据是前、中、后台部门，特别是前台部门关心的数据，这些数据又能如何解决前台部门的用数需求，那麽就能够求同存异，找到关键数据范围作为切入点，并以应用叠代的方式逐步完善基础数据的採集。例如，我们曾帮助企业建立客户交叉销售、客户维挽等数据分析模型以便更好的支持业务经营，通过这类应用项目，前台部门深刻意识到客户数据採集的重要性，并主动开展客户基础数据质量的提升工作。

在基础数据中，有一类数据非常重要，作为不同系统之间交互和共享的关键数据，我们把它称之为「主数据」。例如，不同业务系统都会有客户的数据、产品的数据，那麽如果每个部门、系统都分别维护这些数据，必然会造成不一致。因此，需要在企业级建立统一的主数据管理，如客户主数据、产品主数据、机构主数据等，在全公司范围内都应是统一且唯一的，并确定部门和系统间交互识别这些主数据的唯一标识，如客户编号、产品编号、机构编号等。那麽，这些唯一标识也成为了主数据中的主数据了，是开展主数据管理首先要统一、整合的对象。虽然这是项非常基础性的工作，但是许多企业却不见得做的好，例如有些金融机构的客户编号就没有在企业级范围进行统一，系统间的客户信息无法唯一识别，同一客户可能在机构内存在多条记录却并未整合等等。我们经常说，解决问题就要抓住问题的主要矛盾，那麽主数据问题就是所有数据问题中需要迫切解决的关键性问题了。

（八） 衍生数据、指标

基础数据和衍生数据是相对的两个概念，讲完基础数据，那自然就要说到衍生数据。

衍生数据是指按照一定的规则和逻辑，对已有的数据进行计算和加工后形成的数据。相对于基础数据可直接由前台部门採集而言，衍生数据一般是基于基础数据或其他衍生数据的再加工。

衍生数据常见的质量问题和基础数据不同，基础数据难在採集这个环节，而衍生数据却难在统一口径规范上。一般来说，金融机构的前、中、后台部门由于其视角和目的不同，对特定衍生数据会有不同的计算规则和口径，这也就导致了数据不一致问题。但是，这类问题解决起来相对基础数据还是简单一些，只要企业能够自上而下统一进行标准化的规范，明确衍生数据的唯一定义部门和加工系统，那麽很多问题都迎刃而解了。不过说起来容易做起来难，这裡面仍然需要协调各部门的不同利益和需求。

相对于基础数据中的主数据，衍生数据中也有一些非常关键的数据，用于满足管理层日常的经营管理和决策支持，我们把这些衍生数据称之为指标。

笔者在和业务人员交流的时候，其实很多人都不太清楚衍生数据的概念，但是只要提到指标，大家一下子就明白了，毕竟日常工作中都用到了各种指标。如果非要来区分一下衍生数据和指标的话，那麽可以这麽理解，衍生数据是一个广泛的技术概念，但是指标更偏重于业务应用，这也是为什麽许多金融机构近期致力于建设企业级指标体系的原因，因为只有指标才是管理层和业务部门关心的，衍生数据离业务太远了（笑）。

接下来就简单说明一下有关指标体系的几个基本概念。

（九） 报表统计项、指标、维度、度量

指标是反映对象特徵属性的、可衡量的单位或方法，是具有（业务）意义的指向和标杆。

指标可分为基础指标和衍生指标，其中基础指标是具备宽泛定义的统计对象，从而为指标的灵活组合与多维分析提供基础。衍生指标是在基础指标的基础上，通过添加一个或多个统计维度形成新的指标、或通过不同指标进行运算而形成新的指标。一般来说，指标都属于衍生数据。

维度（或称统计维度、筛选条件）是对指标进行描述的不同视角，用于标示指标的不同方面的属性，例如对贷款馀额这个指标，可以按照产品类型维度（如个人消费贷款、个人住房贷款等）来分析，可按照资金投放行业维度（如农、林、牧、渔等）来分析、可按照贷款期限维度（如长期、中期、短期等）来分析、也可以按照贷款状态维度（如正常、不良等）来分析等。一般来说，维度都属于基础数据。

度量是针对指标而言，和维度并没有关係，是指标的度量衡，本身并无实际统计意义，如金额、馀额、比率、笔数、个数等。例如，贷款馀额这个指标的度量就是馀额。而报表项、统计项等无非就是指标的应用了，企业所抽象出来的指标都是来自于各类报表的具体报表项或统计项，并回过头来应用于不同的报表。

有关具体如何建设企业级指标体系，笔者在这裡就不展开了，内容确实比较多，以后有机会可以专题讨论一下，很有意思的，特别是如何应用指标体系这块，能够有效解决企业的用数需求。

（十） 数据模型、数据分析模型、统计模型、数据建模

近期大数据分析特别火，所以很多朋友都在谈论数据分析和建模，这就引起了许多有关模型的概念了。

其实大家在谈的数据分析模型是一种统计模型（或者叫数学模型），是以数学统计的算法构建的複杂模型，满足于一定的应用场景。目前数据分析模型的应用场景较常用于两个方面，一是客户营销，二是风险管理。在风险管理领域，许多银行金融机构为了满足新资本协议的合规要求，建立了相应的风险计量模型，实际这也是属于统计模型的一种应用了。

数据分析模型一般需要输入大量的历史数据，按照既定的参数和算法进行模型计算后，输出对业务有意义的分析结果，例如有潜在产品需求的目标客户清单、特定价格策略下客户响应预测、存量贷款未来逾期的机率预测、最低资本要求等等。数据分析模型最终还是要服务于特定的业务经营和管理要求。

相对于数据分析模型，数据模型的含义可就不同了。数据模型指的是数据的结构和关係。最顶层的模型一般是概念模型，主要是从业务视角提出的对数据的概念性的需求表述。具体的数据模型一般会分为数据的逻辑模型和物理模型。逻辑模型指的是数据之间的逻辑关係，一般通过实体关係（ER）图来体现；而物理模型大家可以简单理解为资料库的表结构了。

数据分析模型更偏重于业务应用和决策支持，数据模型则更偏重于系统设计和实施，两者的目标是不同的。

解释完这几个模型的区别，那麽模型的构建（简称为「建模」）就很清晰了。业界说的数据建模一般指的还是数据模型的构建，但是描述不够准确，叫「数据模型建模」更加准确。而数据分析模型的构建则可以叫「分析模型建模」或「统计模型建模」。当然，在日常用语中，大家只说建模就可以了。

结束语：希望以上的概念解释和经验分享对朋友们理解相关术语有所帮助，许多观点也都是笔者个人的见解，不一定很准确，也算是抛砖引玉了，欢迎大家批评指正！

关于我们
我们是KPMG专业数据挖掘团队，）中，我们会在每周六晚8点准时推送一篇原创文章。文章都是由项目经验丰富的博士以及资深顾问精心准备，内容也是结合实际业务的理论应用和心得体会等乾货。欢迎大家关注我们的微信公众号，关注原创数据挖掘精品文章。如果想要联繫我们，也可以在公众号中直接发送想说的话与我们联繫交流。
即可关注！也请随手推荐我们给你的小伙伴 ↓↓↓↓

文章来源：毕马威大数据挖掘

喜欢这篇文章吗？快分享吧！
传统数仓转型，如何避开数据处理的深坑
【桉例】恆丰银行——基于大数据技术的数据仓库应用建设
网站数据分析：商业智能
关于命名规范、维度明细层及集市汇总层设计的思考
我所经历的大数据平台发展史-下篇 网际网路数据模型2
我所经历的大数据平台发展史：网际网路时代及数据模型
数据挖掘在金融和医学方面的商业应用
新核心业务系统数据架构规划与数据治理
网站数据仓库整体架构图及介绍
管理软体SAS： 商业智能从BI走向BA

壹读
聊聊有关数据的一些基本概念和常见误区（下）
 2017-02-11 20:01:07
上期文章为大家介绍了四组有关数据的基本概念和常见误区，大家是否对这四组概念有了更加清晰的认识呢？如果觉得意犹未尽的话，就请继续阅读本期后六组概念的分享吧~

在上一期文章中，笔者分享了四组有关数据的基本概念和常见误区，接下来还有六组要讲，任重而道远哪！那麽我们閒话少说，直接进入主题。如果说前四组比较偏纯概念的话，那麽我们接下来就从实实在在的系统平台开始吧。

（五） 数据集市、数据仓库、应用资料库、数据工厂

笔者此前在一家金融机构交流数据集市的时候，了解到许多同事对数据集市以及相关的几个概念理解的不是特别准确，这也给项目工作目标和计划推进造成了一定的困扰。业务人员都纷纷提出来，为什麽要建立数据集市，我们业务部门才不关心科技部是通过数据仓库、数据集市还是什麽其他资料库来实现，我们要的就是简简单单的能够满足日常业务用数需求的一个平台！

是的，笔者百分之百支持业务用户的这个观点。对于用户而言，只要能够满足其用数需求，通过什麽方式那是技术的问题。不过呢，如果我们打个比方，那麽大家就能看到原来数据集市和业务用户还是有着一定关係的（笑）。

首先，我们把企业看成一个家庭，把用户看成家庭成员，把数据看成一种商品（以食品为例吧）。

在家裡，孩子想喝饮料、吃水果等，可以直接去冰箱裡拿。妈妈煮饭炒菜煲汤，也去冰箱裡拿蔬菜、肉类等原材料。用户什麽时候想吃东西了，能够很方便的从冰箱裡直接取来食用，或进行加工后再食用。那麽，这个冰箱就像是应用资料库（或叫应用系统的资料库），冰箱裡的食品就是数据。应用资料库能够快速的解决业务用户日常的用数需求，想要直接拿就行了，快速便捷。

但是，冰箱本身存储的东西是有限的，如果冰箱裡的东西没了怎麽办？那麽当然是重新去超市採购喽，不过这个活不一定要所有的人来干，在家裡买菜这样的活一般还是由妈妈负责，这样看起来妈妈是管理员（当然同时也是用户），爸爸和孩子才是真正的业务用户啊！（笑）

好了，妈妈去超市买菜去了，这裡的超市就可以比做是数据集市。超市里分海鲜区、蔬果区、肉类区、饮料区等等，东西应有尽有，且按照客户的需要进行分门别类管理，便于大家进入相应的区域採购。那麽数据集市也是如此，按照业务应用需求，把数据进行分类，形成不同的主题区，比如风险主题集市存储了风险管理所需的数据，管会主题集市存储了管理会计所需的数据，监管主题集市存储了外部监管统计信息披露所需的数据等等，便于管理和使用。

妈妈买完东西后，再把食品存放到家裡的冰箱裡，以供所有家庭成员取用。同理，业务部门的管理员会定期从主题集市中获取所需的数据，并返给应用资料库支持日常用数，只不过往往这个动作不是人工来执行，而是定製好后通过系统接口自动批量处理。这样，数据集市就完成了对应用资料库的供数了！

噢，那麽大家自然而然就想到了，超市的货仓一定就是数据仓库啦！是的，这个货仓可以存储的东西非常多，虽然也是按照不同的商品分类进行管理（好比数据仓库的分区），但是用户一般不会直接到仓库里买东西，所以和用户来讲是相对隔离的。能接触仓库的就是库管员和送货员，这些人往往都是技术背景的，不属于业务用户的范畴。当然，事无绝对，不排除有些商家採取批发直销的模式，让用户直接去仓库里取货（国外有好多批发超市），不过这种情况毕竟较少了。所以，业务用户直接从数据仓库取数的情况也不是完全不存在的。

通过以上的生活实例，相信大家很容易能看出来数据集市与数据仓库、应用资料库之间的区别和联繫了。所以，数据集市对业务部门来说还是很有意义的，因为至少同时作为管理员和用户的妈妈还是需要到超市里採购东西的，而且不排除作为真正业务用户的爸爸和孩子也愿意去逛逛超市，直接选择自己想要的产品（笑）。大家想想，如果家裡附近没有超市，那是一件多麽不方便的事啊，这就是建立数据集市的意义所在！

最近业界还出现一个很新、很火的概念，叫数据工厂，很多人不理解它的含义和定位。没关係，还是拿上面的例子来解释。大家都知道，去超市买完鱼后回来得自己去鳞、取鳔等，一系列动作特别繁琐且不便，现在的年轻人一般都懒得做。好了，如果说有这样的数据工厂，能够按照用户的需求进行数据的定製加工，帮你把这一系列麻烦事做了，甚至还可以再进一步定製化的把鱼给清炖或红烧了，那麽用户就可以直接获得加工好的数据产品，何乐而不为？不过，这个加工厂如何选址、按什麽工序加工、如何配送那是科技部的事，用户可以不用管了，提出对最终产品的要求即可。

（六） 数据平台、大数据平台、元数据平台、数据服务平台

数据平台是一个很大的概念，但是这个概念在日常工作中往往被用的窄了。比如，很多业务用户认为数据平台就是报表平台、或者数据仓库，这和大家关注点、出发点不同是有很大关係的。

这麽说吧，数据和信息是对等的，平台和系统是对等的，那麽数据平台和信息系统也就是对等的。大家都知道，信息系统是多麽大的概念啊，那麽数据平台也是如此。有些金融机构说我们计划要建个数据平台，想问问专家有什麽建议。这就好比说我想建个信息系统，请你告诉应该怎麽做一样让人难以具体回答，因为还没说清楚要建什麽样的信息系统。所以，咱们首先还是要清楚数据平台到底包括哪些范围和类型。

一般来说，数据平台从大面上可以分为：数据整合平台、数据管理平台、以及数据应用平台。当然，这只是第一层级的划分，不过目前业界的数据相关平台都可以归到这三大类中。

数据整合平台指的是把数据从源系统进行收集和整合、加工处理、存储和共享的平台。数据整合平台的首要目标是形成企业级统一、共享的整合数据，为后续的数据应用提供基础。当然，整合平台也不仅仅是简单整合，还包括一定的数据加工。上面提到的数据集市、数据仓库、数据工厂都属于数据整合平台的范围。

那麽大家自然就会问，那麽大数据平台呢，是否属于数据整合平台的范围？严格意义上不完全是，就看这大数据平台是广义的还是狭义的了。应该这麽讲，广义的大数据平台和数据平台是在一个层级的，它也包括了大数据的整合平台、管理平台和应用平台。不过目前业界所说的大数据平台一般还是偏狭义的，即只包括大数据的整合平台。也就是说，狭义的大数据平台指的就是把海量的大数据进行整合和存储的平台。

区别于传统的数据整合平台，大数据平台架构由于其处理的数据类型和性质而与传统平台存在较大不同。在此前已经和大家聊过大数据的特点，那麽大数据平台比较特别的地方就在于它可以高效处理海量的历史数据、外部数据、特别是非结构化数据，一般是基于Hadoop分布式架构，具有高效性、高容错性和高扩展性的特点。而传统的数据整合平台（如数据仓库）多为关係型资料库，仅比较擅长处理结构化的数据。

再来说数据管理平台，数据管理平台的目标是实现数据全生命周期的管理，确保前面提到的数据治理、管理和管控措施的系统落地，保障数据质量。元数据平台就属于数据管理平台的范畴，通过元数据的管理最终实现对数据的管理。

数据管理平台的用户主要是数据管理部门和技术部门，当然也包括数据的业务用户。不过对于业务用户而言，数据管理平台最重要的意义主要体现在，一是可以查询并了解企业级的数据规范，包括数据的业务定义、统计规则等规范语言，二是在发生数据变化后，能够进行上下游的提示。例如，数据源发生变化后影响到后端的报表应用，报表用户能够及时获取变化信息，评估对自身用数的影响并制定应对措施。这个点其实非常重要，笔者在与很多业务部门同事沟通的过程中，许多人都提出了对此类上下游数据问题的不满，好比财务部的同事会抱怨前台部门修改数据前、后都没有及时和财务部进行沟通，最后报表都已经对外报送了，但是前台数据修改后就又导致业财不一致了。这类问题可通过数据管理平台解决。

最后再来说说数据应用平台，这也是业务用户最关心的、直接使用的用数平台。企业的报表平台、统计报送平台、指标系统、管理驾驶舱系统、决策支持平台、以及高阶的数据分析和挖掘工具等都属于数据应用平台的范畴。简单一句话概括，就是支持最终业务用户日常用数的应用交付平台。不论是数据整合平台、还是大数据平台，最终都是服务于业务用户的用数需求，虽然在应用模式上有所区别，有初阶的固定报表、数据模板应用，也有中阶的指标多维灵活查询、决策驾驶舱应用，还有高阶的大数据分析和挖掘应用，都需要通过数据应用平台进行实现数据服务的最终交付。

很多同事还会问，那麽数据服务平台又是什麽？和数据应用平台有何区别？以笔者个人的观点，数据服务和应用都属于一个范畴，即服务于用户的视角，实在没必要进行如此独立的平台区分，许多金融机构也是只建设统一的数据应用平台。如果非要区分两者间的区别的话，那麽笔者认为，数据应用更偏重于最终的数据结果交付，比如说用户通过应用介面获得想要的报表或指标；而数据服务更偏重于过程的调度，比如进行用户用数偏好分析以便选择用户可能需要的数据进行主动推送等。

（七） 基础数据、主数据

上面讲完有关係统和平台的两组概念后，接下来又要回到比较抽象的几组概念了，不过大家不用担心，笔者会儘量用简单的语言快速说明，大家别打瞌睡（笑）。

先来聊聊基础数据。基础数据是相对于衍生数据而言的，是从业务前端直接产生和採集的，未进行过加工计算的基础性数据。例如，业务交易产生的交易流水数据、採集的客户基础数据、签署的合约数据等，都属于基础数据的范畴。基础数据是企业开展日常业务经营所直接产生的，为中、后台的风险管理、财务核算、管理分析和统计应用等提供了数据基础。

基础数据的缺失和质量不高是目前业界碰到的最大难题。许多企业在开展数据统计和分析时，都或多或少遇到由于基础数据缺失而导致难以进行准确的分析和预测，无法更好的支持经营管理与决策，这个困难直接体现了企业的业务前台和中、后台管理之间的先天矛盾。

从前台的角度来讲，其主要的任务在于提高业绩、提升客户体验。基础数据的採集和维护一般都需要前台业务部门人员来执行，而这项工作却往往没有纳入其绩效考核的内容，因此往往引不起前台业务人员的重视。另一方面，在业务交易过程中进行数据的採集和维护也是需要一定的时间成本，如果在客户办理业务时过分强调信息的高质量採集，也会影响业务的办理效率和客户体验。而中、后台人员为了满足其管理目的，则希望前台能够儘可能多的採集所需的基础数据。看起来，这还真是一个难以调和的矛盾啊。笔者此前就曾遇到这样一种情况，企业的后台部门为了获取统计报送所需数据，提出了前台业务系统改造的数据需求，但是前台业务部门以影响业务效率为由拒绝接受，导致了相关工作难以顺利推进。相信许多中、后台管理人员都有类似的体会吧！

所以，为了更好的协调这个先天的矛盾，我们还是要站在前台业务的视角来解决问题，而不是一味的互相推卸和指责。首先要找到一个契合点，即哪些数据是前、中、后台部门，特别是前台部门关心的数据，这些数据又能如何解决前台部门的用数需求，那麽就能够求同存异，找到关键数据范围作为切入点，并以应用叠代的方式逐步完善基础数据的採集。例如，我们曾帮助企业建立客户交叉销售、客户维挽等数据分析模型以便更好的支持业务经营，通过这类应用项目，前台部门深刻意识到客户数据採集的重要性，并主动开展客户基础数据质量的提升工作。

在基础数据中，有一类数据非常重要，作为不同系统之间交互和共享的关键数据，我们把它称之为「主数据」。例如，不同业务系统都会有客户的数据、产品的数据，那麽如果每个部门、系统都分别维护这些数据，必然会造成不一致。因此，需要在企业级建立统一的主数据管理，如客户主数据、产品主数据、机构主数据等，在全公司范围内都应是统一且唯一的，并确定部门和系统间交互识别这些主数据的唯一标识，如客户编号、产品编号、机构编号等。那麽，这些唯一标识也成为了主数据中的主数据了，是开展主数据管理首先要统一、整合的对象。虽然这是项非常基础性的工作，但是许多企业却不见得做的好，例如有些金融机构的客户编号就没有在企业级范围进行统一，系统间的客户信息无法唯一识别，同一客户可能在机构内存在多条记录却并未整合等等。我们经常说，解决问题就要抓住问题的主要矛盾，那麽主数据问题就是所有数据问题中需要迫切解决的关键性问题了。

（八） 衍生数据、指标

基础数据和衍生数据是相对的两个概念，讲完基础数据，那自然就要说到衍生数据。

衍生数据是指按照一定的规则和逻辑，对已有的数据进行计算和加工后形成的数据。相对于基础数据可直接由前台部门採集而言，衍生数据一般是基于基础数据或其他衍生数据的再加工。

衍生数据常见的质量问题和基础数据不同，基础数据难在採集这个环节，而衍生数据却难在统一口径规范上。一般来说，金融机构的前、中、后台部门由于其视角和目的不同，对特定衍生数据会有不同的计算规则和口径，这也就导致了数据不一致问题。但是，这类问题解决起来相对基础数据还是简单一些，只要企业能够自上而下统一进行标准化的规范，明确衍生数据的唯一定义部门和加工系统，那麽很多问题都迎刃而解了。不过说起来容易做起来难，这裡面仍然需要协调各部门的不同利益和需求。

相对于基础数据中的主数据，衍生数据中也有一些非常关键的数据，用于满足管理层日常的经营管理和决策支持，我们把这些衍生数据称之为指标。

笔者在和业务人员交流的时候，其实很多人都不太清楚衍生数据的概念，但是只要提到指标，大家一下子就明白了，毕竟日常工作中都用到了各种指标。如果非要来区分一下衍生数据和指标的话，那麽可以这麽理解，衍生数据是一个广泛的技术概念，但是指标更偏重于业务应用，这也是为什麽许多金融机构近期致力于建设企业级指标体系的原因，因为只有指标才是管理层和业务部门关心的，衍生数据离业务太远了（笑）。

接下来就简单说明一下有关指标体系的几个基本概念。

（九） 报表统计项、指标、维度、度量

指标是反映对象特徵属性的、可衡量的单位或方法，是具有（业务）意义的指向和标杆。

指标可分为基础指标和衍生指标，其中基础指标是具备宽泛定义的统计对象，从而为指标的灵活组合与多维分析提供基础。衍生指标是在基础指标的基础上，通过添加一个或多个统计维度形成新的指标、或通过不同指标进行运算而形成新的指标。一般来说，指标都属于衍生数据。

维度（或称统计维度、筛选条件）是对指标进行描述的不同视角，用于标示指标的不同方面的属性，例如对贷款馀额这个指标，可以按照产品类型维度（如个人消费贷款、个人住房贷款等）来分析，可按照资金投放行业维度（如农、林、牧、渔等）来分析、可按照贷款期限维度（如长期、中期、短期等）来分析、也可以按照贷款状态维度（如正常、不良等）来分析等。一般来说，维度都属于基础数据。

度量是针对指标而言，和维度并没有关係，是指标的度量衡，本身并无实际统计意义，如金额、馀额、比率、笔数、个数等。例如，贷款馀额这个指标的度量就是馀额。而报表项、统计项等无非就是指标的应用了，企业所抽象出来的指标都是来自于各类报表的具体报表项或统计项，并回过头来应用于不同的报表。

有关具体如何建设企业级指标体系，笔者在这裡就不展开了，内容确实比较多，以后有机会可以专题讨论一下，很有意思的，特别是如何应用指标体系这块，能够有效解决企业的用数需求。

（十） 数据模型、数据分析模型、统计模型、数据建模

近期大数据分析特别火，所以很多朋友都在谈论数据分析和建模，这就引起了许多有关模型的概念了。

其实大家在谈的数据分析模型是一种统计模型（或者叫数学模型），是以数学统计的算法构建的複杂模型，满足于一定的应用场景。目前数据分析模型的应用场景较常用于两个方面，一是客户营销，二是风险管理。在风险管理领域，许多银行金融机构为了满足新资本协议的合规要求，建立了相应的风险计量模型，实际这也是属于统计模型的一种应用了。

数据分析模型一般需要输入大量的历史数据，按照既定的参数和算法进行模型计算后，输出对业务有意义的分析结果，例如有潜在产品需求的目标客户清单、特定价格策略下客户响应预测、存量贷款未来逾期的机率预测、最低资本要求等等。数据分析模型最终还是要服务于特定的业务经营和管理要求。

相对于数据分析模型，数据模型的含义可就不同了。数据模型指的是数据的结构和关係。最顶层的模型一般是概念模型，主要是从业务视角提出的对数据的概念性的需求表述。具体的数据模型一般会分为数据的逻辑模型和物理模型。逻辑模型指的是数据之间的逻辑关係，一般通过实体关係（ER）图来体现；而物理模型大家可以简单理解为资料库的表结构了。

数据分析模型更偏重于业务应用和决策支持，数据模型则更偏重于系统设计和实施，两者的目标是不同的。

解释完这几个模型的区别，那麽模型的构建（简称为「建模」）就很清晰了。业界说的数据建模一般指的还是数据模型的构建，但是描述不够准确，叫「数据模型建模」更加准确。而数据分析模型的构建则可以叫「分析模型建模」或「统计模型建模」。当然，在日常用语中，大家只说建模就可以了。

结束语：希望以上的概念解释和经验分享对朋友们理解相关术语有所帮助，许多观点也都是笔者个人的见解，不一定很准确，也算是抛砖引玉了，欢迎大家批评指正！

