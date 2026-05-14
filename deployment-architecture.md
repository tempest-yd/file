# Search Agent 系统部署架构

## 整体部署拓扑架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         客户端层 (Client Layer)                       │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │  Web Client  │  │ Mobile App   │  │ Third-party  │                │
│  │              │  │              │  │  Services    │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
│         │                  │                  │                       │
└─────────┼──────────────────┼──────────────────┼──────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API Gateway & Load Balancer                        │
│          (Authentication, Rate Limiting, Request Routing)             │
│                                                                       │
│        ┌─────────────────────────────────────────────────────┐        │
│        │  Nginx / Kong / API Gateway (HA with keepalived)    │        │
│        └──────────────────────┬──────────────────────────────┘        │
└──────────���──────────────────────────────────┬──────────────────────────┘
                                              │
                ┌─────────────────────────────┼──────────────────────────┐
                │                             │                          │
                ▼                             ▼                          ▼
┌──────────────────────────────┐ ┌──────────────────────────────┐ ┌──────┐
│  Kubernetes Cluster (Zone A) │ │  Kubernetes Cluster (Zone B) │ │Config │
│                              │ │                              │ │Center│
│ ┌────────────────────────┐   │ │ ┌────────────────────────┐   │ │(Lion)│
│ │ MCP Service (Pod)      │   │ │ │ MCP Service (Pod)      │   │ │      │
│ │ ├─ Request Handler     │   │ │ │ ├─ Request Handler     │   │ │      │
│ │ ├─ Auth & Quota        │   │ │ │ ├─ Auth & Quota        │   │ │      │
│ │ └─ Task Dispatcher     │   │ │ │ └─ Task Dispatcher     │   │ │      │
│ └────────────────────────┘   │ │ └────────────────────────┘   │ └──────┘
│                              │ │                              │
│ ┌────────────────────────┐   │ │ ┌────────────────────────┐   │
│ │ Search Agent (Pod)     │   │ │ │ Search Agent (Pod)     │   │
│ │ ├─ Router Engine       │   │ │ │ ├─ Router Engine       │   │
│ │ ├─ Strategy Engine     │   │ │ │ ├─ Strategy Engine     │   │
│ │ ├─ Task Manager        │   │ │ │ ├─ Task Manager        │   │
│ │ └─ Monitoring          │   │ │ │ └─ Monitoring          │   │
│ └────────────────────────┘   │ │ └────────────────────────┘   │
│                              │ │                              │
│ ┌────────────────────────┐   │ │ ┌────────────────────────┐   │
│ │ Prompt Parser (Pod)    │   │ │ │ Prompt Parser (Pod)    │   │
│ │ ├─ LLM Inference Pool  │   │ │ │ ├─ LLM Inference Pool  │   │
│ │ ├─ NER & Segmentation  │   │ │ │ ├─ NER & Segmentation  │   │
│ │ └─ Intent Detection    │   │ │ │ └─ Intent Detection    │   │
│ └────────────────────────┘   │ │ └────────────────────────┘   │
│                              │ │                              │
│ ┌────────────────────────────────────────────────────────┐   │
│ │ Ingress Controller (Traffic Management)               │   │
│ └────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬────────────────────────────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
                ▼              ▼              ▼
        ┌──────────────┐ ┌────────────┐ ┌──────────────┐
        │  Real-time   │ │  Crawler   │ │  Offline ETL │
        │  Search      │ │  Pool      │ │  Pipeline    │
        │  Backend     │ │            │ │              │
        │ (bin search) │ │ ┌────────┐ │ │ ┌──────────┐ │
        │              │ │ │Download│ │ │ │Batch     │ │
        │              │ │ │Layer   │ │ │ │Collector │ │
        │              │ │ ├────────┤ │ │ └──────────┘ │
        │              │ │ │Render  │ │ │              │
        │              │ │ │Layer   │ │ │ ┌──────────┐ │
        │              │ │ ├────────┤ │ │ │Cleaner   │ │
        │              │ │ │Parse   │ │ │ └──────────┘ │
        │              │ │ │Layer   │ │ │              │
        │              │ │ └────────┘ │ │ ┌──────────┐ │
        │              │ │            │ │ │Vectorizer│ │
        │              │ │ ┌────────┐ │ │ └──────────┘ │
        │              │ │ │Scheduler│ │ │              │
        │              │ │ │& Queue │ │ │ ┌──────────┐ │
        │              │ │ └────────┘ │ │ │Indexer   │ │
        │              │ │            │ │ └──────────┘ │
        │              │ │ ┌────────┐ │ │              │
        │              │ │ │Dedup & │ │ │              │
        │              │ │ │Rate-   │ │ │              │
        │              │ │ │Limit   │ │ │              │
        │              │ │ └────────┘ │ │              │
        │              │ │            │ │              │
        │              │ │ Resource   │ │ Auto-scaling │
        │              │ │ Pool       │ │ & HA config  │
        │              │ │ (16 cores) │ │              │
        │              │ │ (32GB RAM) │ │ (GPU support)│
        └──────────────┘ └────────────┘ └──────────────┘
                │              │              │
                ▼              ▼              ▼
        ┌────────────────────────────────────────────────┐
        │     Message Queue (Mafka - Streaming Layer)    │
        │                                                │
        │  Real-time Parse Queue │ Processing Queue     │
        │  Data Push Topic       │ Storage Topic         │
        │  Index Update Topic    │ Quality Assessment Q  │
        └──────────────┬─────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┬──────────────┐
        │              │              │              │
        ▼              ▼              ▼              ▼
   ┌─────────┐  ┌────────────┐ ┌──────────┐  ┌──────────────┐
   │Database │  │Vector Index│ │Inverted  │  │Object Storage│
   │(Meta)   │  │(Faiss/     │ │Index     │  │(S3/MinIO)    │
   │         │  │Milvus/ES)  │ │(ES/      │  │              │
   │┌───────┐│  │            │ │Lucene)   │  │Raw Content   │
   ││User   ││  │┌──────────┐│ │          │  │Intermediate  │
   ││Queries││  ││Vector DB ││ │┌────────┐│  │Snapshots     │
   │├───────┤│  ││cluster   ││ ││Text idx││  │              │
   ││Search ││  ││replicas  ││ ││cluster ││  │Version &     │
   ││Results││  │└──────────┘│ │└────────┘│  │Lineage Info  │
   │├───────┤│  │            │ │          │  │              │
   ││Crawled││  │┌──────────┐│ │Segment  │  │Audit Logs    │
   ││Data   ││  ││Real-time ││ │Merge    │  │              │
   │├───────┤│  ││index     ││ │Strategy │  │              │
   ││Quality││  │└──────────┘│ │          │  │              │
   ││Score  ││  │            │ │Alias    │  │              │
   │├───────┤│  │Sharding:   │ │Routing  │  │              │
   ││Index  ││  │Hash-based  │ │         │  │              │
   ││Meta   ││  │             │ │         │  │              │
   │├───────┤│  │Replication:│ │         │  │              │
   ││Task   ││  │3 copies    │ │         │  │              │
   ││Status ││  │            │ │         │  │              │
   │└───────┘│  │Recovery    │ │         │  │              │
   │         │  │& Failover  │ │         │  │              │
   │Backup   │  │            │ │         │  │              │
   │(Daily)  │  │            │ │         │  │              │
   └─────────┘  └────────────┘ └─────────┘  └──────────────┘
        │              │              │           │
        └──────────────┴──────────────┴───────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
   ┌──────────────────┐        ┌──────────────────┐
   │ Monitoring Stack │        │  Observability   │
   │                  │        │  & Tracing       │
   │┌────────────────┐│        │                  │
   ││Prometheus +    ││        │┌────────────────┐│
   ││Grafana         ││        ││Jaeger/Zipkin  ││
   │├────────────────┤│        ││Distributed     ││
   ││ELK Stack       ││        ││Tracing         ││
   ││(Logs)          ││        │├────────────────┤│
   │├────────────────┤│        ││ELK/Loki        ││
   ││Alertmanager    ││        ││Logs Centralize ││
   ││SLA/SLO         ││        │├────────────────┤│
   │├────────────────┤│        ││Auditd + SIEM   ││
   ││Incident Mgmt   ││        ││Security Events ││
   ││(On-call)       ││        │└────────────────┘│
   │└────────────────┘│        │                  │
   │                  │        │Dashboard Display │
   │Real-time KPIs:   │        │& Alerting        │
   │- P50/P95/P99     │        │                  │
   │- QPS/Throughput  │        │                  │
   │- Error Rate      │        │                  │
   │- Resource Usage  │        │                  │
   └──────────────────┘        └──────────────────┘
