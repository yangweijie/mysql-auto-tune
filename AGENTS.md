# MySQL Auto-Tune 项目上下文

## 项目概述

**项目名称**: mysql-proxy  
**许可证**: MIT License  
**开发语言**: Go 1.25.4  

这是一个基于 Go 语言实现的 **MySQL 数据库智能代理服务**，作为客户端应用与 MySQL 服务器之间的透明中间层。核心功能包括：

- **SQL 解析与分析**: 使用 TiDB parser 解析 SQL 语句，识别操作类型和涉及的表
- **读写分离**: 根据操作类型自动路由到主库或从库
- **Binlog 监听**: 实时订阅 MySQL binlog，用于缓存失效和数据同步
- **连接池管理**: 高效管理到上游 MySQL 的数据库连接

## 架构设计

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────┐
│   客户端      │ --> │   Go Proxy       │ --> │ MySQL Master  │
│  (Laravel等)  │     │   (监听 3306)     │     │   (写操作)     │
└──────────────┘     └──────────────────┘     └───────────────┘
                              │                        │
                              │                        │
                              │ (Binlog 同步)          │
                              v                        │
                      ┌───────────────┐                │
                      │  Binlog 监听器 │                │
                      │  (缓存失效)    │                │
                      └───────────────┘                │
                                                       v
                                              ┌───────────────┐
                                              │  MySQL Slave  │
                                              │   (读操作)     │
                                              └───────────────┘
```

## 技术栈

| 组件 | 用途 | 依赖包 |
|------|------|--------|
| **MySQL 协议代理** | 接收客户端连接，解析 MySQL 协议包 | `github.com/go-mysql-org/go-mysql/server` |
| **SQL 解析** | 解析 SQL 语句，构建 AST | `github.com/pingcap/tidb/pkg/parser` |
| **MySQL 驱动** | 连接上游 MySQL 数据库 | `github.com/go-sql-driver/mysql` |
| **Binlog 同步** | 模拟从库订阅 binlog 事件 | `github.com/go-mysql-org/go-mysql/replication` |

## 核心配置

项目中的关键配置常量（位于 `main.go`）：

```go
const (
    listenAddr      = ":3306"           // 代理监听端口
    masterAddr      = "127.0.0.1:3307" // MySQL 主库地址
    slaveAddr       = "127.0.0.1:3308" // MySQL 从库地址
    replicationUser = "replicator"      // 复制用户名
    replicationPass = "password"        // 复制密码
    serverID        = 1001              // Binlog 同步 Server ID
)
```

**注意**: 实际部署时需要根据环境修改这些配置。

## 构建与运行

### 构建项目

```bash
# 构建二进制文件
go build -o mysql-proxy main.go

# 或直接运行
go run main.go
```

### 运行要求

1. **MySQL 主库**: 需要开启 binlog（ROW 模式）
2. **MySQL 从库**: 用于读操作分流
3. **复制账户**: 需要具有 REPLICATION SLAVE 权限

### 数据库准备

```sql
-- 创建复制用户
CREATE USER 'replicator'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
```

## 核心模块说明

### 1. ProxyHandler

处理 MySQL 协议命令，核心方法：

- `HandleQuery(query string)`: 接收 SQL 查询，解析并路由执行
- 支持结果集转换为 MySQL 协议格式
- 记录查询日志和性能指标

### 2. isWriteOperation(node ast.StmtNode)

判断 SQL 操作类型：

- **写操作**: INSERT, UPDATE, DELETE → 路由到主库
- **读操作**: SELECT 等 → 路由到从库

### 3. startBinlogListener()

Binlog 监听器：

- 模拟 MySQL 从库连接到主库
- 实时接收数据变更事件
- 支持事件类型：DELETE_ROWS, UPDATE_ROWS, WRITE_ROWS, QUERY, XID

### 4. handleConnection(conn net.Conn, s *server.Server)

连接处理：

- 为每个客户端连接创建独立的 goroutine
- 处理认证和命令执行
- 支持优雅关闭

## 数据流详解

1. **客户端连接** → 代理监听 3306 端口
2. **SQL 解析** → 使用 TiDB parser 构建 AST
3. **操作分类** → 判断读/写操作
4. **路由决策** → 选择主库或从库
5. **执行查询** → 通过连接池执行 SQL
6. **结果转换** → 将结果集转为 MySQL 协议格式
7. **返回客户端** → 响应查询结果

## 待实现功能

根据代码中的 TODO 标记和 `tech.md` 文档，以下功能待开发：

### P0 - 核心功能
- [ ] **缓存模块**: 实现 SELECT 结果缓存（Redis/内存 LRU）
- [ ] **缓存失效逻辑**: 基于 binlog 事件的自动缓存清除
- [ ] **错误处理优化**: 更完善的错误处理和重试机制

### P1 - 增强功能
- [ ] **查询重写引擎**: 根据 AST 修改 SQL 语句
- [ ] **配置管理**: 支持配置文件和热加载
- [ ] **连接池优化**: 动态调整连接池大小

### P2 - 监控与管理
- [ ] **Prometheus 指标**: 暴露性能指标
- [ ] **管理 API**: 提供配置查询和统计接口
- [ ] **查询分析器**: 慢查询分析和索引建议

## 开发规范

### 代码风格

- 遵循 Go 官方代码规范
- 使用 `gofmt` 格式化代码
- 函数命名使用驼峰式
- 错误处理不应忽略

### 日志规范

```go
// 查询日志
log.Printf("Received query: %s", query)

// 错误日志
log.Printf("Failed to parse SQL: %v", err)

// 事件日志
log.Printf("Received binlog event: %v", ev.Header.EventType)
```

### 测试要求

- 新增功能需编写单元测试
- 测试覆盖率目标: ≥ 80%
- 使用 `go test` 运行测试

```bash
# 运行测试
go test -v ./...

# 测试覆盖率
go test -cover ./...
```

## 常见问题

### 1. Binlog 连接失败

**原因**: 主库未开启 binlog 或复制用户权限不足

**解决**: 
```sql
-- 检查 binlog 配置
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';

-- 授予权限
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
```

### 2. SQL 解析失败

**原因**: 不支持的 SQL 语法或复杂查询

**解决**: 查看日志中的解析错误，考虑添加异常处理或跳过解析

### 3. 连接池耗尽

**原因**: 高并发下连接数不足

**解决**: 调整 `SetMaxOpenConns` 和 `SetConnMaxLifetime` 参数

## 性能优化建议

1. **连接复用**: 增大连接池大小，减少连接建立开销
2. **SQL 解析缓存**: 对相同 SQL 的解析结果进行缓存
3. **结果集缓存**: 实现查询结果缓存，减少数据库负载
4. **并发处理**: 每个连接独立 goroutine，充分利用多核
5. **协议优化**: 考虑使用预处理语句减少解析开销

## 相关文档

- [tech.md](./tech.md): 详细的技术设计文档
- [go-mysql 文档](https://github.com/go-mysql-org/go-mysql)
- [TiDB Parser 文档](https://github.com/pingcap/tidb/tree/master/pkg/parser)

## 快速开始示例

```bash
# 1. 克隆项目
git clone <repository-url>
cd mysql-auto-tune

# 2. 安装依赖
go mod download

# 3. 修改配置（编辑 main.go 中的常量）

# 4. 构建
go build -o mysql-proxy

# 5. 运行
./mysql-proxy

# 6. 连接测试
mysql -h 127.0.0.1 -P 3306 -u proxy -p
```

## 贡献指南

1. Fork 项目
2. 创建功能分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 创建 Pull Request

---

**最后更新**: 2026-03-17  
**维护者**: jay
