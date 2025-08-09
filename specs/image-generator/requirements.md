# 需求文档

## 介绍

图片生成器模块是OneManage系统中的一个新功能模块，用于根据文本信息批量生成透明背景的PNG图片。该模块使用Pillow库进行图像处理，支持自定义字体、像素大小等参数，并提供灵活的数据源支持（CSV文件和数据库）。模块采用分阶段开发方式，最终集成到消息队列系统中。

## 需求

### 需求 1

**用户故事：** 作为系统管理员，我希望能够批量生成带有文本内容的透明背景PNG图片，以便用于各种展示和标识用途。

#### 验收标准

1. WHEN 提供文本信息列表 THEN 系统 SHALL 为每个文本信息生成一张独立的PNG图片
2. WHEN 生成图片 THEN 系统 SHALL 确保图片背景为透明（24位PNG格式）
3. WHEN 处理N个文本信息 THEN 系统 SHALL 输出N张对应的图片文件
4. WHEN 生成完成 THEN 系统 SHALL 提供生成结果的状态反馈

### 需求 2

**用户故事：** 作为用户，我希望能够自定义图片生成的参数（字体、大小、样式等），以便满足不同场景的视觉需求。

#### 验收标准

1. WHEN 指定字体文件 THEN 系统 SHALL 使用该字体渲染文本
2. WHEN 设置字体大小 THEN 系统 SHALL 按照指定像素大小渲染文本
3. WHEN 配置输出参数 THEN 系统 SHALL 支持调节图片尺寸、文本颜色等属性
4. IF 未指定字体 THEN 系统 SHALL 使用默认系统字体
5. WHEN 参数无效 THEN 系统 SHALL 提供清晰的错误提示

### 需求 3

**用户故事：** 作为开发人员，我希望系统支持从CSV文件读取数据进行测试，以便验证图片生成功能的正确性。

#### 验收标准

1. WHEN 提供CSV文件路径 THEN 系统 SHALL 读取文件中的文本内容
2. WHEN CSV文件包含多列 THEN 系统 SHALL 支持指定文本内容所在的列
3. WHEN CSV文件格式错误 THEN 系统 SHALL 提供详细的错误信息
4. WHEN 处理CSV数据 THEN 系统 SHALL 逐行迭代处理文本内容
5. IF CSV文件为空 THEN 系统 SHALL 给出相应提示

### 需求 4

**用户故事：** 作为系统集成者，我希望系统能够从数据库读取数据生成图片，以便与现有的数据管理流程集成。

#### 验收标准

1. WHEN 连接数据库 THEN 系统 SHALL 支持常见的数据库类型（MySQL、PostgreSQL等）
2. WHEN 执行查询 THEN 系统 SHALL 根据指定的SQL查询获取文本数据
3. WHEN 处理数据库结果 THEN 系统 SHALL 支持指定文本字段名称
4. WHEN 数据量较大 THEN 系统 SHALL 支持分批处理以避免内存溢出
5. IF 数据库连接失败 THEN 系统 SHALL 提供连接错误的详细信息

### 需求 5

**用户故事：** 作为架构师，我希望图片生成功能能够集成到taskiq + NATS消息队列系统中，以便支持异步处理和系统解耦。

#### 验收标准

1. WHEN 接收taskiq + NATS发布的任务消息 THEN 系统 SHALL 通过订阅监听模式解析任务中的生成参数
2. WHEN 处理taskiq任务 THEN 系统 SHALL 异步执行图片生成操作
3. WHEN 生成完成 THEN 系统 SHALL 通过NATS发布结果消息
4. WHEN 处理失败 THEN 系统 SHALL 支持taskiq的重试机制并发布错误消息
5. WHEN 集成到现有消息队列 THEN 系统 SHALL 严格按照当前项目的taskiq + NATS发布订阅监听模式
6. IF NATS服务不可用 THEN 系统 SHALL 提供连接错误信息反馈