---
description: "## 秒杀  ### 主要问题  - 超卖  \t- 乐观锁  \t\t- update 库存表 set    已售数=已售数+1,版本号=版本号+1 where 秒杀id =#{id} and 版本号 = #{version}  \t\t- CAS（Compare And Swap）的ABA问题  \t\t\t- ABA问题解决就是没吃Swap后给数据变更版本号，如上  - 并发  \t- 限流  \t\t- 令牌桶"
---

# 分布式应用技术基础之归终

