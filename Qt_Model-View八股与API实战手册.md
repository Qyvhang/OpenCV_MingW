# Qt Model/View 八股与 API 实战手册

> 适用：Qt 6.x、C++17 及以上
> 用途：表格/列表/树视图开发、面试复习、文件扫描等「列表式」界面快速上手
>
> API 主体按 Qt 6.8.3 核对；Qt 6 的小版本在重载和弃用状态上可能有差异，项目编译时以本机 Qt 文档和头文件为准。
> 本文以 QAbstractTableModel + QTableView 为主线，文中示例默认绑定文末的 FileTableModel 场景（文件名/路径/大小/修改时间）。

---

## 目录

1. 核心模型
2. QModelIndex 与索引
3. QAbstractTableModel 的三个纯虚函数
4. data() 与角色
5. headerData() 与表头
6. 模型修改通知
7. QTableView 常用 API
8. 选择模型
9. 排序与过滤代理模型
10. Model 与线程
11. 高频面试问答
12. 项目检查清单

---

# 1. 核心模型

## 1.1 为什么要把数据和界面分开

Model/View 是把「数据本身」和「数据的显示」拆成两个对象的设计：

| 对象 | 负责什么 | 不负责什么 |
|---|---|---|
| Model | 提供数据、报告行列数量、通知变化 | 不画界面 |
| View | 绘制表格、处理交互（点击、滚动） | 不保存数据 |
| Delegate | 单个单元格的绘制和编辑 | 不持有业务数据 |

分离后能获得三件事：

- 同一份 Model 可以同时给表格、树、图表等多个 View 使用；
- 换界面（表格换成列表）不用改数据代码；
- 数据量很大时，View 只按需向 Model 索取当前可见的数据，而不是一次加载全部。

Qt 里这种设计叫 Model/View 架构，别叫 MVC——Qt 官方用的就是 Model/View 这个说法。

## 1.2 QTableView 自己保存数据吗

不保存。

~~~cpp
auto *model = new FileTableModel(this);
auto *table = new QTableView(this);

table->setModel(model);   // View 只“认识”Model，数据仍然只在 Model 里
~~~

表格里的每一格内容都是 View 在绘制时**实时向 Model 提问**拿到的：问行数、问列数、问某一格显示什么。Model 一旦销毁，View 立刻失去数据来源。没有持久化逻辑时，程序退出后数据自然消失。

## 1.3 反向控制流：View 来问，Model 来答

新手最容易搞反的方向：**不是 Model 把数据推给 View，而是 View 在需要绘制时来问 Model。**

~~~text
QTableView 开始绘制
      ↓
这个表格有多少行？    →  rowCount()
有多少列？           →  columnCount()
第 (r, c) 格显示什么？ →  data(index, role)
列头写什么？          →  headerData(section, orientation, role)
      ↓
View 把拿到的答案画到屏幕上
~~~

## 1.4 数据存在哪

QAbstractTableModel 只是接口，不提供任何存储。数据容器由你自己选：

~~~cpp
class FileTableModel : public QAbstractTableModel {
    // ...
private:
    QVector<FileInfo> m_files;   // 数据真正存在这里
};
~~~

一个元素 = 表格的一行；行数自然就是 `m_files.size()`。列数是你的界面设计（4 列），由 `columnCount()` 写死返回，和容器大小无关。

---

# 2. QModelIndex 与索引

## 2.1 QModelIndex 是什么

View 用 QModelIndex 告诉 Model「我现在要的是哪一格」。它内部记着三样东西：

- `row()` —— 第几行（从 0 开始）；
- `column()` —— 第几列（从 0 开始）；
- `parent()` —— 父索引（表格模型下永远是无效索引）。

注意：**行在前，列在后，都从 0 开始计数。**「第 3 行第 2 列」写成索引就是 `row() == 2`、`column() == 1`。

## 2.2 QModelIndex 常用 API

| API | 返回值 | 作用 |
|---|---|---|
| row | int | 行号，从 0 开始 |
| column | int | 列号，从 0 开始 |
| parent | QModelIndex | 父索引，表格模型下无效 |
| isValid | bool | 索引是否指向一个有效位置 |
| data(role) | QVariant | 直接取这个索引对应格子的数据 |
| sibling(row, column) | QModelIndex | 同一父节点下指定行列的兄弟索引 |

`QModelIndex()` 默认构造出来的是一个**无效索引**，`isValid()` 返回 false。Qt 常用它表示「不存在」或「根」，所以表格模型的 `parent.isValid()` 才总是 false。

## 2.3 两个重要结论

- QModelIndex 是轻量值类型（内部通常就是指针 + 行 + 列），可以随意拷贝和比较；
- 但它是**短暂**的：底层 Model 一旦插入、删除或排序，旧索引就可能指向错误的位置甚至失效。需要长期保存某个位置时用 QPersistentModelIndex，而不是裸存 QModelIndex；

