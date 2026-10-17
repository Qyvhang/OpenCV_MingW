# Qt Model/View 八股与 API 实战手册（修订版）

> 适用：Qt 6.x、C++17 及以上。用于表格/列表开发、面试复习与文件扫描场景。
>
> Qt 小版本之间可能存在 API 和元类型注册要求差异；请以项目目标版本的官方文档、头文件和编译结果为准。本手册用于学习，不替代 API 文档。

## 目录

1. Model/View 核心概念
2. QModelIndex 与索引生命周期
3. QAbstractTableModel 核心函数
4. data() 与角色
5. 表头
6. 模型变更通知
7. QTableView 常用 API
8. 选择模型
9. 排序与过滤代理
10. Model 与线程
11. 高频面试问答
12. 工程检查清单

---

## 1. Model/View 核心概念

| 对象 | 负责 | 不负责 |
|---|---|---|
| Model | 提供数据、行列数量、角色数据和变化通知 | 绘制界面 |
| View | 绘制数据、处理滚动和交互 | 作为业务数据的唯一存储 |
| Delegate | 单元格绘制和编辑交互 | 持有主要业务数据 |

View 按需向 Model 查询行数、列数、单元格数据和表头。Model 不必一定使用内存容器，也可以读取数据库、文件或网络数据。

```cpp
auto *model = new FileTableModel(this);
auto *table = new QTableView(this);
table->setModel(model);
```

对于表格模型，通常一个数据记录对应一行，列数由界面字段决定。

## 2. QModelIndex 与索引生命周期

`QModelIndex` 描述模型中的一个位置，常用接口有 `row()`、`column()`、`parent()`、`isValid()` 和 `data(role)`。行、列均从 0 开始。

`QAbstractTableModel` 是扁平模型，因此其顶层索引的 parent 无效；当传入的 parent 有效时，`rowCount()` 和 `columnCount()` 通常返回 0。

**不要把普通 QModelIndex 当成永久业务 ID。** 它并非每次插入都会失效；是否仍有效取决于执行的模型操作以及模型是否正确发出结构变更通知。不要跨任意结构变更长期缓存普通索引。

`QPersistentModelIndex` 可在模型正确维护的前提下跨越部分结构变化跟踪模型项，但对应项删除或模型重置时仍可能失效。长期识别文件、用户、设备等业务对象时，优先保存稳定业务 ID，需要时再查询模型索引。

```cpp
QPersistentModelIndex saved(index);
if (saved.isValid()) {
    const QVariant value = saved.data(Qt::DisplayRole);
}
```

## 3. QAbstractTableModel 核心函数

自定义表格模型至少要实现：

```cpp
int rowCount(const QModelIndex &parent = QModelIndex()) const override;
int columnCount(const QModelIndex &parent = QModelIndex()) const override;
QVariant data(const QModelIndex &index,
              int role = Qt::DisplayRole) const override;
```

示例：

```cpp
int FileTableModel::rowCount(const QModelIndex &parent) const
{
    if (parent.isValid())
        return 0;
    return m_files.size();
}

int FileTableModel::columnCount(const QModelIndex &parent) const
{
    if (parent.isValid())
        return 0;
    return 4;
}

QVariant FileTableModel::data(const QModelIndex &index, int role) const
{
    if (!index.isValid() || index.model() != this ||
        role != Qt::DisplayRole)
        return {};

    if (index.row() < 0 || index.row() >= m_files.size() ||
        index.column() < 0 || index.column() >= columnCount())
        return {};

    const FileInfo &file = m_files.at(index.row());
    switch (index.column()) {
    case 0: return file.name;
    case 1: return file.path;
    case 2: return file.size;
    case 3: return file.modified;
    default: return {};
    }
}
```

边界检查要在访问容器之前完成。若模型还要支持编辑、颜色、图标、勾选框等功能，需要按角色返回相应数据，并按需重写 `flags()`、`setData()` 等函数。

## 4. data() 与角色

同一个单元格可以针对不同角色返回不同值：

| 角色 | 常见用途 |
|---|---|
| `Qt::DisplayRole` | 显示文本 |
| `Qt::EditRole` | 编辑值 |
| `Qt::TextAlignmentRole` | 对齐方式 |
| `Qt::ForegroundRole` / `BackgroundRole` | 前景色 / 背景色 |
| `Qt::FontRole` | 字体 |
| `Qt::DecorationRole` | 图标 |
| `Qt::ToolTipRole` | 提示文本 |
| `Qt::CheckStateRole` | 勾选状态 |
| `Qt::UserRole` 及以上 | 自定义数据，如排序用原始数值 |

`DisplayRole` 和 `EditRole` 可以返回同一值，但不是必须相同。若显示文本经过格式化，例如“1.5 GB”，不要把它当作原始字节数用于计算或排序；应另设返回数值的自定义角色。

## 5. 表头

表头通过 `headerData()` 提供，而不是把列名塞进普通数据行：

