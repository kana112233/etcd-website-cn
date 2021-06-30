---
title: Issue triage guidelines
---

## Purpose 目标

Speed up issue management.  加快问题管理。

`etcd`问题列在https://github.com/etcd-io/etcd/issues和用标签标识。
The `etcd` issues are listed at https://github.com/etcd-io/etcd/issues
and are identified with labels. 
例如，已确定为bug的问题最终将被设置为标签`area/bug`.
For example, an issue that is identified
as a bug will eventually be set to label `area/bug `. 
新的issues在开始时没有任何标签，但是通常`etcd`维护者和活跃的贡献者根据他们的发现添加标签。
New issues will start out without any labels, but typically `etcd` maintainers and active contributors add labels based on their findings. 
标签的详细列表可以在https://github.com/kubernetes/kubernetes/labels
The detailed list of labels can be found at
https://github.com/kubernetes/kubernetes/labels

为方便起见，以下是一些预先搜索的的问题
Following are few predetermined searches on issues for convenience:
* [Bugs](https://github.com/etcd-io/etcd/labels/area%2Fbug)
* [Help Wanted](https://github.com/etcd-io/etcd/labels/Help%20Wanted)
* [Longest untriaged issues](https://github.com/etcd-io/etcd/issues?utf8=%E2%9C%93&q=is%3Aopen+sort%3Aupdated-asc+)

## Scope 范围
这些指南可作为对问题进行分类主要文档
These guidelines serves as a primary document for triaging an incoming issues in
`etcd`. 
欢迎每个人帮忙管理问题和PRs，但是本文档中讨论的工作和职责是在创建etcd维护者和积极贡献者的情况下创建的。
Everyone is welcome to help manage issues and PRs but the work and responsibilities discussed in this document are created with `etcd` maintainers and active contributors in mind.

## Validate if an issue is a bug 验证issue是否是一个bug

验证问题是否确实是一个bug。如果不是，请添加带有结果的评论并关闭不重要的issue。
Validate if the issue is indeed a bug. If not, add a comment with findings and close trivial issue. 
对于重要的问题，等待问题报告者的回应，看看是否有任何异议。
For non-trivial issue, wait to hear back from issue reporter and see if there is any objection. 
如果问题报告者没有在30天恢复，就关闭这个问题。
If issue reporter does not reply in 30 days, close the issue. 
如果问题不可以复现或者请求更多的信息，给问题报告者留下一个评论。
If the problem can not be reproduced or require more information, leave a comment for the issue reporter.

## Inactive issues 不活跃的问题

如果问题报告者未在60天内提供信息，则问题报告者缺乏足够信息的问题应关闭。
Issues that lack enough information from the issue reporter should be closed if issue reporter do not provide information in 60 days.

## Duplicate issues 重复的问题

如果是重复的问题，添加一个带有原始问题的引用评论并关闭它。
If an issue is a duplicate, add a comment stating so along with a reference for the original issue and close it.

## Issues that don't belong to etcd 不属于etcd的问题

有时报告的问题实际上属于使用`etcd`的其他项目。例如，`grpc` or `golang`的问题。
Sometime issues are reported that actually belongs to other projects that `etcd` use. For example, `grpc` or `golang` issues. 
此类问题应通过要求报告这在适当的其他项目中打开问题来解决。
Such issues should be addressed by asking reporter to open issues in appropriate other project. 
除非维护者和问题报告者认为需要保持打开状态以进行跟踪，否则关闭问题。
Close the issue unless a maintainer and issue reporter see a need to keep it open for tracking purpose.

## Verify important labels are in place 验证重要的标签是否正确到位

确保问题在它所属的区域上有标签，添加适当的受托人并确定里程碑。
Make sure that issue has label on areas it belongs to, proper assignees are added and milestone is identified. 
如果没有这些标签，就添加一个。如果由于权限有限而无法分配标签或无法确定正确的标签，那没关系，如果需要，请联系维护人员。
If any of these labels are missing, add one. If labels can not be assigned due to limited privilege or correct label can not be decided, that’s fine, contact maintainers if needed.

## Poke issue owner if needed 如果需要找出问题的拥有者

如果问题的所属人在30天没有创建一个PR，请联系问题所属者，求情PR或者如果需要，释放所有权。
If an issue owned by a developer has no PR created in 30 days, contact the issue owner and ask for a PR or to release ownership if needed.