~~~cpp
QPersistentModelIndex saved(index);   // index 来自任意来源，例如 View 的当前格
// 之后结构变化（插入、排序）时，持续索引会尽量跟随同一个数据项；
// 若那个数据项本身被删除，持续索引同样会失效，用时仍要以 isValid() 检查。
~~~

---

# 3. QAbstractTableModel 的三个纯虚函数

QAbstractTableModel 要求子类必须实现：

~~~cpp
int rowCount(const QModelIndex &parent = QModelIndex()) const override;
int columnCount(const QModelIndex &parent = QModelIndex()) const override;
QVariant data(const QModelIndex &index, int role = Qt::DisplayRole) const override;
~~~

## 3.1 rowCount / columnCount

~~~cpp
int FileTableModel::rowCount(const QModelIndex &parent) const
{
    if (parent.isValid())
        return 0;

    return m_files.size();   // 几行 = 容器里几个文件
}

int FileTableModel::columnCount(const QModelIndex &parent) const
{
    if (parent.isValid())
        return 0;

    return 4;                // 几列 = 你的界面设计，写死
}
~~~

必须记住：

- rowCount 通常动态，取决于容器大小；
- columnCount 通常写死，取决于表格要显示几个字段；
- 两个函数都要处理 `parent.isValid()`。

## 3.2 为什么 parent.isValid() 时返回 0

QAbstractTableModel 是**扁平**结构，没有父子层级。表格里的每一行都是顶层（父 = 根 = 无效索引），不存在「某一行下面还有子行」。

当 View 问「这个父节点下有几行」时：

- parent 无效 → 问的是根下的行数 → 返回 `m_files.size()`；
- parent 有效 → 表格模型没有任何节点有孩子 → 返回 0。

树形模型（QAbstractItemModel 实现层级）才会在 parent 有效时返回非 0。表格模型这段基本是固定模板，抄即可。

## 3.3 data()：按索引取一格

~~~cpp
QVariant FileTableModel::data(const QModelIndex &index, int role) const
{
    if (!index.isValid() || role != Qt::DisplayRole)
        return {};

    if (index.row() < 0 || index.row() >= m_files.size())
        return {};
    if (index.column() < 0 || index.column() >= columnCount())
        return {};

    const FileInfo &file = m_files.at(index.row());

    switch (index.column()) {
    case 0: return file.name;      // 第 0 列 ← 文件名
    case 1: return file.path;      // 第 1 列 ← 路径
    case 2: return file.size;      // 第 2 列 ← 大小
    case 3: return file.modified;  // 第 3 列 ← 修改时间
    default: return {};
    }
}
~~~

流程是两步定位：先按 `index.row()` 取出是哪条文件记录，再按 `index.column()` 决定显示这条记录的哪个字段。

必须记住：

- 防御检查的顺序可以随意，但 `at()` 之前**必须**确认 row 不越界；
- `columnCount()` 返回几，switch 就必须覆盖到「几减一」，漏一个 case 那一列就是静默空白——不报错、不崩溃、最难查。

---

# 4. data() 与角色

## 4.1 role 是什么

View 不只是问「格子显示什么文字」这一件事。同一个格子，View 会问很多类问题：

- 文字内容？
- 文字颜色？
- 背景色？
- 对齐方式？
- 字体？
- 图标？
- 能不能编辑？

role 就是「问题的类型」。Qt 用 Qt::ItemDataRole 枚举报种类：

| 角色 | 含义 |
|---|---|
| Qt::DisplayRole | 显示文本 |
| Qt::EditRole | 编辑框里的内容（通常与 Display 相同） |
| Qt::TextAlignmentRole | 对齐方式，返回 int；Qt 6 也允许 QVariant::fromValue(Qt::Alignment) |
| Qt::ForegroundRole | 文字颜色，返回 QColor |
| Qt::BackgroundRole | 背景颜色 |
| Qt::FontRole | 字体，返回 QFont |
| Qt::DecorationRole | 图标，返回 QIcon |
| Qt::ToolTipRole | 悬停提示文本 |
| Qt::CheckStateRole | 勾选框状态，返回 Qt::Checked / Qt::Unchecked |
| Qt::UserRole | 自定义角色起点，返回任意 QVariant |

## 4.2 第一行判断的含义

~~~cpp
if (!index.isValid() || role != Qt::DisplayRole)
    return {};
~~~

这两条是两件完全不同的事：

- `index.isValid()` —— 这个索引是否有效（不在范围内就无效）；
- `role != Qt::DisplayRole` —— 这次提问的类型不是「显示文本」。

只处理自己关心的角色，其余一律返回空 QVariant，View 会按默认值处理。不要把它理解成「判断格子里有没有数据」。

## 4.3 角色实战：给大小列上颜色

