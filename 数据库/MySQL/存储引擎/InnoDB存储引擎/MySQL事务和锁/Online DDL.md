---
title: Online DDL
date: 2023-02-19T22:33:28Z
lastmod: 2023-02-19T22:33:40Z
---

# Online DDL

## Online DDL的过程

1. 拿MDL写锁
2. 降级成MDL读锁
3. 真正做DDL
4. 升级成MDL写锁
5. 释放MDL锁
