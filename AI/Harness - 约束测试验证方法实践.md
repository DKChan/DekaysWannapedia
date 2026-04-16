# 设计方法
## 理想流程 vs 实际迭代

### 视角一：理想自顶向下（设计阶段）

```plain
PRD → 需求澄清 → 用户旅程(BDD) → 领域建模 → 系统设计 → 实现 → 单测
         ↑___________________________________________________________|
                              (发现实现约束，反馈调整设计)
```

### 视角二：实际迭代（AI编程时代更常见）

```plain
第1轮：PRD → BDD场景 → AI生成实现 → 单测暴露问题 → 发现领域概念缺失
            ↑___________________________________________|
第2轮：补充领域模型 → 调整BDD → 重新生成实现 → 单测验证
            ↑___________________________________________|
第3轮：发现性能边界 → 调整系统设计 → ...
```

# Harness TDD 单测

## AI编程时代，单测必须守住的5个约束点

### 1. **契约约束 → 保护接口测试**


```python
def test_public_api_contract_stability():
    """
    单测约束：公共API的签名、返回值结构、异常类型必须稳定
    这样BDD/接口测试才能依赖这些契约编写
    """
    result = service.process(Order(id="123", amount=100))
    
    # 约束1: 返回结构稳定
    assert hasattr(result, 'status')  
    assert hasattr(result, 'transaction_id')
    
    # 约束2: 异常类型稳定（接口测试能catch住）
    with pytest.raises(InvalidOrderError) as exc:  # 不是随意的Exception
        service.process(Order(id="", amount=-1))
    assert exc.value.error_code == "ORDER_INVALID"  # 错误码稳定
```

**给上层测试的承诺**：调用这个服务，返回结构永远是这样，异常永远可预期。


----
### 2. **边界约束 → 保护集成测试**


```python
def test_resource_boundaries():
    """
    单测约束：资源使用有明确边界，集成测试不用猜
    """
    # AI可能生成内存/CPU/连接数无界的代码
    assert service.MAX_BATCH_SIZE == 1000  # 明确暴露上限
    assert service.CONNECTION_TIMEOUT == 30  # 超时明确
    
    # 边界行为可预期
    with pytest.raises(BatchTooLargeError):
        service.process_batch([item] * 1001)  # 超过边界明确报错
```

**给上层测试的承诺**：集成测试知道什么时候会失败，失败行为是什么。


----
### 3. **状态约束 → 保护系统测试**


```python
def test_state_machine_completeness():
    """
    单测约束：所有状态转换被显式定义，系统测试能验证完整生命周期
    """
    # AI可能遗漏状态转换
    transitions = [
        (DRAFT, SUBMIT, PENDING),
        (PENDING, APPROVE, APPROVED),
        (PENDING, REJECT, REJECTED),
        (APPROVED, CANCEL, CANCELLED),  # AI可能漏这条
    ]
    
    for from_state, action, to_state in transitions:
        assert state_machine.transition(from_state, action) == to_state
    
    # 非法转换明确禁止
    assert state_machine.transition(CANCELLED, SUBMIT) is ILLEGAL
```

**给上层测试的承诺**：系统测试遍历状态机时，不会遇到未定义行为。


----
### 4. **依赖约束 → 保护集成/契约测试**


```python
def test_dependency_isolation_points():
    """
    单测约束：所有外部依赖都有明确的注入点/接口
    这样集成测试可以mock或替换
    """
    # 验证AI没有偷偷直连数据库/HTTP/文件系统
    service = PaymentService(
        repository=MockRepository(),  # 必须能注入
        gateway=MockPaymentGateway(),  # 必须能注入
        logger=MockLogger(),
    )
    
    # 约束：没有隐藏的静态调用或全局状态
    assert not hasattr(service, '_secret_db_connection')  
```

**给上层测试的承诺**：集成测试可以安全地替换依赖，不会漏掉隐藏耦合。


----
### 5. **并发约束 → 保护性能/压力测试**


```python
def test_thread_safety_boundaries():
    """
    单测约束：并发行为明确，压力测试有依据
    """
    # AI可能生成线程不安全的代码但不标注
    assert service.is_thread_safe() == True  # 明确声明
    
    # 或者：明确标注临界区和锁策略
    assert service.lock_strategy == "per_account_mutex"
    
    # 并发边界：哪些操作可以并行，哪些必须串行
    assert service.concurrent_read_allowed() == True
    assert service.concurrent_write_allowed() == False  # 明确约束
```

**给上层测试的承诺**：压力测试知道并发上限在哪里，不会遇到诡异的竞态bug。