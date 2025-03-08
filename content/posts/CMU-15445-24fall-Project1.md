---
title: "CMU 15445 24fall Project1"
date: 2025-03-07T18:09:40+08:00
lastMod: 2025-03-07T18:09:40+08:00
draft: false # 是否为草稿
author:  ["tkk"]

categories:  ["CMU15445 24fall", C++]

tags:  [database, C++, Project]

keywords:  ["CMU15445 24fall", bustub, Project1]

description:  "CMU 15445 24fall Project1 实现及优化" # 文章描述, 与搜索优化相关
summary:  "CMU 15445 24fall Project1 实现及优化" # 文章简单描述, 会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: false
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

## 前言

开学后便开始启动24fall的bustub了，做这一部分主要是因为23fall缺少了B+树的部分，所以来做24fall来学习一下B+树的实现。先说说23到24的变化，变化主要有以下3点：

1. 首先是23的Page到24变成了FrameHeader并放到了buffer_pool_manager文件中，也就是说在24中需要自己实现底层的Page需要自己实现了，也是自由度更高了，但是我的实现也没有太大变化
2. 其次便是删除了BasicPageGuard，只保留ReadGuard和WriteGuard，并且保证从BufferManager中读取的页面只能够为ReadGuard或WriteGuard，即读取的页面必须加了读锁或写锁，这样也能够实现我23fall中说的将加锁和解锁部分放在ReadGuard和WriteGuard（好耶）
3. 最后也是最重要的，修改了NewPage和FetchPage的分工，这里的设计比23fall合理多了。在24fall中NewPage只获取页面page_id，将所有的页面读取放在了ReadPage和WritePage中，这样就可以只实现一个FetchPage，然后分别在ReadPage和WritePage中构建ReadGuard和WriteGuard，这样获取页面就需要写一个逻辑了（舒服了）。
4. 补充一下，在lru_k中函数有小部分修改，Evict返回optional值了，而不是原先通过指针和bool变量返回两个值，更合理的设计。磁盘读写没注意不知道改过没，我还是用的23fall的实现。

总的来说，24fall设计得比23fall更好了，需要实现的部分也稍微变多了，大体实现还是不变的，最重要的是自由度更高了（嗯，我的观点），并发感觉便难了，因为page的读写需要自己实现。在24fall中也发现23fall中的错误，当然在23fall中的测试是测不出来的，后面在讲。大体优化还是和23fall一样，并发稍微修改了下，LeaderBoard排名第6（2025/03/07），打开就能看见了（哈哈），下面是优化结果。

![24fall-Project1-LeaderBoard](/images/24fall-Project1-LeaderBoard.png)

这里就不细讲实现了，我只写下我遇到的问题，大体实现和23fall差不多

## 并发

我先讲讲我原本的并发加锁思路，这里所写的是从磁盘中获取页面，即读取不在内存中的页面。

![24fall-Project1-img1](/images/24fall-Project1-img1.png)

23fall中的思路为对所有线性执行（不包含并发操作的部分）加bpm锁，每次调用bpm都需要获取bpm锁，对frame的操作（有IO操作）根据frame_id加锁，保证每个frame_id的操作能够独立执行，不受影响。同时先获取bpm锁，然后获取frame锁，然后释放bpm锁，最后释放frame锁，保证了顺序执行，不会被其他线程插队。但是这样写其实还存在问题，只是在23fall中的测试没有测出来，在24fall中遇到了，也是折腾了我好久（以为我的思路是正确的）。下面展示下原因：

![24fall-Project1-img2](/images/24fall-Project1-img2.png)

并发失败的原因就是不能够保证page的顺序执行，有两种解决办法：

1. 在线程1写入页面B后释放bpm锁，这样就能够保证page读写的顺序执行了，但是并发度会大幅度下降，显然不是我们所需要的
2. 添加一个page_mutexes_的锁，对每一个page加锁，但是由于page太多，我根据leaderboard中page数设置page_mutexes数组大小为6400，并且使用page_mutexes[page_id % 6400]获取page锁。page_mutexes锁获取放在获取frames_mutexes锁之前，保证获取frame锁之前拥有page的锁，并且一次获取所有需要操作的page的锁。

![24fall-Project1-img2](/images/24fall-Project1-img3.png)

改了之后可以正常运行，但是不能够先获取frame锁，再获取page锁，这样你甚至无法通过本地测试（哈哈），具体哪个我忘了，有兴趣可以试试。具体原因是获取frame锁后获取page锁失败，导致一直持有bpm锁和frame锁，导致其他线程无法执行，因为释放pageGuard是需要获取bpm锁和frame锁的。

## 写在最后

并发还是博大精深，需要学习的太多了。总的来说，24fall和23fall变化并不是很多，虽然多了FrameHeader部分和修改了PageGuard部分，但是总体还是差不多的，主要是并发部分的错误折腾我太久了，先入为主的认为原本的设计是正确的了（沉默）。
