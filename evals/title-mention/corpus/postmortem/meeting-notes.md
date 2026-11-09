---
title: "会议记录"
type: note
---

参会人：Alice Example、Jack Example、Carol Example、Karen Example。议程：10 月发布版本的问题与后续步骤。Jack 总结了订单服务在 schema 迁移删除一列后无法回滚的情况，以及从快照恢复耗时 70 分钟。Carol 主张迁移应在健康检查后作为独立的受控阶段运行。Alice 询问流水线中的回滚路径是否演练过；答案是没有。决议：所有服务采用扩展-收缩迁移，在发布检查清单中加入预发演练，并在流水线中阻止同一发布版本内删除列。Karen 提到故障期间的告警噪音使确认延迟了六分钟。行动项：Jack 周五前起草迁移指南，Carol 更新流水线阶段，Alice 安排下次发布版本的演练。下次会议：两周后，Harbor Street 会议室。