~~~cpp
QVariant FileTableModel::data(const QModelIndex &index, int role) const
{
    if (!index.isValid())
        return {};
    if (index.row() < 0 || index.row() >= m_files.size())
        return {};

    const FileInfo &file = m_files.at(index.row());

    if (role == Qt::DisplayRole) {
        switch (index.column()) {
        case 0: return file.name;
        case 1: return file.path;
        case 2: return file.size;
        case 3: return file.modified;
        }
    } else if (role == Qt::TextAlignmentRole) {
        if (index.column() == 2)
            return int(Qt::AlignRight | Qt::AlignVCenter);
    } else if (role == Qt::ForegroundRole) {
        if (index.column() == 2 && file.size > 1024 * 1024)
            return QColor(Qt::red);
    }

    return {};
}
~~~

同一个格子，不同 role 返回不同类型：DisplayRole 返回 QString，TextAlignmentRole 返回 int 对齐标志（写成 int 在 Qt 5 / Qt 6 都兼容），ForegroundRole 返回 QColor。QVariant 装得下这些类型。这个示例同样补了行边界检查——isValid() 只表示「位置合法」，并不能保证 row 不越界。

## 4.4 编辑需要另外两个虚函数

View 只能读，**不能直接写**数据。想支持编辑，Model 还要重写：

~~~cpp
Qt::ItemFlags flags(const QModelIndex &index) const override;
bool setData(const QModelIndex &index, const QVariant &value, int role) override;
~~~

真正决定「能不能编辑」的是两层都要放行：

1. `flags()` 里要包含 `Qt::ItemIsEditable`。QAbstractItemModel 默认的 flags() 只含 `Qt::ItemIsEnabled | Qt::ItemIsSelectable`——所以展示型模型即使不设 NoEditTriggers，双击也不会出现编辑框；
2. View 的 editTriggers 要允许触发编辑（第 7.2 节）。

setData 是 View 把用户输入写回 Model 的入口，最小实现：

~~~cpp
bool FileTableModel::setData(const QModelIndex &index,
                             const QVariant &value, int role)
{
    if (!index.isValid() || role != Qt::EditRole)
        return false;
    if (index.row() < 0 || index.row() >= m_files.size())
        return false;

    switch (index.column()) {
    case 0: m_files[index.row()].name = value.toString(); break;
    default: return false;
    }

    emit dataChanged(index, index, { Qt::DisplayRole, Qt::EditRole });
    return true;
}
~~~

三件套缺一不可：**flags 放行 + setData 保存 + emit dataChanged 通知**。示例按 `role == Qt::EditRole` 接收是标准写法；若项目里的自定义 delegate 用 DisplayRole 回写，setData 的 role 判断要按它实际传入的角色兼容。文件扫描结果只做展示，这两项完全不需要。

---

# 5. headerData() 与表头

## 5.1 三个参数

~~~cpp
QVariant FileTableModel::headerData(int section,
                                    Qt::Orientation orientation,
                                    int role) const
~~~

| 参数 | 含义 |
|---|---|
| section | 第几个表头，从 0 开始 |
| orientation | Qt::Horizontal（列头）或 Qt::Vertical（行头） |
| role | 问什么类型，通常只处理 Qt::DisplayRole |

实现：

~~~cpp
QVariant FileTableModel::headerData(int section,
                                    Qt::Orientation orientation,
                                    int role) const
{
    if (role != Qt::DisplayRole || orientation != Qt::Horizontal)
        return {};

    switch (section) {
    case 0: return QStringLiteral("文件名");
    case 1: return QStringLiteral("路径");
    case 2: return QStringLiteral("大小");
    case 3: return QStringLiteral("修改时间");
    default: return {};
    }
}
~~~

必须记住：

- `Qt::Horizontal` 是**列头**（横着排的四个字段名）；
- `Qt::Vertical` 是**行头**（左侧的行号），不处理时返回空，View 会用默认序号 1、2、3…。

## 5.2 行头怎么藏

行号列是 View 的 QHeaderView 画出来的，藏不藏是 View 的事，不是 Model 的 headerData 决定的：

~~~cpp
table->verticalHeader()->setVisible(false);   // 隐藏左侧行号
~~~

---

# 6. 模型修改通知

## 6.1 为什么数据改了界面不动

View 有自己内部的行数缓存和索引状态。`m_files.push_back()` 只是改了你的容器，**Qt 完全不知道**。必须显式告诉 Model「结构变了」，Model 再转发给 View。通知的原语就是 begin/end 函数对。

## 6.2 addFile 标准写法

~~~cpp
void FileTableModel::addFile(const FileInfo &file)
{
    const int row = m_files.size();            // ① 先算行号

    beginInsertRows(QModelIndex(), row, row);  // ② 通知：准备插入
    m_files.push_back(file);                   // ③ 真正改数据
    endInsertRows();                           // ④ 通知：插入完成
}
~~~

执行语义：

~~~text
当前有几行
    ↓
新行下标 = 当前行数（0 起始，等价于插入前的 size()）
    ↓
beginInsertRows 声明“我要在 row 处插入一行”
    ↓
数据真正进入容器
    ↓
endInsertRows 让 View 刷新
~~~

## 6.3 beginInsertRows 的三个参数

~~~cpp
void beginInsertRows(const QModelIndex &parent, int first, int last);
~~~

