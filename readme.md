# 系统架构
![alt text](./架构.png)

# 领域说明
## User Domain
### 用户域负责：

**账号注册 / 登录**

1. 创建新用户账号

2. 用户名 / 邮箱唯一性校验

3. 密码校验

4. 明文密码 → Hash 后存储

5. 登录时比对密码

**Token（JWT）签发**

为已验证身份的用户签发 JWT，包含基础 Claims：user_id、roles、exp 等

**用户状态管理**

1. 启用 / 禁用账号 _(被禁用的用户无法登录 / 创建订单)_

2. 权限（Role）信息提供（如需要）

3. 普通用户 / 管理员等角色

4. 提供角色信息给 BFF 或其他服务做权限控制

### 聚合根

用户是唯一聚合根，因为用户域内所有模型都依附于用户。

| 字段           | 类型       | 说明                             |
| ------------ | -------- | ------------------------------ |
| ID           | int      | 用户唯一标识（主键，自增或雪花）               |
| Email        | string   | 登录账号，需唯一                       |
| PasswordHash | string   | 加密后的密码（如 bcrypt）               |
| Nickname     | string   | 用户昵称                           |
| Status       | enum     | 用户状态：启用/禁用/封控中                     |
| Roles    | []string | 角色列表，例如：`["user"]`、`["admin"]` |
| CreatedAt    | time     | 创建时间                           |
| UpdatedAt    | time     | 更新时间                           |  
  
## Resource Domain  

### 资源域负责

1. **实例生命周期管理（创建 / 启停 / 删除）**

    1. 创建实例（Create Instance）
    2. 启停 / 重启实例（Start / Stop / Restart）
    3. 删除实例（Delete Instance）

2. **K8s 的交互与状态同步**

    1. 下发资源（Apply K8s Resources）
    2. 状态同步（Sync Status）

3. **域名 / 访问入口管理**

    1. 域名生成
    2. 消息转发

### 聚合根
Instance（用户实例） 是唯一聚合根，其他模型（规格 / 状态 / K8s 绑定信息）都附着于它。

| 字段                                | 类型     | 说明                               |
| --------------------------------- | ------ | -------------------------------- |
| InstanceID                        | int64  | 雪花变体主键 [UserID:24][TS:36][Seq:4] |
| UserID                            | int    | 用户 ID                            |
| Name                              | string | 实例名称                             |
| Status                            | enum   | 核心状态机字段                          |
| CreatedAt / UpdatedAt / DeletedAt | time   | 生命周期字段                           |

### 子实体

1. InstanceSpec（规格实体）

| 字段         | 类型     | 说明    |
| ---------- | ------ | ----- |
| InstanceID | int64  | 对应聚合根 |
| CPU        | int    | 核心数   |
| Memory     | int    | 内存    |
| GPU        | int    | 可为null, int 代表不同类型  |
| Image      | string | 镜像    |
| ConfigJSON | json   | 扩展配置  |

2. InstanceK8s（运行态实体，高频更新，保存redis最近一次的快照）

| 字段             | 类型     | 说明             |
| -------------- | ------ | -------------- |
| InstanceID     | int64  | 聚合根            |
| Namespace      | string | NS             |
| DeploymentName | string | K8s Deployment |
| ServiceName    | string | K8s Service    |
| IngressName    | string | K8s Ingress    |
| LastK8sState   | string | Pod Phase      |
| LastK8sMessage | string | 错误信息           |
| ErrorReason    | string | 最新错误原因         |
| UpdatedAt      | time   | 最新活动时间         |

3. InstanceNetwork（用户私有入口相关）

| 字段          | 类型     |
| ----------- | ------ |
| InstanceID  | int64  |
| EndpointURL | string |
| Domain      | string |
| Port        | int    |
| CreatedAt   | time   |
| UpdatedAt   | time   |

4. InstanceLogs（领域日志表）

| 字段         | 说明                                         |
| ---------- | ------------------------------------------ |
| ID         | 自增主键                                       |
| InstanceID | 关联实例                                       |
| LogType    | 日志类型：INFO / ERROR / WARNING / STATE_CHANGE |
| Message    | 日志内容（文本）                                   |
| DataJSON   | 可选（结构化信息）                                  |
| CreatedAt  | 日志时间                                       |

