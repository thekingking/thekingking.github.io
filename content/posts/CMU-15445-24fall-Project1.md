---
title: "CMU 15445 24fall Project1"
date: 2025-03-07T18:09:40+08:00
lastMod: 2025-03-21T18:09:40+08:00
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

## 23fall解决思路

我先讲讲我原本的并发加锁思路，这里所写的是从磁盘中获取页面，即读取不在内存中的页面。

![24fall-Project1-img1](/images/24fall-Project1-img1.png)

23fall中的思路为对所有线性执行（不包含并发操作的部分）加bpm锁，每次调用bpm都需要获取bpm锁，对frame的操作（有IO操作）根据frame_id加锁，保证每个frame_id的操作能够独立执行，不受影响。同时先获取bpm锁，然后获取frame锁，然后释放bpm锁，最后释放frame锁，保证了顺序执行，不会被其他线程插队。但是这样写其实还存在问题，只是在23fall中的测试没有测出来，在24fall中遇到了，也是折腾了我好久（以为我的思路是正确的）。下面展示下原因：

![24fall-Project1-img2](/images/24fall-Project1-img2.png)

## 错误的尝试

并发失败的原因就是不能够保证page的顺序执行，有两种解决办法：

1. 在线程1写入页面B后释放bpm锁，这样就能够保证page读写的顺序执行了，但是并发度会大幅度下降，显然不是我们所需要的
2. 添加一个page_mutexes_的锁，对每一个page加锁，但是由于page太多，我根据leaderboard中page数设置page_mutexes数组大小为6400，并且使用page_mutexes[page_id % 6400]获取page锁。page_mutexes锁获取放在获取frames_mutexes锁之前，保证获取frame锁之前拥有page的锁，并且一次获取所有需要操作的page的锁。

![24fall-Project1-img2](/images/24fall-Project1-img3.png)

改了之后可以正常运行，但是不能够先获取frame锁，再获取page锁，这样你甚至无法通过本地测试（哈哈），具体哪个我忘了，有兴趣可以试试。具体原因是获取frame锁后获取page锁失败，导致一直持有bpm锁和frame锁，导致其他线程无法执行，因为释放pageGuard是需要获取bpm锁和frame锁的。

这个方法能够通过p1的测试，但是在p2的测试中存在问题，所有有了下面的更好的解决办法

## 最终大招：引入条件变量

既然出现问题是因为写入和读取在在并发时不能够保证顺序执行，那么我就引入一个条件变量，在从内存中读取页面A之前判断是否有无页面A的脏页面没有写入或正在写入，具体流程如下：

在bufferPoolManager添加属性如下：

```cpp
    /** @brief A set of dirty pages that need to be flushed to disk. */
    std::unordered_set<page_id_t> dirty_pages_;
    /** @brief A mutex to protect the dirty pages set. */
    std::mutex flush_mutex_;
    /** @brief A condition variable to notify the flusher thread that there are dirty pages to flush. */
    std::condition_variable flush_cv_;
```

操作流程如下：

1. FetchPage获取新页面中，在释放bpm_latch锁前，判断原本的frame是否是脏页，如果是脏页，将其写入dirty_pages_中，表明这个page_id对应的page是脏页并且没有写入
2. 在释放bpm_latch锁后，进入脏页写入，成功写入脏页后将对应page_id从dirty_pages_中删除，表明对应page_id的脏页不存在了，同时使用notify_all唤醒等待的线程
3. 在读取页面之前，使用flush_wait判断是否有脏页未写入，如果有就陷入等待，释放flush_mutex_锁（不释放frame的锁）

下面是最终优化结果，后续应该不会再写bustub了，b+树也已经写完了，后面部分没啥必要再写了，和23fall差不多。

![24fall-Project1-LeaderBoard-new](/images/24fall-Project1-LeaderBoard-new.png)

## 写在最后

并发还是博大精深，需要学习的太多了。总的来说，24fall和23fall变化并不是很多，虽然多了FrameHeader部分和修改了PageGuard部分，但是总体还是差不多的，主要是并发部分的错误折腾我太久了，先入为主的认为原本的设计是正确的了（沉默）。
