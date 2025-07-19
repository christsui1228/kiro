# Design Document

## Overview

尺码排序模块是一个独立的功能模块，集成到OneManage系统的features目录下。该模块提供尺码数据的排序和统计功能，使用Polars进行高效数据处理，通过FastAPI提供RESTful API接口，支持前端表格输入和结果导出。

## Architecture

### 模块结构
```
OneManage/features/size_sorting/
├── __init__.py
├── models.py          # 数据模型定义
├── schemas.py         # Pydantic模式定义
├── routers.py         # FastAPI路由定义
├── services.py        # 业务逻辑服务
└── utils.py           # 工具函数
```

### 技术栈集成
- **FastAPI**: 与现有系统保持一致的API框架
- **Polars**: 高性能数据处理库，替代pandas进行数据操作
- **Pydantic**: 数据验证和序列化
- **SQLModel**: 如需持久化，与现有数据库模式保持一致
- **现有认证系统**: 集成现有的用户认证机制

## Components and Interfaces

### 1. 数据模型 (models.py)

#### SizeOrder枚举
```python
from enum import Enum

class SizeOrder(str, Enum):
    S = "S"
    M = "M" 
    L = "L"
    XL = "XL"
    XXL = "XXL"
    XXXL = "XXXL"
    XXXXL = "XXXXL"
    XXXXXL = "XXXXXL"
    XXXXXXL = "XXXXXXL"
    XXXXXXXL = "XXXXXXXL"
```

#### 可选的持久化模型
如果需要保存处理历史，可以定义：
```python
class SizeSortingRecord(SQLModel, table=True):
    __tablename__ = "size_sorting_records"
    
    record_id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    user_id: uuid.UUID = Field(foreign_key="users.user_id")
    input_data: str  # JSON格式存储原始数据
    sorted_data: str  # JSON格式存储排序结果
    statistics: str   # JSON格式存储统计信息
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

### 2. 数据模式 (schemas.py)

#### 输入模式
```python
class SizeDataItem(BaseModel):
    size: str
    description: str

class SizeSortingRequest(BaseModel):
    data: List[SizeDataItem]
    
class SizeSortingBatchRequest(BaseModel):
    data: List[List[str]]  # 支持二维数组输入
```

#### 输出模式
```python
class SortedSizeItem(BaseModel):
    size: str
    description: str
    sort_order: int

class SizeStatistics(BaseModel):
    size: str
    count: int
    percentage: float

class SizeSortingResponse(BaseModel):
    sorted_data: List[SortedSizeItem]
    statistics: List[SizeStatistics]
    total_count: int
    processing_time_ms: float
```

### 3. 业务服务 (services.py)

#### SizeSortingService类
```python
class SizeSortingService:
    def __init__(self):
        self.size_order_map = {size.value: idx for idx, size in enumerate(SizeOrder)}
    
    async def sort_sizes(self, data: List[SizeDataItem]) -> SizeSortingResponse:
        # 使用Polars进行数据处理
        
    async def process_batch_data(self, data: List[List[str]]) -> SizeSortingResponse:
        # 批量处理表格数据
        
    def _create_polars_dataframe(self, data) -> pl.DataFrame:
        # 创建Polars DataFrame
        
    def _sort_by_size_order(self, df: pl.DataFrame) -> pl.DataFrame:
        # 按尺码顺序排序
        
    def _calculate_statistics(self, df: pl.DataFrame) -> List[SizeStatistics]:
        # 计算统计信息
```

### 4. API路由 (routers.py)

#### 主要端点
```python
@router.post("/size-sorting/sort", response_model=SizeSortingResponse)
async def sort_sizes(
    request: SizeSortingRequest,
    current_user: User = Depends(get_current_user)
):
    # 排序处理

@router.post("/size-sorting/batch", response_model=SizeSortingResponse) 
async def sort_sizes_batch(
    request: SizeSortingBatchRequest,
    current_user: User = Depends(get_current_user)
):
    # 批量处理

@router.get("/size-sorting/export/{session_id}")
async def export_results(
    session_id: str,
    format: str = "csv",
    current_user: User = Depends(get_current_user)
):
    # 导出功能
