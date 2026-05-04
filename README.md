# payment-service
privacy
Payment Service - 支付服务 L3 详细设计
==================================================
> **对应简版**：：L3_PaymentService_brief_v1.0.md
> **对应用例**：UC-004-user-create-pay-order-v1.0.md - {用户创建支付订单}
---

## 1. 元信息

- **系统**：通用业务平台
- **容器名称**：Payment Service
- **版本**：v1.0
- **最后更新**：2026-05-04
- **状态**：Draft
- **作者 / Owner**：杨永钰
- **运行时环境**：Python + FastAPI
- **上游文档**：
*L2：L2-Container-v1.0.md
*L3 简版：L3_PaymentService_brief_v1.0.md
- **相关契约 / Schema**：
 * OpenAPI：OpenAPI-level1-template.yaml
---

## 2. 设计目标与约束
### 2.1 设计目标

- **统一支付订单生命周期管理**
聚合订单创建、支付状态流转、回调处理、结果查询，提供统一支付入口。

- **对接第三方支付渠道**
封装渠道差异，向上游提供标准化支付接口。

- **业务逻辑与数据访问解耦**
支付逻辑内聚、数据访问统一封装，便于测试、扩展与替换渠道。
幂等、安全、可追溯
保证支付不重复、不丢单、不篡改，全流程可审计。

### 2.2 非目标

- **不负责账户 / 余额管理**
账户、余额、积分由账户服务管理。

- **不负责退款审核与人工处理**
仅提供退款执行接口，审核流程由上层系统控制。

- **不存储用户核心信息**
仅保留支付必需的最小用户标识，不维护用户资料。

- **不做支付渠道的缓存**
不处理通知推送
支付结果通知由消息服务统一发送。

---

## 3. 内部架构总览

### 3.1 组件视图
时序图：<img width="4558" height="1057" alt="image" src="https://github.com/user-attachments/assets/416990f6-532c-44d9-a421-a7427bebf8b1" />
:::mermaid
flowchart TD
subgraph PS ["Payment Service（内部模块）"]
API ["API 层（FastAPI Router）"]
subgraph SRV ["服务层"]
PG ["PayGenerator（订单生成器）"]
CB ["CallbackHandler（回调处理器）"]
VALID ["Validator（校验器）"]
IDEMP ["Idempotent（幂等器）"]
end
DAL ["DataAccessLayer（数据访问层）"]
subgraph INFRA ["基础设施层"]
PGDB ["PostgreSQL Client"]
RD ["Redis Client"]
CH ["Channel Handler（渠道适配器）"]
end
end
API --> VALID
VALID --> IDEMP
IDEMP --> PG
PG --> DAL
PG --> CH
CB --> DAL
DAL --> PGDB
DAL --> RD
PG --> API
CB --> API
:::
  
---

**说明：**

- API 层：请求路由、参数解析、响应包装、异常统一捕获。
- 服务层：
  - PayGenerator：创建支付单、调用渠道、组装支付信息。
  - CallbackHandler：处理渠道异步回调、更新状态。
  - Validator：金额、商品、权限、状态合法性校验。
  - Idempotent：防重、幂等校验，避免重复下单。
- 数据访问层：统一封装支付单、流水、日志的读写。
- 基础设施层：数据库、缓存、第三方渠道客户端。

### 3.2 进程模型
- 单进程 + 异步 I/O 模型：FastAPI + async/await
  - 无状态、可水平扩展
  - 无长耗时后台任务
- 关键配置：
  - 数据库连接池：20
  - 请求超时：30s
  - 回调超时：10s
  - Worker 由 K8s 控制

## 4. 内部数据模型（Informative）

### 4.1 核心实体
PayOrder（支付订单）
```python
class PayOrder:
    id: UUID
    order_no: str               # 商户订单号（唯一）
    user_id: str                # 用户ID
    total_amount: float         # 订单总金额
    status: str                 # PENDING / PAID / FAILED / CLOSED
    channel: str                 # 支付渠道 ALIPAY / WXPAY
    created_at: datetime
    paid_at: datetime | None
```