```

## 关键部署特性

### 高可用性设计
- **多可用区部署**: 跨 Zone A/B 的 K8s 集群
- **服务副本**: MCP、Search Agent、Prompt Parser 均运行多个副本
- **负载均衡**: 基于延��和健康状态的智能路由
- **故障转移**: 自动转移到健康实例，支持快速恢复

### 弹性扩缩容
- **水平扩展**: 爬虫池、消息队列消费者支持自动扩缩容
- **资源隔离**: 不同优先级请求使用独立资源池
- **优雅降级**: 高负载下自动降低采样率和实时功能

### 数据一致性
- **对象存储**: 作为单一真实源（Source of Truth）
- **三副本策略**: 向量索引和倒排索引各维护 3 份副本
- **版本管理**: 所有索引版本、构建参数、checksum 完整记录
- **增量更新**: Delta 索引 + 定期合并策略

### 监控和可观测性
- **分布式追踪**: Jaeger/Zipkin 跟踪请求全生命周期
- **指标采集**: Prometheus 实时采集系统性能指标
- **日志集中化**: ELK/Loki 集中管理应用和系统日志
- **安全审计**: Auditd + SIEM 记录所有敏感操作

### 发布和变更管理
- **金丝雀发布**: 通过 MCM 平台支持灰度发布（5%→10%→50%→100%）
- **快速回滚**: 保留前一版本快照，支持秒级回滚
- **配置灰度**: Lion 配置中心支持按实例/分组灰度下发
- **自动化测试**: PR 自动触发单元测试和性能回归检查

## 部署命令示例

```bash
# 部署到 Kubernetes
kubectl apply -f mcp-service.yaml
kubectl apply -f search-agent-service.yaml
kubectl apply -f prompt-parser-service.yaml
kubectl apply -f crawler-pool.yaml
kubectl apply -f data-processing.yaml