| 参数 | 含义 | 表格模型传什么 |
|---|---|---|
| parent | 插入位置的父亲节点 | `QModelIndex()`，无效索引 = 根 |
| first | 新行区间的**起始**下标 | row |
| last | 新行区间的**结束**下标 | row（插入一行时两者相同） |

必须记住：

- `first` 和 `last` 是**闭区间**，都是行下标，不是数量。一次插 5 行就是 `beginInsertRows(QModelIndex(), row, row + 4)`；
- parent 必须和数据的真实位置一致：表格数据都在根下，所以传无效索引。

## 6.4 顺序为什么不能调换

“**先 begin，改数据，后 end**”是硬性协议：

- `beginInsertRows` 必须在数据改动**之前**调用——View 需要在这个时刻记下改动前的结构；
- 如果先 push 再 begin，View 记下的「旧状态」其实已经是新的，对比不出差异，刷新照样失效；
- 忘了 begin 只写 end（或反过来），属于违反协议，Qt 内部状态错乱，可能崩溃或刷新异常。

## 6.5 只 push 不通知的后果

数据在，界面不刷新，通常立即可见。更深的问题在后面：

- View 缓存的行数和真实容器大小错位；
- 用户点击靠后的行时，View 用**旧**行号去问 data()，拿到错误的文件甚至越界。

所以「忘通知」不是少了刷新动画那么简单，是索引状态被污染。

## 6.6 删除与批量

~~~cpp
void FileTableModel::removeFile(int row)
{
    if (row < 0 || row >= m_files.size())
        return;                                // 先边界检查

    beginRemoveRows(QModelIndex(), row, row);
    m_files.removeAt(row);
    endRemoveRows();
}

void FileTableModel::addFiles(const QVector<FileInfo> &files)
{
    if (files.isEmpty())
        return;

    const int first = m_files.size();          // 取一次，改数据前算好
    const int last  = first + files.size() - 1;

    beginInsertRows(QModelIndex(), first, last);
    m_files.append(files);
    endInsertRows();
}
~~~

「先算行号、取一次 size、改数据之前全部定好」是同一个原则：**插入通知的参数描述新行的插入位置；删除通知的参数描述删除前要移除的现有行区间。两者都必须在改数据之前算出来。**

## 6.7 修改已有数据用 dataChanged

插入/删除用 begin/end，**已有格子内容变化**用另一个信号：

~~~cpp
m_files[index.row()].name = newName;
emit dataChanged(index, index, { Qt::DisplayRole });
~~~

- 参数是「变化的矩形区域」的左上角和右下角索引；
- 第三个参数是变化的角色列表。

## 6.8 全量刷新用 reset

结构完全推倒重来（比如清空后重扫）时的方案：

~~~cpp
beginResetModel();
m_files.clear();
m_files = newList;   // 或任何大规模重建
endResetModel();
~~~

副作用：View 的选择、滚动位置、展开状态全部丢失，QPersistentModelIndex 也会大范围失效。能用 begin/end 精确通知时不要用 reset；扫描进度这类高频变化更不能拿 reset 当刷新手段。

---

# 7. QTableView 常用 API

## 7.1 接上 Model

~~~cpp
auto *model = new FileTableModel(this);
auto *table = new QTableView(this);

table->setModel(model);
~~~

Model 可以没有 parent（不 setModel 的 View 也是空表）；View 不拥有 Model，setModel 后谁都不自动替对方 delete。

## 7.2 显示与交互设置

| API | 作用 |
|---|---|
| setAlternatingRowColors(bool) | 斑马纹隔行变色 |
| setSelectionBehavior(QAbstractItemView::SelectRows) | 点击选中整行 |
| setSelectionMode(QAbstractItemView::SingleSelection) | 单选：一次只能选中一行 |
| setEditTriggers(QAbstractItemView::NoEditTriggers) | 禁止编辑触发（展示型表格建议设置） |
| setShowGrid(bool) | 是否显示网格线 |
| setWordWrap(bool) | 单元格文字换行 |
| setTextElideMode(Qt::ElideMiddle) | 长文本省略号方式 |
| setSortingEnabled(true) | 点表头排序（基础版） |
| setColumnWidth(col, px) | 指定列宽 |
| setColumnHidden(col, true) | 隐藏某列 |
| resizeColumnsToContents() | 列宽适应内容 |
| horizontalHeader()->setSectionResizeMode(QHeaderView::Stretch) | 列自动拉伸填满 |

QAbstractItemView::SelectionBehavior / SelectionMode / EditTrigger 是三个独立枚举：

- SelectionMode 例：NoSelection、SingleSelection、ExtendedSelection（拖动连续选）、MultiSelection（逐个点选）；
- SelectionBehavior 例：SelectItems（选格）、SelectRows（选整行）、SelectColumns（选整列）；
- EditTriggers 例：NoEditTriggers、DoubleClicked、SelectedClicked、EditKeyPressed（F2）、AnyKeyPressed、AllEditTriggers。

必须记住：