PayChannel（支付渠道）
```python
class PayChannelRecord:
    id: UUID
    order_no: str                # 内部订单号
    channel_order_no: str        # 外部渠道订单号
    channel_status: str          # 渠道状态
    paid_amount: float
    raw_data: dict               # 渠道原始回调数据
```

PayCallbackLog（回调日志）
```python
class PayCallbackLog:
    id: UUID
    order_no: str
    channel: str
    success: bool
    raw_body: str
    created_at: datetime
```

PayResult（支付结果）
```python
class PayResult:
    order_no: str
    status: str
    paid_amount: float
    paid_at: datetime | None
    channel: str
```

###4.2 报告数据结构
```python
class PayCreateRequest:
    user_id: str
    total_amount: float
    channel: str  # ALIPAY / WXPAY
    subject: str

class PayCreateResponse:
    order_no: str
    pay_url: str
    status: str
    total_amount: float

class PayQueryResponse:
    order_no: str
    user_id: str
    total_amount: float
    status: str
    paid_amount: float
    paid_at: datetime | None

class PayCallbackResult:
    success: bool
    return_msg: str

PayOrderDTO

PayCreateRequest

PayCallbackRequest

PayQueryResponse
```
##5. 内部组件设计
###5.1 API Handler
- 职责：
  - 接收 HTTP 请求，解析参数
  - JWT 鉴权
  - 调用 PaymentGenerator 创建 / 查询订单
  - 调用 CallbackHandler 处理渠道回调
  - 统一异常处理、标准化响应
  - 日志与指标埋点
  - 
  - 关键接口
``` python
async def create_pay_order(req: PayCreateRequest) -> PayCreateResponse:
    """创建支付订单"""

async def get_pay_order(order_no: str) -> PayQueryResponse:
    """查询支付订单状态"""

async def pay_callback(channel: str, body: dict) -> dict:
    """支付渠道异步回调"""
```
###5.2 PayGenerator
- 职责：
  - 编排支付订单创建流程
  - 调用 Validator 做业务校验
  - 调用数据访问层落库
  - 调用渠道获取支付链接
  - 组装返回结果
 
核心流程
```python 
class PaymentGenerator:
    async def create_order(self, req: PayCreateRequest) -> PayCreateResponse:
        # 1. 校验金额、渠道、权限
        await self.validator.validate_create_request(req)
        
        # 2. 幂等校验：同一用户同一金额不可重复下单
        await self.idempotent.check(req.user_id, req.total_amount)
        
        # 3. 创建支付订单
        order = await self.dal.create_pay_order(req)
        
        # 4. 调用支付渠道
        channel_resp = await self.channel_client.create_pay(order)
        
        # 5. 返回支付信息
        return PayCreateResponse(
            order_no=order.order_no,
            pay_url=channel_resp.pay_url,
            status=order.status,
            total_amount=order.total_amount
        )
```

###5.3 Validator
- 职责：
  - 纯校验逻辑，无 I/O
  - 金额、状态、幂等、权限校验

关键算法
```python
class Validator:
    async def validate_create_request(self, req: PayCreateRequest):
        if req.total_amount <= 0:
            raise HTTPException(400, "金额必须大于0")
        
        if req.channel not in ["ALIPAY", "WXPAY"]:
            raise HTTPException(400, "不支持的支付渠道")

    async def validate_order_available(self, order: PayOrder):
        if order.status != "PENDING":
            raise HTTPException(409, "订单已支付或已关闭", code="ORDER_NOT_PENDING")
```

###5.4 CallbackHandler
- 职责：
  - 处理渠道回调
  - 验签
  - 更新订单状态
  - 记录日志
  - 保证幂等

