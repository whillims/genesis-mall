# BP-0045-advisor-kernel.md

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## 决策工具内核（Advisor Kernel）

**版本**: v1.0
**日期**: 2026-05-12
**类型**: 商场域后台分析进程池
**规范依据**: advisor-kernel-governance.md

---

## 1. 架构概览

```
┌─────────────────────────────────────────────────┐
│              Advisor Kernel（商场域）            │
│                                                  │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐     │
│  │ mine      │ │ score     │ │ match     │     │
│  │ 历史挖掘  │ │ 生产者评分│ │ 需求匹配  │     │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘     │
│        └─────────────┼─────────────┘            │
│                      │                          │
│              ┌───────┴───────┐                  │
│              │    report     │                  │
│              │   报告生成    │                  │
│              └───────┬───────┘                  │
│                      │                          │
│              VEC-PUB-ADVISO 矢量                 │
│              （结果推送通道）                     │
└──────────────────────┬──────────────────────────┘
                       │ API 轮询
                       ▼
              消费者域 UI（纯渲染）
```

---

## 2. 数据模型

### 2.1 交易记录（TransactionRecord）

```python
{
    "tx_id": str,              # 交易唯一ID
    "task_id": int,            # 关联任务ID
    "consumer_id": str,        # 消费者ID
    "producer_id": str,        # 生产者ID
    "product_type": str,       # 产品类型
    "status": str,             # completed/failed/timeout
    "budget_consumed": int,    # 消耗预算
    "time_limit": int,         # 时间限制
    "actual_time_s": float,    # 实际耗时（秒）
    "receipt_count": int,      # 回执数量
    "receipt_total": int,      # 回执总数
    "created_at": float,       # 创建时间戳
    "completed_at": float,     # 完成时间戳
}
```

### 2.2 生产者档案（ProducerProfile）

```python
{
    "producer_id": str,
    "total_tx": int,           # 总交易数
    "completed_tx": int,       # 完成交易数
    "failed_tx": int,          # 失败交易数
    "total_budget": int,       # 总消耗预算
    "avg_time_s": float,       # 平均耗时
    "success_rate": float,     # 成功率 (0~1)
    "score": float,            # 综合评分 (0~100)
    "score_history": [float],  # 评分历史（最近N次）
    "last_active": float,      # 最后活跃时间
}
```

### 2.3 分析结果（AnalysisResult）

```python
{
    "analysis_id": str,
    "analysis_type": str,      # mine/score/match/report
    "requested_by": str,       # 请求者
    "created_at": float,
    "result": dict,            # 具体分析结果
    "ttl": int,                # 生存时间（秒）
}
```

---

## 3. 内核组件 API

### 3.1 advisor_kernel_mine（历史模式挖掘）

```python
def mine_patterns(ctx: dict) -> dict:
    """
    挖掘历史交易模式

    输入: ctx = {"tx_records": [...], "top_n": 10}
    输出: {
        "status": "ok",
        "patterns": [
            {"producer_id": str, "total_tx": int, "success_rate": float, "avg_time_s": float},
            ...
        ],
        "hot_consumers": [...],
        "peak_hours": [...]
    }
    """
```

### 3.2 advisor_kernel_score（生产者动态评分）

```python
def score_producer(ctx: dict) -> dict:
    """
    计算生产者综合评分

    评分维度:
    - 成功率权重: 0.35
    - 速度权重: 0.25 (time_limit / actual_time)
    - 预算效率: 0.20 (1 - budget_consumed / max_budget)
    - 活跃度: 0.10 (最近交易频率)
    - 稳定性: 0.10 (评分标准差倒数)

    输入: ctx = {"producer_id": str, "tx_records": [...]}
    输出: {
        "status": "ok",
        "producer_id": str,
        "score": float,
        "dimensions": {
            "success_rate": {"value": float, "weight": 0.35, "score": float},
            "speed": {"value": float, "weight": 0.25, "score": float},
            "budget_efficiency": {"value": float, "weight": 0.20, "score": float},
            "activity": {"value": float, "weight": 0.10, "score": float},
            "stability": {"value": float, "weight": 0.10, "score": float},
        },
        "rank": str,  # S/A/B/C/D
        "history": [float]
    }
    """
```

### 3.3 advisor_kernel_match（消费者需求匹配）

```python
def match_producer(ctx: dict) -> dict:
    """
    为消费者匹配最佳生产者

    输入: ctx = {"consumer_id": str, "consumer_type": str, "task_type": str}
    输出: {
        "status": "ok",
        "consumer_id": str,
        "recommendations": [
            {"producer_id": str, "match_score": float, "reason": str},
            ...
        ]
    }
    """
```

### 3.4 advisor_kernel_report（报告生成）

```python
def generate_report(ctx: dict) -> dict:
    """
    生成标准化分析报告

    输入: ctx = {"task_id": int, "include_details": bool}
    输出: {
        "status": "ok",
        "report": {
            "task_id": int,
            "summary": str,
            "metrics": {...},
            "producer_scores": [...],
            "timeline": [...],
            "recommendations": [...]
        }
    }
    """
```

---

## 4. 矢量通道

| 通道ID | 方向 | 说明 |
|--------|------|------|
| VEC-PUB-ADVISO | 商场→消费者域 | 分析结果推送（已脱敏摘要） |

---

## 5. 目录结构

```
src/advisor/
├── __init__.py           # 模块导出
├── kernel.py             # AdvisorKernel 主类
├── mine.py               # 历史模式挖掘引擎
├── score.py              # 生产者动态评分引擎
├── match.py              # 消费者需求匹配引擎
└── report.py             # 报告生成引擎
```

---

## 6. 验收标准

- [ ] mine: 能从交易记录中提取模式（生产者排名、热门消费者、峰值时段）
- [ ] score: 能计算生产者五维评分并给出等级（S/A/B/C/D）
- [ ] match: 能根据消费者类型和任务类型推荐生产者
- [ ] report: 能生成包含摘要、指标、时间线的完整报告
- [ ] 矢量推送: 分析结果通过 VEC-PUB-ADVISO 推送
- [ ] API 集成: FastAPI 提供 /api/advisor/* 端点
- [ ] 相产品分离: 推送数据为已脱敏摘要，非原始交易记录