# 配置自动扩缩容
kubectl autoscale deployment search-agent --min=3 --max=20 --cpu-percent=70
kubectl autoscale deployment crawler-pool --min=5 --max=50 --cpu-percent=80

# 检查部署状态
kubectl get pods -n search-system
kubectl describe pod <pod-name> -n search-system
kubectl logs -f <pod-name> -n search-system

# 监控指标
curl http://prometheus:9090/api/v1/query?query=search_agent_latency_p99
```

## 部署资源需求

| 组件 | CPU | 内存 | 存储 | 副本数 | 高可用配置 |
|-----|-----|------|------|--------|----------|
| MCP Service | 4 cores | 8GB | - | 3 | 跨 Zone 部署 |
| Search Agent | 8 cores | 16GB | - | 3 | 跨 Zone 部署 |
| Prompt Parser | 16 cores | 32GB | GPU(可选) | 2-5 | 动态扩缩 |
| Crawler Pool | 32 cores | 64GB | - | 5-50 | 基于负载 |
| 向量索引集群 | 16 cores | 64GB | 1-2TB | 3 | 三副本 |
| 倒排索引集群 | 8 cores | 32GB | 500GB | 3 | 三副本 |
| 消息队列 | 8 cores | 16GB | 500GB | 3 | 三副本 |
| 元数据数据库 | 4 cores | 16GB | 100GB | 2 | 主从复制 |