```python
class CallbackHandler:
    async def handle(self, channel: str, body: dict):
        # 1. 验签
        await self.channel_verifier.verify(channel, body)
        
        # 2. 获取订单号
        order_no = body["out_trade_no"]
        
        # 3. 幂等去重
        if await self.dal.is_callback_processed(order_no):
            return {"success": True}
        
        # 4. 更新订单为已支付
        await self.dal.update_order_to_paid(order_no, body)
        
        # 5. 记录回调日志
        await self.dal.save_callback_log(order_no, channel, True)
        
        return {"success": True}
```
###5.5 数据访问层 DAL
```python
class DataAccessLayer:
    async def create_pay_order(self, req: PayCreateRequest) -> PayOrder:
        # 事务创建订单
        pass

    async def get_order_by_no(self, order_no: str) -> PayOrder:
        pass

    async def update_order_to_paid(self, order_no: str, raw_data: dict):
        # 事务更新状态 + 写入渠道流水
        pass

    async def is_callback_processed(self, order_no: str) -> bool:
        pass
```
---

##6. 关键流程详解
###6.1 创建支付订单流程
时序图：<img width="2982" height="1412" alt="image" src="https://github.com/user-attachments/assets/bb8b4505-7858-467c-bafd-3114fe10dd83" />
:::mermaid
  sequenceDiagram
    participant C as Client
    participant A as API Handler
    participant V as Validator
    participant I as Idempotent
    participant PG as PayGenerator
    participant DAL as DataAccessLayer
    participant DB as PostgreSQL
    participant CH as Channel
    C->>A: POST /pay/order/create
    A->>V: 校验参数、金额
    V->>I: 幂等校验
    I->>DAL: 查询订单是否已存在
    DAL->>DB: 返回订单
    I-->>PG: 允许创建
    PG->>DAL: 创建待支付订单
    DAL->>DB: INSERT order
    PG->>CH: 请求支付渠道
    CH-->>PG: 返回支付信息
    PG-->>A: 返回支付参数
    A-->>C: 200 OK
:::
关键步骤：
1.校验与幂等（~5ms）
2.创建待支付订单（~20–50ms）
3.请求渠道（~100–300ms）
4.返回支付信息

###6.2 支付回调流程
   时序图：<img width="2214" height="1356" alt="image" src="https://github.com/user-attachments/assets/d6918c67-4e2e-420e-929d-d2060db3157a" />
1.接收渠道回调
2.验签
3.幂等去重
4.更新订单状态为成功 / 失败
5.记录回调日志

###6.3 错误处理流程
   6.3.1 订单已支付
   ```python
   async def validate_order_available(self, order_no: str):
    order = await self.dal.get_order_by_no(order_no)
    if not order:
        raise HTTPException(404, "订单不存在", code="ORDER_NOT_FOUND")
    
    if order.status != "PENDING":
        raise HTTPException(
            status_code=409,
            detail="订单已支付或已关闭",
            code="ORDER_NOT_PENDING"
        )
```

   6.3.2 金额非法
```python
   async def validate_amount(amount: float):
    if amount <= 0:
        raise HTTPException(
            status_code=400,
            detail="支付金额必须大于0",
            code="INVALID_AMOUNT"
        )
```

##7. 可靠性、容错与配置

###7.1 关键配置项

环境变量	         默认值	         说明
REQUEST_TIMEOUT	  30s	         接口超时
DB_POOL_SIZE	    20	          连接池
CHANNEL_TIMEOUT	  10s	       渠道调用超时
IDEMP_TTL	        10m	      幂等锁过期时间
 
###7.2 容错策略 

数据库异常：重试 3 次，快速失败
渠道超时：熔断，避免阻塞
回调重复：直接幂等丢弃
状态不一致：支持补偿查询
防超卖：金额、库存统一校验
##8. Observability：Metrics & 日志

日志规范
订单创建：order_no、amount、channel
回调：验签结果、状态、来源 IP
异常：渠道错误、堆栈、上下文
性能告警：>500ms 打警告

##9. 测试要点
正常创建支付单
重复创建 → 幂等拦截
金额非法 → 拒绝
重复回调 → 丢弃
渠道超时 → 友好降级
金额非法 → 拒绝
重复回调 → 丢弃
渠道超时 → 友好降级
