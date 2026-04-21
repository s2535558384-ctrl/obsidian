
---

# MVC 架构中信号（Signal）与回调（Callback）的选择指南

## 一、核心原则

| 原则                   | 说明                                                                                    |
| -------------------- | ------------------------------------------------------------------------------------- |
| **View 不持有业务服务**     | View 永远不持有 `VariableManager`、`CommandManger`、`ScriptDocument` 等业务对象的指针，也永远不主动调用它们的方法。 |
| **View 只负责展示和转发**    | View 通过 `setXxx(data)` 接收数据，通过信号发射用户意图。                                               |
| **Controller 是调度中心** | Controller 持有业务服务，负责响应 View 信号、调用服务、更新 Model、向 View 推送数据。                             |

---

## 二、信号（Signal）与回调（Callback）的本质区别

| 对比维度 | 信号（Signal） | 回调（Callback） |
|----------|---------------|------------------|
| **执行方式** | 异步语义（即使同线程也是事件驱动） | 同步调用 |
| **返回值** | **无返回值** | 可以有返回值 |
| **适用场景** | 通知某事发生，不关心结果何时返回 | 需要立即获取结果用于后续逻辑 |
| **调用频率** | 低频（用户交互、状态变化） | 可高频（循环、递归中） |
| **代码连贯性** | 逻辑跨越多个函数 | 逻辑集中在一处 |
| **依赖方向** | 发射者不依赖接收者 | 调用者依赖被调用者的签名 |

---

## 三、何时使用信号（Signal）

### 适用场景

1. **用户界面交互**：按钮点击、列表选择、文本改变。
2. **异步操作通知**：网络请求完成、文件加载结束、定时器超时。
3. **状态变化广播**：Model 数据改变，需要通知多个 View 刷新。
4. **View 请求数据**：View 需要某些数据（如下拉框选项），但**不要求立即返回**。

### 典型模式：View 发射请求信号，Controller 响应并推送数据

```cpp
// ===== View 端 =====
class PropertyDialog : public QDialog {
    Q_OBJECT
public:
    void setVariables(const QStringList& vars) {
        m_combo->addItems(vars);
    }
signals:
    void requestVariables();  // 表达“我需要数据”
};

// ===== Controller 端 =====
void MainWindow::openPropertyDialog() {
    auto* dlg = new PropertyDialog(this);
    // 连接请求信号
    connect(dlg, &PropertyDialog::requestVariables, this, [this, dlg]() {
        // Controller 从服务获取数据，推送给 View
        QStringList vars = m_variableManager->getAllVariableNames();
        dlg->setVariables(vars);
    });
    dlg->exec();
}
```

### 为什么这里不用回调？

- 回调需要在构造时传入，而 Controller 可能尚未准备好数据。
- 信号允许 View **按需请求**（例如点击下拉按钮时才发射），更灵活。
- 信号天然支持异步，如果获取数据是耗时操作，可以无缝切换为多线程。

---

## 四、何时使用回调（Callback）

### 适用场景

1. **需要同步返回结果的过滤/判断逻辑**。
2. **遍历过程中对每个元素执行外部定义的操作**。
3. **算法中需要由外部决定的策略**（如排序比较函数）。

### 典型模式：View 提供回调设置接口，外部注入同步逻辑

```cpp
// ===== View 端 =====
class FileTreeWidget : public QTreeWidget {
public:
    using FileFilterCallback = std::function<bool(const QString& filePath)>;
    void setFileFilter(FileFilterCallback callback) { m_filter = callback; }

private:
    void loadDirectory(const QString& path) {
        for (const QFileInfo& file : files) {
            // 同步调用回调，立即获取结果
            if (m_filter && !m_filter(file.absoluteFilePath()))
                continue;
            // 创建树节点...
        }
    }
    FileFilterCallback m_filter;
};

// ===== Controller 端 =====
void ScriptEdtior::setupFileTree() {
    m_fileTree->setFileFilter([this](const QString& path) -> bool {
        QFileInfo info(path);
        return m_commandWord->contains(info.fileName());
    });
}
```

### 为什么这里不用信号？

- 信号没有返回值，无法在遍历循环中同步获取判断结果。
- 如果用信号，必须将当前文件列表暂存为成员变量，并在槽中继续处理，导致逻辑割裂、状态管理复杂。
- 回调保证了代码的**局部性**和**可读性**。

---

## 五、对比表格：信号 vs 回调

| 特征 | 信号 | 回调 |
|------|------|------|
| **语法复杂度** | 需要声明 `signals:`，需要 `connect` | 需要 `std::function`，直接调用 |
| **返回值** | 无 | 有 |
| **一对多** | 天然支持（多个槽） | 需手动管理多个回调 |
| **跨线程** | 天然支持（`Qt::QueuedConnection`） | 需手动处理线程安全 |
| **代码跟踪** | 调试时需要跳转，堆栈不直观 | 单步调试流畅 |
| **解耦程度** | 极高（发射者不知接收者） | 较低（调用者需知道签名） |

---

## 六、实战决策流程图

```
需要外部提供数据/逻辑？
        │
        ├─ 是否需要立即返回结果？
        │       │
        │       ├─ 是 → 使用回调
        │       │       （如：过滤、排序、判断）
        │       │
        │       └─ 否 → 使用信号请求 + Controller 推送
        │               （如：填充下拉框、获取配置）
        │
        └─ 是否通知事件发生？
                │
                └─ 是 → 使用信号
                        （如：按钮点击、数据变化）
```

---

## 七、常见误区与澄清

| 误区 | 澄清 |
|------|------|
| “信号比回调更解耦，所以优先用信号” | 解耦不应牺牲代码可读性。同步逻辑用回调更清晰。 |
| “回调会破坏 MVC 分层” | 只要回调定义在 View 内部，外部只负责注入，分层依然完整。 |
| “信号不能传参数” | 信号可以传参，但**不能返回值**。这是选择的关键。 |

---

## 八、总结金句

- **需要返回值的，用回调。**
- **只通知不等待的，用信号。**
- **View 永远不主动拉取数据，只被动接收或发射请求信号。**

这份笔记可以作为你在新项目中设计分层交互时的速查手册。