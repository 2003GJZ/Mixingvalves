# 中间件职责

## 核心功能
该中间件的核心功能是接收和处理视频点赞的请求，通过冷热数据分离、采样判断、数据迁移以及异步备份，实现高并发、低延迟的点赞统计服务。其设计目的是提供一个稳定、独立的服务，使业务系统可以轻松调用和扩展。

## 热冷数据分离策略

### 热数据
- 使用 Redis 缓存，满足高频点赞请求，快速读取和更新数据，以确保高并发场景下的响应速度。

### 冷数据
- 使用硬盘文件存储冷数据，对于未达到高频访问的视频，直接将点赞信息记录在文件中。

### 冷热数据迁移
- 通过采样和触发机制检测点赞请求量，当冷数据视频的访问量达到一定阈值时，迁移至 Redis 缓存；当热数据视频的访问量下降，则回写至冷数据存储。

## 采样判断与迁移机制

### 采样检测
- 定期或按需触发，使用采样方法选取一部分冷数据文件并统计点赞量，通过访问频率判断是否需要迁移为热数据。

### 触发条件
- 若单个文件在一段时间内的点赞数超过设定的阈值，或者文件记录达到指定行数，则触发一次热度判断。随机取样方法的采样频率和采样量会根据系统压力自动调整。

### 触发迁移
- 当采样检测认为冷数据已达到热数据标准时，将对应视频的数据从硬盘文件迁移到 Redis。

## 异步数据备份

### 数据持久化
- 冷热数据均异步写入 MySQL，以保障数据的持久性。即便是 Redis 中的热数据在一段时间内也会备份至 MySQL，确保缓存中的点赞统计能够定期持久化。

### 备份方式
- 利用定时任务或队列，逐步将 Redis 缓存的热数据和冷数据文件的点赞记录批量写入 MySQL，实现备份和归档，方便后续分析或恢复。

## 模块划分及职责

### API 层
- 提供统一的 HTTP 接口，处理来自业务系统的点赞请求。

### 冷数据管理模块
- 负责将冷数据存储在文件系统中。每个视频对应一个文件，记录点赞的时间戳，支持大批量记录时的快速写入。

### 热数据缓存模块
- Redis 负责存储高频访问的点赞数，通过视频 ID 作为键进行存取。

### 采样判断模块
- 负责定期或按需采样，判断冷数据视频的热度变化，决定是否需要将冷数据迁移到 Redis。

### 迁移模块
- 根据采样结果，将达到热度条件的冷数据导入 Redis，或将长时间不活跃的热数据迁移回冷数据文件。

### 数据备份模块
- 定期或批量将热数据和冷数据的点赞统计同步到 MySQL，实现持久化存储。

## 流程概述

### 点赞请求处理
- 接收到点赞请求后，根据视频当前的热度状态决定将点赞数记录在 Redis 或冷数据文件中。

### 采样和迁移判断
- 采样模块定期或按需触发，通过采样计算冷数据的访问量，将符合条件的冷数据转移至 Redis，同时冷数据文件到达指定行数阈值时也会触发一次检查。

### 异步备份
- 定时将 Redis 热数据和冷数据文件中累计的点赞记录写入 MySQL，确保数据的完整和安全。