- 「点击整行高亮」是 setSelectionBehavior(SelectRows)，新手常误以为是 SelectionMode；
- 「避免双击编辑」建议设 setEditTriggers(NoEditTriggers)，但真正决定能否编辑的是 flags() 是否含 ItemIsEditable 和 editTriggers 两层——QAbstractTableModel 默认 flags() 不含 ItemIsEditable，展示型模型不设也不会出现编辑框；
- `setSortingEnabled(true)` 只是让表头可点击并调用 model->sort()，普通 QAbstractTableModel 默认的 sort() 不会真正重排数据；真正的排序走第 9 章的代理模型。

## 7.3 列宽细节

`setColumnWidth` 是固定像素，窗口拉大不会跟着变。要「最后一列吃掉剩余空间」用：

~~~cpp
table->horizontalHeader()->setSectionResizeMode(QHeaderView::Stretch);
~~~

它是 QHeaderView 的方法，要通过 horizontalHeader() 拿到表头对象再调。注意两个常见 API 的现状：`setResizeMode` 已被 `setSectionResizeMode` 替代（弃用）；`setStretchLastSection(bool)` **没有**弃用，它仍是「最后一列拉伸填满剩余空间」的专用开关，与 `setSectionResizeMode(QHeaderView::Stretch)`（所有列均匀拉伸）语义不同。

## 7.4 拿到当前行的数据

View 永远通过 index 和 Model 交互：

~~~cpp
QModelIndex index = table->currentIndex();      // 当前格
if (!index.isValid())
    return;

const FileInfo &file = model->m_files.at(index.row());   // 取决于你的访问设计
~~~

如果 View 接的是 QSortFilterProxyModel，`index.row()` 是**代理行号**，直接拿去取源数据会错位——先 `proxy->mapToSource(index).row()` 拿到源行号再取值（见 9.2）。

更正规的路径是给 Model 加一个取值方法：

~~~cpp
// FileTableModel 内
const FileInfo *fileAt(int row) const
{
    if (row < 0 || row >= m_files.size())
        return nullptr;
    return &m_files.at(row);
}
~~~

View 不回传数据，取数据永远走 Model。

---

# 8. 选择模型

## 8.1 多选时怎么取

setModel 之后 Qt 会自动为 View 建一个 QItemSelectionModel 记录选择状态：

~~~cpp
QItemSelectionModel *sel = table->selectionModel();

QModelIndexList rows = sel->selectedRows();    // 所有选中行的第一列索引
int row = sel->currentIndex().row();           // 当前行（虚实线框那一格所在行）
~~~

注意「当前项」和「选中项」是两回事：ExtendedSelection 下拖一片，current 只是最后点击的那一格，selectedRows 才是全部。

## 8.2 常用 API 与信号

| API / 信号 | 作用 |
|---|---|
| selectedRows() | 所有选中行的索引列表 |
| selectedIndexes() | 所有选中单元格的索引列表 |
| currentIndex() | 当前格的索引 |
| clearSelection() | 清空选择 |
| setCurrentIndex(index, flags) | 程序化设置当前格 |
| signal selectionChanged | 选中集合变化 |
| signal currentChanged | 当前格变化 |

双击事件走 View 的信号：

~~~cpp
connect(table, &QTableView::doubleClicked, this,
        [this](const QModelIndex &index) {
            // index.row() 就是用户双击的那一行
        });
~~~

---

# 9. 排序与过滤代理模型

## 9.1 为什么要代理

排序、过滤如果写进 Model 本身，数据就得真的重排——源数据顺序被破坏，而且过滤时行数对不上。QSortFilterProxyModel 站在 View 和 Model 中间，**只改展示映射，不动源数据**：

~~~text
QTableView ← QSortFilterProxyModel ← FileTableModel ← m_files
~~~

~~~cpp
auto *proxy = new QSortFilterProxyModel(this);
proxy->setSourceModel(model);
table->setModel(proxy);              // View 接的是代理
~~~

之后对表格的所有操作（排序、过滤、取 index）都对着 proxy；proxy 内部的 `mapToSource()` / `mapFromSource()` 负责把索引翻译回真实 Model。

## 9.2 常用 API

| API | 作用 |
|---|---|
| setSourceModel(model) | 设置源模型 |
| setSortRole(role) | 用哪个角色比较排序 |
| sort(column, order) | 按某列排序，order 取 Qt::AscendingOrder / DescendingOrder |
| setDynamicSortFilter(true) | 源数据变化时自动重新排序过滤 |
| setFilterKeyColumn(col) | 过滤只看哪一列 |
| setFilterFixedString(text) | 文本过滤，包含即通过 |
| setFilterRegularExpression(re) | 正则过滤 |
| setFilterCaseSensitivity(flag) | 是否区分大小写 |
| mapToSource(proxyIndex) | 代理索引 → 源索引 |
| mapFromSource(sourceIndex) | 源索引 → 代理索引 |

