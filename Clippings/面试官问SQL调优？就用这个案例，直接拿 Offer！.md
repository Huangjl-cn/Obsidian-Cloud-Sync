---
title: "面试官问SQL调优？就用这个案例，直接拿 Offer！"
source: "https://www.bilibili.com/video/BV1qQUUBQE44?spm_id_from=333.788.recommend_more_video.1&trackid=web_related_0.router-related-2479604-gjmc5.1787578673026.322&vd_source=6f2fb11b60bcc48d85309f56fa59e87f"
author:
  - "[[面试鸭]]"
published: 2025-11-25
created: 2026-08-24
description: "分享一个 vivo 慢 SQL 优化实战。程序员面试刷题工具【面试鸭】：mianshiya点com"
tags:
  - "clippings"
---
<iframe width="560" height="315" src="https://player.bilibili.com/player.html?bvid=BV1qQUUBQE44&amp;page=1&amp;high_quality=1&amp;danmaku=0" title="Bilibili video player" frameborder="0" allowfullscreen=""></iframe>

分享一个 vivo 慢 SQL 优化实战。  
  
程序员面试刷题工具【面试鸭】：mianshiya点com

## Transcript

**0:00** · 现在程序员去面试都会被问一句你有线上收购调教经验吗很多人当场心虚八股文看过但我真的没上线调过呀别换今天我把VIVO献上了一个真实慢售后优化实战喂到你嘴里那听完这一分钟这就是你的亲身经历发车你说你当时负责优化系统从现场监控看到了一条慢SQL 咨询时间竟然是分钟级而且这条售后看起来平平无奇左表连右表插点字段那最后按T1的时间倒序排一下那数据也不大那T1才3000行 T2几万行暂停两秒凭直觉猜猜这种小表交易怎么可能跑到分钟级呢

**0:30** · 直接上EXSPLAY 那两张表都全表扫描但以这个函数来看全表扫描根本漫不到分钟级那真正问题藏在extra里面一开始我以为是file shot或者template man的当时就用optimized trace来验证临时表示memory类型不落磁盘内存排序只涉及3000行毫秒级搞定所以排序不是瓶颈那问题也就出现在了BNL身上那他的逻辑是这样的读一段左表数据进内存那全表扫描右表逐含匹配再读下一段左表那再全表扫描右表一次重复反复持续那右表几万函

**0:59** · 他就这样被扫描了几十次上百次开销直接呈指数级膨胀所以慢到分钟一点都不奇怪那之所以这样是因为右表关键字段没索引导致只能走BNL解决办法非常简单那在右表的T2的join字段A上加一个索引加完之后 MYSQL的算法瞬间变成了index next to the loop join 你右表可以按索引快速定位不在全表扫B2L直接被淘汰那重点来了面试时怎么把这个案例变成你的呢直接背下面这套那我之前遇到一个线上慢斯后来个join数据不大却跑了几十秒我先看explain

**1:29** · 出现了template的fl shot和BNL 但我没着急下结论我用OPTIMMY的trace验证了排序在内存中非常怪不是瓶颈最终定位到右表关联字段无索引导致走了BNL右表被反复全表扫描我给关联字段加了索引 join变成了NLJ 查询从分钟级优化到了20ms 性能提升了3000倍搞定那推荐收藏多看几遍你就是一位有线上收购调优经验的软件工程师