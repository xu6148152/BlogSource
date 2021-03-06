---
title: ReactNative-CSSLayout
tags: React-Native
---

主要两个核心类：  

 * ``CSSNode``用于存储``View``节点的信息
 * ``LayoutEngine``用于布局``View``

``canUseCachedMeasurement``是否使用缓存的测量结果

``layoutNodeInternal``内部布局的方法

``layoutNodeImpl``布局的实现

布局的步骤：

 1. 为算法计算值(CALCULATE VALUES FOR REMAINDER OF ALGORITHM)
 2. 决定可用的大小(DETERMINE AVAILABLE SIZE IN MAIN AND CROSS DIRECTIONS)
 3. 决定每个节点弹性基准(DETERMINE FLEX BASIS FOR EACH ITEM)
 4. COLLECT FLEX ITEMS INTO FLEX LINES(COLLECT FLEX ITEMS INTO FLEX LINES)
 5. 解析主基准弹性条目(RESOLVING FLEXIBLE LENGTHS ON MAIN AXIS)
 6. 调整主基准(MAIN-AXIS JUSTIFICATION & CROSS-AXIS SIZE DETERMINATION)
 7. 共用基准对齐(CROSS-AXIS ALIGNMENT)
 8. 多行内容对齐(MULTI-LINE CONTENT ALIGNMENT)
 9. 计算最终尺寸(COMPUTING FINAL DIMENSIONS)
 10. 循环设置子节点位置(SETTING TRAILING POSITIONS FOR CHILDREN)
 11. 定位和定大小绝对节点(SIZING AND POSITIONING ABSOLUTE CHILDREN)