必须记住：排序和过滤之后，**显示行号和真实行号不再一样**。比如过滤后表上第 0 行，源数据里可能是第 7 行。要拿真正的 FileInfo，先 `mapToSource` 再 `row()`。

## 9.3 默认排序够用吗

默认 lessThan 按 sortRole 返回的 QVariant 比较：data() 返回字符串就按字典序（`10` 会排在 `9` 前面），返回数值类型则按数值比较。想让「大小」列按数值排、或让目录永远排前面，要子类化重写 `lessThan()`：

~~~cpp
bool MyProxyModel::lessThan(const QModelIndex &left,
                            const QModelIndex &right) const
{
    if (left.column() == 2)          // 大小列按数值比较
        return left.data().toLongLong() < right.data().toLongLong();
    return QSortFilterProxyModel::lessThan(left, right);
}
~~~

典型面试点：数字列如果以字符串返回就会按字典序（`10` 排在 `9` 前）——要么重写 lessThan 按数值比较，要么统一让这一列返回数值类型。

## 9.4 显示文本和排序值分离：sortRole + UserRole

想在界面上显示 `1.5 GB`，但排序按字节数比较，就让 Model 用自定义角色存放排序值，再让代理按该角色排序：

~~~cpp
// Model::data() 内
if (role == Qt::UserRole && index.column() == 2)
    return file.size;        // 数值：专用于排序

// 使用处
proxy->setSortRole(Qt::UserRole);   // 按 UserRole（数值）排序
~~~

默认 sortRole 是 Qt::DisplayRole；过滤默认看 filterRole（同样默认 DisplayRole），可以用 setFilterRole 换成别的角色。排序和过滤是两组独立的角色配置，彼此不影响。

---

# 10. Model 与线程

## 10.1 铁的边界

- QWidget 及其子类（包括 QTableView）只能在 GUI 主线程用；
- Model 挂在 View 上，工程上同样只由主线程修改；
- Qt 的 Model 和 View 都不是为多线程并发访问设计的，跨线程直接调用是未定义行为。

由此推出：**Worker 线程永远不碰 Model。**

## 10.2 文件扫描的正确数据流

~~~text
Worker 子线程扫描目录，发现一个文件
      ↓
emit fileFound(file);            // 只发信号，参数按值复制
      ↓
（跨线程 QueuedConnection 自动排队）
      ↓
MainWindow 的槽在主线程执行
      ↓
model->addFile(file);            // begin/end 通知也在主线程完成
      ↓
QTableView 刷新
~~~

## 10.3 错误写法

~~~cpp
// Worker::scan() 内 —— 错误！
model->addFile(file);     // Worker 线程直接改 Model
~~~

后果不是「一定会崩」，而是数据竞争和 UI 状态错乱都可能发生，属于未定义行为，不值得赌。

## 10.4 自定义类型跨线程要注册

FileInfo 是自定义 struct 时，queued 信号要求类型可复制、可默认构造，并注册元类型：

~~~cpp
// fileinfo.h，类型定义之后
Q_DECLARE_METATYPE(FileInfo)

// main() 里，启动时执行一次
qRegisterMetaType<FileInfo>("FileInfo");
~~~

- `Q_DECLARE_METATYPE` 是**编译期**声明，放头文件；
- `qRegisterMetaType` 是**运行期**调用，放 main 或模块初始化处。

Qt 6 新式 connect 下很多类型能自动注册，但跨线程 queued + 自定义结构显式注册最稳，面试也按「声明放头文件、注册放 main」答。

## 10.5 完整接线示例

~~~cpp
// MainWindow 构造中
fileModel = new FileTableModel(this);

auto *table = new QTableView(this);
table->setModel(fileModel);
table->setSelectionBehavior(QAbstractItemView::SelectRows);
table->setEditTriggers(QAbstractItemView::NoEditTriggers);
table->setAlternatingRowColors(true);

thread = new QThread(this);
worker = new ScannerWorker;               // 无 parent
worker->moveToThread(thread);

connect(thread, &QThread::started, worker, &ScannerWorker::startScan);
connect(worker, &ScannerWorker::fileFound,
        this,    &MainWindow::onFileFound);   // 跨线程，自动 Queued
connect(worker, &ScannerWorker::finished,
        thread, &QThread::quit);
connect(thread, &QThread::finished,
        worker, &QObject::deleteLater);
connect(thread, &QThread::finished,
        thread, &QObject::deleteLater);

thread->start();
~~~

主线程槽：

~~~cpp
void MainWindow::onFileFound(const FileInfo &file)
{
    fileModel->addFile(file);     // 每个文件一次 begin/end
}
~~~

## 10.6 高频文件时的批量优化

每找到一个文件就 addFile 一次，成百上千个文件会触发成百上千次 View 刷新。更好的做法是 Worker 攒一批再发：

~~~text
Worker 攒满 100 个 → emit filesFound(QVector<FileInfo>)
主线程槽 → model->addFiles(list)   // 一次 begin/end
~~~

批量版的 addFiles 见 6.6。核心原则不变：**通知放主线程，一次通知带一批数据。**