```cpp
QVariant FileTableModel::headerData(
    int section, Qt::Orientation orientation, int role) const
{
    if (role != Qt::DisplayRole)
        return {};

    if (orientation == Qt::Horizontal) {
        switch (section) {
        case 0: return tr("文件名");
        case 1: return tr("路径");
        case 2: return tr("大小");
        case 3: return tr("修改时间");
        default: return {};
        }
    }
    return section + 1;
}
```

## 6. 模型变更通知

只修改内部容器不会自动告诉 View 结构发生变化。插入和删除必须使用匹配的 begin/end 通知。

```cpp
void FileTableModel::addFile(const FileInfo &file)
{
    const int row = m_files.size();
    beginInsertRows(QModelIndex(), row, row);
    m_files.push_back(file);
    endInsertRows();
}

void FileTableModel::removeFile(int row)
{
    if (row < 0 || row >= m_files.size())
        return;

    beginRemoveRows(QModelIndex(), row, row);
    m_files.removeAt(row);
    endRemoveRows();
}
```

关键规则：

- `beginInsertRows()` 必须在数据插入前调用；`endInsertRows()` 在容器更新后调用。
- `first` 和 `last` 是闭区间行号，不是行数。
- 已有单元格内容变化使用 `dataChanged()`，并传递受影响的索引和相关角色。
- 结构变化使用对应的插入、删除或移动协议，不能用 `dataChanged()` 代替。
- 全量重建可用 `beginResetModel()` / `endResetModel()`，但会使视图选择、索引等状态重置；能精确通知时不要滥用 reset。
- 批量插入优先一次通知覆盖连续的新行，避免大量重复更新。

```cpp
const QModelIndex changed = index(row, column);
emit dataChanged(changed, changed,
                 {Qt::DisplayRole, Qt::EditRole});
```

## 7. QTableView 常用 API

```cpp
table->setModel(model);
table->setAlternatingRowColors(true);
table->setSelectionBehavior(QAbstractItemView::SelectRows);
table->setSelectionMode(QAbstractItemView::SingleSelection);
table->setEditTriggers(QAbstractItemView::NoEditTriggers);
table->horizontalHeader()->setSectionResizeMode(QHeaderView::Stretch);
table->verticalHeader()->setVisible(false);
```

注意：

- `SelectionBehavior` 决定选择单元格、行或列；`SelectionMode` 决定单选、多选等行为。
- 编辑能力由模型 `flags()` 中的 `Qt::ItemIsEditable` 与 View 的 `editTriggers` 共同决定。
- `QHeaderView::setSectionResizeMode()` 是当前推荐 API；`setStretchLastSection(true)` 仍然有效，但它只控制最后一列拉伸，与所有列使用 `Stretch` 不同。
- `setSortingEnabled(true)` 会启用表头排序交互，但普通 `QAbstractTableModel` 不会因此自动重排内部数据。

## 8. 选择模型

`QTableView` 设置 Model 后会关联一个 `QItemSelectionModel`：

```cpp
QModelIndex current = table->currentIndex();
QModelIndexList selectedRows =
    table->selectionModel()->selectedRows();
```

当前索引和选中项不是同一概念。若 View 使用代理模型，返回的索引属于代理模型，取业务记录前应先映射到源模型。不要从 View 代码直接访问 Model 的私有容器。

可以给源模型提供受控的读取接口：

```cpp
std::optional<FileInfo> FileTableModel::fileAt(int row) const
{
    if (row < 0 || row >= m_files.size())
        return std::nullopt;
    return m_files.at(row);
}
```

如果使用指针或引用返回内部容器元素，要注意底层容器扩容、删除和重建可能使其失效，不应长期保存。

## 9. 排序与过滤代理模型

常见结构：

```text
QTableView <- QSortFilterProxyModel <- FileTableModel <- 数据容器
```

```cpp
auto *proxy = new QSortFilterProxyModel(this);
proxy->setSourceModel(model);
proxy->setDynamicSortFilter(true);
proxy->setFilterKeyColumn(0);
proxy->setFilterFixedString("cpp");
proxy->sort(2, Qt::DescendingOrder);
table->setModel(proxy);
```

过滤和排序改变显示顺序，不应把代理行号当作源模型行号。取源索引：

```cpp
const QModelIndex proxyIndex = table->currentIndex();
if (proxyIndex.isValid()) {
    const QModelIndex sourceIndex = proxy->mapToSource(proxyIndex);
    // 用 sourceIndex 访问源模型；业务层更推荐稳定 ID。
}
```

默认排序依赖 `sortRole` 返回的值。若角色返回字符串，可能出现按字典序排序（例如字符串“10”排在“9”前）。推荐让排序角色返回真实数值：

```cpp
// Model::data()
if (role == Qt::UserRole && index.column() == 2)
    return file.size; // 真实字节数，而不是“1.5 GB”之类的格式化文本

// 配置代理
proxy->setSortRole(Qt::UserRole);
```

若必须重写 `lessThan()`，应处理类型转换失败等情况，而不是无条件假定 `toLongLong()` 成功。过滤角色和排序角色是独立设置的。

## 10. Model 与线程

### 10.1 基本边界