```

## Data Models

### 核心数据流

1. **输入数据结构**
   ```json
   {
     "data": [
       {"size": "XL", "description": "大码T恤"},
       {"size": "M", "description": "中码T恤"},
       {"size": "S", "description": "小码T恤"}
     ]
   }
   ```

2. **Polars DataFrame处理**
   - 创建包含size、description、sort_order列的DataFrame
   - 使用自定义排序逻辑对size列进行排序
   - 计算统计信息

3. **输出数据结构**
   ```json
   {
     "sorted_data": [
       {"size": "S", "description": "小码T恤", "sort_order": 0},
       {"size": "M", "description": "中码T恤", "sort_order": 1},
       {"size": "XL", "description": "大码T恤", "sort_order": 3}
     ],
     "statistics": [
       {"size": "S", "count": 1, "percentage": 33.33},
       {"size": "M", "count": 1, "percentage": 33.33},
       {"size": "XL", "count": 1, "percentage": 33.33}
     ],
     "total_count": 3,
     "processing_time_ms": 15.2
   }
   ```

### Polars数据处理策略

1. **自定义排序函数**
   ```python
   def get_size_sort_key(size: str) -> int:
       return size_order_map.get(size, 999)  # 未知尺码排在最后
   ```

2. **高效统计计算**
   ```python
   df.group_by("size").agg([
       pl.count().alias("count"),
       (pl.count() / pl.len() * 100).alias("percentage")
   ])
   ```

## Error Handling

### 错误类型定义
```python
class SizeSortingError(Exception):
    pass

class InvalidSizeError(SizeSortingError):
    pass

class DataProcessingError(SizeSortingError):
    pass
```

### 错误处理策略

1. **输入验证错误**
   - 返回400状态码
   - 提供详细的字段验证错误信息

2. **数据处理错误**
   - 返回422状态码
   - 记录错误日志
   - 提供用户友好的错误信息

3. **系统错误**
   - 返回500状态码
   - 记录详细错误日志
   - 返回通用错误信息

4. **性能限制**
   - 数据量超过限制时返回413状态码
   - 处理时间超时返回408状态码

## Testing Strategy

### 单元测试

1. **服务层测试**
   ```python
   # test_services.py
   async def test_sort_sizes_basic():
       # 测试基本排序功能
       
   async def test_sort_sizes_with_unknown():
       # 测试包含未知尺码的排序
       
   async def test_statistics_calculation():
       # 测试统计计算
   ```

2. **工具函数测试**
   ```python
   # test_utils.py
   def test_size_order_mapping():
       # 测试尺码顺序映射
       
   def test_polars_dataframe_creation():
       # 测试DataFrame创建
   ```

### 集成测试

1. **API端点测试**
   ```python
   # test_routers.py
   async def test_sort_endpoint():
       # 测试排序API端点
       
   async def test_batch_endpoint():
       # 测试批量处理端点
       
   async def test_export_endpoint():
       # 测试导出端点
   ```

2. **认证集成测试**
   ```python
   async def test_authenticated_access():
       # 测试认证用户访问
       
   async def test_unauthenticated_access():
       # 测试未认证用户访问
   ```

### 性能测试

1. **大数据集测试**
   - 测试1000、10000、100000条记录的处理性能
   - 验证内存使用情况
   - 确保响应时间在可接受范围内

2. **并发测试**
   - 测试多用户同时访问的性能
   - 验证系统稳定性

### 测试数据

```python
# 测试用例数据
TEST_DATA_BASIC = [
    {"size": "XL", "description": "大码"},
    {"size": "S", "description": "小码"},
    {"size": "M", "description": "中码"}
]

TEST_DATA_WITH_UNKNOWN = [
    {"size": "XL", "description": "大码"},
    {"size": "UNKNOWN", "description": "未知尺码"},
    {"size": "S", "description": "小码"}
]

TEST_DATA_LARGE = [
    # 生成大量测试数据用于性能测试
]
```

## Integration Points

### 与现有系统集成

1. **认证系统集成**
   - 使用现有的`get_current_user`依赖
   - 遵循现有的权限控制模式

2. **数据库集成**
   - 如需持久化，使用现有的数据库连接
   - 遵循现有的迁移管理模式

3. **路由集成**
   - 在主应用中注册size_sorting路由
   - 使用统一的API前缀

4. **日志集成**
   - 使用现有的Loguru日志配置
   - 遵循现有的日志格式和级别

### 前端集成点

1. **API接口**
   - 提供OpenAPI文档
   - 支持CORS配置

2. **数据格式**
   - 支持JSON格式的数据交换
   - 提供清晰的错误响应格式

3. **导出功能**
   - 支持CSV格式导出
   - 提供下载链接或直接返回文件内容