---

# 11. 高频面试问答

## 11.1 Model/View 为什么分离？

数据与显示解耦：一份 Model 多 View 复用；换界面不动数据；大数据按需索取。View 只画不管数据，Model 只供数据不画界面。

## 11.2 QTableView 自己保存数据吗？

不保存。每个格子都是 View 在绘制时向 Model 实时提问。Model 销毁，View 失去数据来源。程序没有持久化逻辑时，重启数据消失。

## 11.3 Model 一定要自己存数据吗？

不一定。QAbstractTableModel 只是接口，数据可以来自内存容器、文件、数据库或网络，只要能在 rowCount / columnCount / data 里答出来即可。

## 11.4 rowCount、columnCount、data 分别负责什么？

rowCount 报行数，columnCount 报列数，data 按索引提供某格某角色的数据。三个都是 View 来问、Model 来答。

## 11.5 data() 的第一行判断在防什么？

`!index.isValid()` 防无效索引；`role != Qt::DisplayRole` 表示本次问的不是显示文本。两者不是一回事。

## 11.6 role 是什么？

View 提问的类型。同一个格子 View 会分别问 DisplayRole、ForegroundRole、TextAlignmentRole 等，Model 按角色返回不同类型的数据。

## 11.7 QModelIndex 的 row() 和 column() 表示什么？

行列下标，都从 0 开始。第 3 行第 2 列就是 row()==2、column()==1。行在前，列在后。

## 11.8 表格模型里 parent.isValid() 为什么返回 0？

表格是扁平结构，任何有效索引都没有子行，只有根（无效索引）下有数据，所以对有效 parent 一律返回 0。

## 11.9 为什么不能只 push_back？

View 不知道数据变了：界面不刷新，且 View 缓存的索引状态和真实数据错位，后续访问可能越界或错位。

## 11.10 beginInsertRows 的参数和顺序？

parent 为无效索引，first/last 是新行区间的闭区间下标。顺序必须是：先 begin（改数据前），再改数据，最后 end。

## 11.11 新行下标怎么算？

插入前的 `m_files.size()`。size() 在 push 之前取。3 行时新行下标是 3，size() 恰好返回 3。

## 11.12 修改已有单元格和插入行的通知方式一样吗？

不一样。结构变化用 beginInsertRows / endInsertRows（或 beginRemoveRows）；已有内容变化用 `emit dataChanged(index, index, ...)`；全量重建用 beginResetModel / endResetModel。

## 11.13 headerData 的三个参数是什么？

section 是第几个表头；orientation 分 Qt::Horizontal（列头）和 Qt::Vertical（行头）；role 是提问类型。

## 11.14 行号列怎么去掉？

`table->verticalHeader()->setVisible(false);` 这是 View 的 QHeaderView 设置，不是 Model 的 headerData 所能控制的。

## 11.15 怎么让表格点击选中整行？

`table->setSelectionBehavior(QAbstractItemView::SelectRows);` 注意区分 SelectionMode（单选/多选）和 SelectionBehavior（选格/选行）。

## 11.16 怎么得到一个选中行的数据？

`table->selectionModel()->selectedRows()` 拿所有选中行索引；单个当前行用 `currentIndex().row()`；最终取业务数据通过 Model 自己的方法（如 fileAt(row)）。

## 11.17 排序为什么用代理模型？

直接排会破坏源数据顺序，且过滤时行数对不上。QSortFilterProxyModel 只改展示映射：View 接代理，代理映射回源模型。

## 11.18 过滤后表上的行号和真实行号一样吗？

不一样，必须 `mapToSource()` 转成源索引再取真实数据。

## 11.19 数字列排序为什么可能不对？

数字列如果以字符串返回 DisplayRole，默认就按字典序（`10` 排在 `9` 前）。返回数值类型、或让 sortRole 指向一个数值角色，就按数值比较；其余特殊需求重写 lessThan。

## 11.20 Worker 线程能直接调 model->addFile 吗？

不能。View 和 Model 只该主线程碰。Worker 通过信号把数据发回，主线程槽里再调 addFile。

## 11.21 跨线程传自定义 struct 要做什么？

头文件里 Q_DECLARE_METATYPE(FileInfo)，main 里 qRegisterMetaType<FileInfo>("FileInfo")。前者编译期，后者运行期，别写在一处。

## 11.22 QModelIndex 能长期保存吗？

不建议。模型变动后旧索引会失效或错位，Model 只保证索引短暂有效。长期保存用 QPersistentModelIndex。

## 11.23 columnCount()==4 但 data() 少一个 case 会怎样？

第 4 列不崩、不报错，静默显示空白。列数和 switch 分支数必须一致。

---

# 12. 项目检查清单

写 Model/View 功能前逐项回答：

