# Design Review组织架构和流程

## 一 为什么需要Design Review

Fluid本身的开发由社区驱动，一方面需要加强社区方向的引导和梳理，一方面需要社区成员加强协同争取一次把事情做对。随着软件不断迭代，生产环境部署越来越多，我们偿还技术债务的成本会越来越高。在过去的研发过程中，我们经常会遇到如下问题：

- 需求在碎片化的讨论中由于未得到明确，导致设计偏离了要解决的问题；
- 因缺乏设计文档，关联合作方未能及时review并发现设计中的问题；
- 方案本身缺乏比较导致技术债或者设计缺陷;
- 过一段时间后关于为什么这样设计已经很难追踪;

从小的方面来说以上这些问题可能影响的是研发工作效率。而从更大的层面来说，我们要提高系统的工程质量，构建领先的开源系统，需要具备严谨的工作方式，而这些需要我们在系统设计层面提高设计质量。写好Design Doc从技术层面有如下意义：

- 帮助我们清晰地定义要解决的问题，列出解决方案以及背后的思考路径；
- 通过清楚的整体设计描述，帮助开发团队理解整体架构和思路，提升开发效率；
- 关联合作方可以及时的获取高质量的项目架构和设计信息并提供及时反馈；

另外从个人成长来看，撰写高质量结构化的设计文档是职业发展甚至晋升的必要素质之一。
​
## 二 Design Review Committee
#### Steering Committee
成员：Toc成员
职责：对整 Design review模版/流程有最终解释权
​
#### Review Committee
成员：Committee， maintainer， Toc成员
职责：对分配到的review request进行评审，提出问题/修改意见。最终signoff满意的版本。


| 步骤 | 执行人员 | 备注 |
| --- | --- | --- |
| 1、按照模板编写设计文档 | 研发人员（doc owner） | 文档模板：
[design doc template](template.md) |
| 2、确定Reviewers
Reviewer要求：
- 至少有一个Toc成员
- 最低signoff人数是3人。小于3人需要Toc批准。signoff人数不设上限。
- Toc有权要求增加reviewer。| Doc owner+Toc | 除特殊情况，doc owner不能是reviewer。 |
| 3 Design review
1）Doc owner在钉钉群“Fluid研发**Design Review**工作群”中@具体的reviewer，附上doc link。
2）Online doc review：Reviewer确保在**7天内完成**。
3）Meeting评审：如果online doc review无法达成共识，则doc owner发起会议评审，前提是各reviwer已经把意见反馈到doc文档的comment区域。
4）Signoff：每一个reviewer都需要最终表态这个设计方案是否通过/打回。 | Reviewers | 如果能够通过在线review完成，则不需要开会；如果需要会议review，具体形式由doc owner和reviewers商议决定。 |

## FAQ

问：什么粒度的方案需要发 Design Review？  
答：以下两类情况必须发 Design Review：  
1）对现有API（无论是对外的还是对内的）进行修改或新增API（比如新CRD）；
2）Feature级别改动；

问：Doc owner对最终设计方案负责，但是如果方案有问题reviewers是否也需要承担相应的一定的责任？  
答：是。让reviewer参与signoff，就是一种共同担责的模式。
​

问：Reviewer是否有权利拒绝一个Tech Review的邀请？  
答：可以拒绝。doc owner和TL与reviewer沟通好即可。


问：如果reviewer因为其他原因无法在7天内完成review，该怎么办？  
答：reviewer需要向doc owner说明原因，申请延期多久完成或者建议换reviewer。

问：如果临近7天截止期了，仍然有大部分人没有参与review也没有signoff，那应该怎么办？  
答：doc owner是整个事情的driver，他需要线下推动大家完成review工作和signoff，哪怕亲自上门去提醒。

问：如果reviewer没有全部完成signoff，是否可以开工？  
答：review没有完成，并非完全不能开工。我们有时会邀请比较多的人参与review，解决reviewer的所有comments可能耗时比较长，如果整个开发工作block在那里，效率也不高。因此doc owner需要判断在一轮review后，是否收到的反馈里面有关键的、可能导致方案产生大的修改的问题要解决，如果没有这样问题，可以并行投入开发，但是需要承担一定风险（后续review过程中提到的方案修改可能导致返工）。 另外对于reviewer来说，也需要尽量把自己的反馈写的更清楚，并建议分类：如果是阻塞项目开发的意见，**建议用P1来标示醒目，表示这个问题不解决，作为评审不同意该方案继续进行**。