- `QWidget` 及其子类只能在 GUI 线程使用。
- 对挂接到 GUI View 的 Model，建议统一在 GUI 线程读写。
- QObject 有线程亲和性，但 `moveToThread()` 不会让对象自动线程安全，也不会把一个正在执行的阻塞函数变成异步任务。
- 普通跨线程函数调用不会自动排队；跨线程信号在适当的 queued 连接下会将调用排入接收线程事件循环。

本手册采用的工程约定是：Worker 不直接修改挂接到 View 的 Model；Worker 发信号，GUI 线程中的槽再更新 Model。

### 10.2 推荐数据流

```text
Worker 扫描目录
  -> emit filesFound(batch)
  -> queued signal
  -> GUI 线程槽
  -> model->addFiles(batch)
  -> View 更新
```

每找到一个文件就触发一次通知，在大量文件时会产生额外开销。可以由 Worker 分批发送结果，并在 Model 中用一次 begin/end 通知插入一批连续行。

### 10.3 自定义类型与 queued 连接

自定义类型的元类型声明/注册要求取决于目标 Qt 版本、连接方式和类型使用场景。需要时可以使用：

```cpp
// 类型定义之后
Q_DECLARE_METATYPE(FileInfo)

// 在需要运行期注册的场景中，于初始化阶段执行
qRegisterMetaType<FileInfo>("FileInfo");
```

不要把“所有 Qt 6 跨线程自定义类型都必须同时写这两句”当成无条件规则。根据目标版本文档和编译/运行时诊断决定；显式注册是兼容性手段，不是对连接语义的替代。

### 10.4 取消任务与安全退出

基础接线示例不等同于完整的生产级线程管理器。实际项目必须设计：

1. **协作式取消**：Worker 在遍历目录或处理批次时定期检查取消标志。
2. **避免随意使用 `QThread::terminate()`**：强制终止可能使锁、资源和对象状态不一致。
3. **理解 `quit()` 的边界**：它请求线程事件循环退出，不会强行中断正在运行的长时间阻塞函数。
4. **关闭窗口前停止任务**：确认工作线程按设计退出后再销毁依赖对象；不要假设销毁 QThread 会自动安全终止任务。
5. **检查生命周期**：`deleteLater()` 依赖事件循环及退出时序，避免重复删除对象或让线程仍运行时销毁线程对象。
6. **阻塞任务的取消方案**：如果 Worker 长时间占用线程执行一个槽，queued 的取消槽可能无法及时执行。可使用原子取消标志、分块任务，或适合场景的 `QThreadPool` / `QtConcurrent`。

至少测试正常完成、空目录、访问失败、大量文件、用户取消、扫描中关闭窗口和重复启动。

## 11. 高频面试问答

### Model/View 为什么分离？
数据和显示解耦；同一个 Model 可以服务多个 View，界面变化不必改写数据层。

### QAbstractTableModel 的核心函数是什么？
`rowCount()` 报行数，`columnCount()` 报列数，`data()` 按索引和角色提供数据；通常还需要实现 `headerData()`。

### 为什么 beginInsertRows/endInsertRows 不能省略？
它们是模型结构变化协议的一部分，帮助 View、选择模型和代理模型正确处理行插入。只修改容器不能替代通知。

### dataChanged 和 begin/end 通知有什么区别？
`dataChanged` 用于现有索引的内容/角色发生变化；插入、删除、移动要用相应结构变化协议。

### 为什么代理模型索引不能直接当源模型索引？
代理模型可过滤和重排行，显示位置与源模型位置不一定相同。需要用 `mapToSource()` / `mapFromSource()` 转换。

### 数字排序为什么可能错误？
如果排序角色返回的是格式化字符串，可能按字典序排序。应提供真实数值角色或实现有类型检查的 `lessThan()`。

### Worker 能否直接调用挂接到 QTableView 的 Model？
本手册的工程规则是不直接调用。Worker 通过 queued 信号传回数据，由 GUI 线程中的槽修改 Model。

### QModelIndex 能否长期保存？
不要把它当成跨任意结构变化的稳定引用。可用 `QPersistentModelIndex` 跟踪部分模型项，但业务对象长期识别应优先使用稳定 ID。

## 12. 工程检查清单

- [ ] `rowCount()` / `columnCount()` 正确处理有效 parent。
- [ ] `data()` 检查索引有效性、模型来源、行列边界和角色。
- [ ] `headerData()` 与列定义一致。
- [ ] 插入、删除、移动、重置使用正确的模型通知协议。
- [ ] 现有数据变化发出带正确角色的 `dataChanged()`。
- [ ] View 不访问 Model 的私有容器。
- [ ] 代理模型索引在访问源数据前已映射。
- [ ] 排序角色提供可比较的正确类型。
- [ ] GUI View 所使用的 Model 在 GUI 线程更新。
- [ ] 取消、关闭窗口、异常路径和线程清理经过测试。
- [ ] 针对实际使用的 Qt 版本完成编译和运行验证。

## 修订说明

本版重点修正了索引生命周期的绝对化表述、代理索引与源索引混用、直接访问私有容器、自定义类型元类型注册的过度概括、数值排序的转换假设，以及线程取消和对象生命周期说明。代码片段用于讲解关键机制，仍应在目标 Qt 版本中构建、运行并测试边界场景。