### Redis缓存模型状态

**字段说明**

| 字段             | 说明                                            |
| -------------- | --------------------------------------------- |
| state          | 资源域用状态（Running / Failed / Creating）           |
| phase          | Pod Phase（Running / Error / CrashLoopBackOff） |
| ready_replicas | Pod 就绪数                                       |
| total_replicas | Deployment 副本数                                |
| pod_name       | 当前对应 Pod                                      |
| message        | 最近一次事件消息                                      |
| error_reason   | K8s 错误原因                                      |
| updated_at     | 更新时间戳（秒级）                                     |

``` redis
HSET instance:k8s:{instance_id} \
    state "Running" \
    phase "Running" \
    ready_replicas "1" \
    total_replicas "1" \
    pod_name "llm-7846fdfb7f-k82xn" \
    message "Container started successfully" \
    error_reason "" \
    updated_at "1739504500"
```

### 消息队列

1. 实例事件 (instance.events)

```json
{
  "event_type": "INSTANCE_CREATED",
  "instance_id": 912345678901,
  "timestamp": 1739525000,
  "user_id": 88,
  "name": "my-llm-instance",
  "data": {
    "cpu": 2,
    "memory": 4096,
    "gpu": 0,
    "image": "registry.xxx/llm:v1"
  }
}
```

事件信息
| 行为                     | 事件类型                     |
| ---------------------- | ------------------------ |
| 创建实例                   | INSTANCE_CREATED         |
| 删除实例                   | INSTANCE_DELETED         |
| 规格变更（CPU/Memory/Image） | INSTANCE_SPEC_CHANGED    |
| 镜像删除                   | INSTANCE_IMAGE_REMOVED   |
| 镜像更新                   | INSTANCE_IMAGE_UPDATED   |
| 启动实例                   | INSTANCE_STARTED         |
| 停止实例                   | INSTANCE_STOPPED         |
| 实例状态变化（Running/Failed） | INSTANCE_STATUS_CHANGED  |
| K8s 状态回传               | INSTANCE_K8S_SYNC        |
| 域名/网络更新                | INSTANCE_NETWORK_UPDATED |


## Product Domain

### 商品域负责

负责整个业务链路中“可售卖内容”与“交易行为”的核心逻辑

### 聚合根  

| 字段          | 类型           | 说明                 |
| ----------- | ------------ | ------------------ |
| ProductID   | int          | 主键                 |
| Name        | string       | 名称                 |
| Description | string       | 描述                 |
| Status      | enum         | ENABLED / DISABLED |
| CreatedAt   | time         | 创建                 |
| UpdatedAt   | time         | 更新                 |
| Spec        | ProductSpec  | 实例配置值对象                |
| Price       | uint32 | 价格                |

### 子实体

ProductSpec 实例配置
| 字段         | 类型     | 说明   |
| ---------- | ------ | ---- |
| CPU        | int    |  单位c    |
| Memory     | int    |  单位h    |
| GPU        | int    |  可为null, int 代表不同类型   |
| Image      | string |  镜像来源    |
| ConfigJSON | json   | 扩展字段 |

Order 订单信息（聚合根）

| 字段         | 类型     | 说明                                           |
| ---------- | ------ | -------------------------------------------- |
| OrderID    | int64  | 订单唯一标识（主键，雪花ID）                              |
| UserID     | int    | 用户ID（关联User Domain）                          |
| ProductID  | int    | 商品ID（关联Product）                              |
| InstanceID | int64  | 实例ID（订单创建成功后关联的实例，关联Resource Domain）         |
| Status     | enum   | 订单状态：PENDING / PAID / CANCELLED / COMPLETED |
| Amount     | uint32 | 订单金额（分）                                      |
| CreatedAt  | time   | 创建时间                                         |
| UpdatedAt  | time   | 更新时间                                         |
| PaidAt     | time   | 支付时间（可为null）                                 |
| CancelledAt | time  | 取消时间（可为null）                                 |
| CompletedAt | time  | 完成时间（可为null）                                 |