1. 数据存哪里？容器、数据库还是其他数据源？
2. 行数来自什么？列数写几？
3. columnCount 返回的列数，data() 和 headerData() 是否都覆盖到了？
4. data() 有没有 isValid、行列越界、role 三组检查？
5. 每次增删结构是否都有 begin/end 通知？
6. row 是否在 push_back 之前取？
7. 修改已有数据是否 emit dataChanged？
8. 展示型表格是否确认不可编辑（NoEditTriggers + flags() 不含 ItemIsEditable）？
9. 是否设了 SelectRows、斑马纹、列宽模式？
10. 排序过滤是否走的代理模型？
11. 取数据是否先 mapToSource？
12. Worker 是否只发信号、从不直接碰 Model？
13. 自定义类型跨线程是否声明+注册？
14. 批量结果是否攒批一次 addFiles？

## 12.1 就业实用水平标准

不需要背完 Qt 全部 Model 类。达到下面标准即可进入真实文件扫描项目：

- 能说清 Model/View 的职责边界和反向提问方向；
- 能独立写 rowCount、columnCount、data、headerData；
- 能解释 role 和 DisplayRole 判断；
- 能写出完整 addFile（先算行号、begin、push、end）；
- 能区分插入通知（begin/end）与修改通知（dataChanged）；
- 能设置表格的选行、列宽，并说清编辑能力由 flags() 与 editTriggers 共同决定；
- 能说出排序过滤必须走代理模型；
- 能画 Worker → 信号 → 主线程槽 → addFile 的完整链路；
- 能注册并安全传递自定义类型。

---

# 附录：最小 API 速查

~~~cpp
// Model 必实现
int rowCount(const QModelIndex &parent) const override;      // 行数
int columnCount(const QModelIndex &parent) const override;   // 列数
QVariant data(const QModelIndex &index, int role) const override;          // 格子数据
QVariant headerData(int section, Qt::Orientation o, int role) const;       // 表头
void addFile(const FileInfo &file);                          // 自定义增行
~~~

~~~cpp
// 索引与防御
index.isValid();
index.row();
index.column();
index.parent();
if (!index.isValid() || role != Qt::DisplayRole) return {};
if (index.row() < 0 || index.row() >= m_files.size()) return {};
~~~

~~~cpp
// 修改通知
beginInsertRows(QModelIndex(), row, row);
m_files.push_back(file);
endInsertRows();

beginRemoveRows(QModelIndex(), row, row);
m_files.removeAt(row);
endRemoveRows();

emit dataChanged(index, index, { Qt::DisplayRole });
beginResetModel(); /* ... */ endResetModel();
~~~

~~~cpp
// View 设置
table->setModel(model);
table->setAlternatingRowColors(true);
table->setSelectionBehavior(QAbstractItemView::SelectRows);
table->setSelectionMode(QAbstractItemView::SingleSelection);
table->setEditTriggers(QAbstractItemView::NoEditTriggers);
table->horizontalHeader()->setSectionResizeMode(QHeaderView::Stretch);
table->verticalHeader()->setVisible(false);
table->setColumnHidden(2, true);
table->resizeColumnsToContents();
~~~

~~~cpp
// 选择
QModelIndex idx = table->currentIndex();
QModelIndexList rows = table->selectionModel()->selectedRows();
table->selectionModel()->clearSelection();
~~~

~~~cpp
// 代理模型
auto *proxy = new QSortFilterProxyModel(this);
proxy->setSourceModel(model);
table->setModel(proxy);
proxy->setDynamicSortFilter(true);
proxy->setFilterKeyColumn(0);
proxy->setFilterFixedString("cpp");
proxy->sort(2, Qt::DescendingOrder);
proxy->mapToSource(proxyIndex).row();
~~~

~~~cpp
// 跨线程
Q_DECLARE_METATYPE(FileInfo)              // 头文件
qRegisterMetaType<FileInfo>("FileInfo");  // main()
connect(worker, &ScannerWorker::fileFound,
        this, &MainWindow::onFileFound);  // 自动 Queued，槽里再 addFile
~~~

## Qt 5 → Qt 6 常见措辞速查

| 旧写法 / 常见误解 | 现状 |
|---|---|| QHeaderView::setResizeMode | 已弃用，用 setSectionResizeMode |
| QHeaderView::setStretchLastSection | 未弃用，仍是「最后一列拉伸」专用；全列均匀拉伸用 setSectionResizeMode(QHeaderView::Stretch) |
| 点了表头会排序（只设 setSortingEnabled） | QAbstractTableModel 默认 sort() 不重排数据；排序用 QSortFilterProxyModel |
| 没设 NoEditTriggers 就一定能编辑 | 编辑能力由 flags() 的 ItemIsEditable 与 editTriggers 共同决定 |

## 一句话总结

Qt Model/View 不是「把数据丢给控件显示」，而是：

~~~text
数据归 Model，显示归 View，绘制按需提问
rowCount / columnCount 报数量，data / headerData 报内容
所有修改必须 begin/end 或 dataChanged 显式通知
表格是扁平的：parent 永远无效，0 起始，行前列后
排序过滤走代理，永不改写源数据
Worker 只发信号，改 Model 永远回主线程
~~~