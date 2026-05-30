# Qt Model/View架构深度解析与自定义委托实战

> 作者：蔡浩宇 | Qt开发笔记

## 引言

Qt的Model/View（模型/视图）架构是处理大规模数据展示的强大框架。不同于简单的"控件+数据"模式，Model/View将数据存储（Model）与数据展示（View）彻底分离，同一个数据集可以同时以表格、列表、树形等多种方式展示。本文将深入解析MVC架构原理，并通过一个自定义委托（Delegate）实战项目展示其强大能力。

---

## 1. Model/View架构概述

### 1.1 三层分离

```
┌──────────────────────────────────────────┐
│                Model/View架构            │
│                                          │
│  ┌─────────┐    ┌─────────┐             │
│  │  View   │◄──►│ Model   │  数据层     │
│  │(展示层) │    │(数据层) │             │
│  └────┬────┘    └─────────┘             │
│       │                                  │
│  ┌────▼────┐                             │
│  │Delegate │  渲染与编辑                 │
│  │(委托层) │                             │
│  └─────────┘                             │
└──────────────────────────────────────────┘

Model  → 提供数据（数据访问接口）
View   → 展示数据（布局与交互）
Delegate→ 控制每个单元格的渲染和编辑方式
```

### 1.2 核心优势

| 优势 | 说明 |
|------|------|
| 数据与视图分离 | 同一Model可用于多个View |
| 内存高效 | 只加载可见区域的数据（懒加载） |
| 灵活定制 | 通过Delegate自定义任意单元格的显示/编辑 |
| 可扩展 | 自定义Model连接数据库/网络/文件等任意数据源 |

---

## 2. 标准Model使用

### 2.1 QStringListModel（列表数据）

```cpp
#include <QStringListModel>
#include <QListView>

// 创建Model
QStringListModel *model = new QStringListModel(this);
model->setStringList({"Beijing", "Shanghai", "Guangzhou", "Shenzhen"});

// 创建View
QListView *listView = new QListView(this);
listView->setModel(model);

// 通过Model修改数据
model->insertRow(4);  // 在末尾插入空行
model->setData(model->index(4, 0), "Hangzhou");  // 设置数据

// 数据变更通知View自动刷新——无需手动update
```

### 2.2 QStandardItemModel（表格数据）

```cpp
QStandardItemModel *model = new QStandardItemModel(4, 3, this);
model->setHorizontalHeaderLabels({"Name", "Score", "Grade"});

// 设置表格数据
model->setItem(0, 0, new QStandardItem("Alice"));
model->setItem(0, 1, new QStandardItem("95"));
model->setItem(0, 2, new QStandardItem("A"));

model->setItem(1, 0, new QStandardItem("Bob"));
model->setItem(1, 1, new QStandardItem("82"));
model->setItem(1, 2, new QStandardItem("B"));

// 通过QTableView展示
QTableView *tableView = new QTableView(this);
tableView->setModel(model);
tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
tableView->setEditTriggers(QAbstractItemView::DoubleClicked);
tableView->horizontalHeader()->setStretchLastSection(true);
tableView->resizeColumnsToContents();
```

---

## 3. 自定义Model

### 3.1 继承QAbstractTableModel

```cpp
#ifndef STUDENTMODEL_H
#define STUDENTMODEL_H

#include <QAbstractTableModel>
#include <QVector>

struct Student {
    QString name;
    int     score;
    QString grade() const {
        if (score >= 90) return "A";
        if (score >= 80) return "B";
        if (score >= 70) return "C";
        if (score >= 60) return "D";
        return "F";
    }
};

class StudentModel : public QAbstractTableModel
{
    Q_OBJECT
public:
    enum Roles {
        NameRole = Qt::UserRole + 1,
        ScoreRole,
        GradeRole
    };

    explicit StudentModel(QObject *parent = nullptr);

    /* 必须重写的虚函数 */
    int rowCount(const QModelIndex &parent = QModelIndex()) const override;
    int columnCount(const QModelIndex &parent = QModelIndex()) const override;
    QVariant data(const QModelIndex &index, int role = Qt::DisplayRole) const override;
    QVariant headerData(int section, Qt::Orientation orientation,
                        int role = Qt::DisplayRole) const override;

    /* 可选：支持编辑 */
    Qt::ItemFlags flags(const QModelIndex &index) const override;
    bool setData(const QModelIndex &index, const QVariant &value,
                 int role = Qt::EditRole) override;

    /* 数据操作接口 */
    void addStudent(const Student &s);
    void removeStudent(int row);

private:
    QVector<Student> m_students;
};

#endif // STUDENTMODEL_H
```

### 3.2 Model实现

```cpp
#include "studentmodel.h"

StudentModel::StudentModel(QObject *parent)
    : QAbstractTableModel(parent)
{
    // 预置一些数据
    m_students = {
        {"Alice", 95}, {"Bob", 82}, {"Charlie", 71}, {"David", 58}
    };
}

int StudentModel::rowCount(const QModelIndex &) const
{
    return m_students.size();
}

int StudentModel::columnCount(const QModelIndex &) const
{
    return 3;  // Name, Score, Grade
}

QVariant StudentModel::data(const QModelIndex &index, int role) const
{
    if (!index.isValid() || role != Qt::DisplayRole)
        return QVariant();

    const Student &s = m_students[index.row()];
    switch (index.column()) {
    case 0: return s.name;
    case 1: return s.score;
    case 2: return s.grade();
    }
    return QVariant();
}

QVariant StudentModel::headerData(int section, Qt::Orientation orientation,
                                    int role) const
{
    if (role != Qt::DisplayRole || orientation != Qt::Horizontal)
        return QVariant();

    switch (section) {
    case 0: return "Name";
    case 1: return "Score";
    case 2: return "Grade";
    }
    return QVariant();
}

Qt::ItemFlags StudentModel::flags(const QModelIndex &index) const
{
    if (!index.isValid()) return Qt::NoItemFlags;
    return Qt::ItemIsEnabled | Qt::ItemIsSelectable | Qt::ItemIsEditable;
}

bool StudentModel::setData(const QModelIndex &index, const QVariant &value, int role)
{
    if (!index.isValid() || role != Qt::EditRole) return false;

    Student &s = m_students[index.row()];
    switch (index.column()) {
    case 0: s.name = value.toString(); break;
    case 1: s.score = value.toInt(); break;
    case 2: return false;  // Grade是计算字段，不可编辑
    }

    emit dataChanged(index, index, {role});  // 通知View数据已变更
    return true;
}

void StudentModel::addStudent(const Student &s)
{
    beginInsertRows(QModelIndex(), m_students.size(), m_students.size());
    m_students.append(s);
    endInsertRows();
}

void StudentModel::removeStudent(int row)
{
    beginRemoveRows(QModelIndex(), row, row);
    m_students.remove(row);
    endRemoveRows();
}
```

> **关键**：增删行操作必须调用 `beginInsertRows/endInsertRows` 和 `beginRemoveRows/endRemoveRows`，否则View无法正确更新。

---

## 4. 自定义委托（Delegate）

### 4.1 进度条委托

```cpp
#ifndef PROGRESSDELEGATE_H
#define PROGRESSDELEGATE_H

#include <QStyledItemDelegate>
#include <QProgressBar>
#include <QApplication>

class ProgressDelegate : public QStyledItemDelegate
{
    Q_OBJECT
public:
    using QStyledItemDelegate::QStyledItemDelegate;

    /* 绘制单元格 */
    void paint(QPainter *painter, const QStyleOptionViewItem &option,
               const QModelIndex &index) const override
    {
        if (index.column() == 1)  // Score列显示为进度条
        {
            int score = index.data(Qt::DisplayRole).toInt();
            int percent = qBound(0, score, 100);

            QStyleOptionProgressBar progressBarOption;
            progressBarOption.rect = option.rect.adjusted(4, 4, -4, -4);
            progressBarOption.minimum = 0;
            progressBarOption.maximum = 100;
            progressBarOption.progress = percent;
            progressBarOption.text = QString::number(score);
            progressBarOption.textVisible = true;

            // 根据分数设置颜色
            if (score >= 90)
                progressBarOption.palette.setColor(QPalette::Highlight, QColor("#4CAF50"));
            else if (score >= 60)
                progressBarOption.palette.setColor(QPalette::Highlight, QColor("#FF9800"));
            else
                progressBarOption.palette.setColor(QPalette::Highlight, QColor("#F44336"));

            QApplication::style()->drawControl(QStyle::CE_ProgressBar,
                                               &progressBarOption, painter);
        }
        else
        {
            QStyledItemDelegate::paint(painter, option, index);
        }
    }
};

#endif // PROGRESSDELEGATE_H
```

### 4.2 使用自定义委托

```cpp
QTableView *view = new QTableView(this);
StudentModel *model = new StudentModel(this);

view->setModel(model);
view->setItemDelegateForColumn(1, new ProgressDelegate(this));  // 第1列用进度条
view->setItemDelegateForColumn(2, new GradeDelegate(this));     // 第2列用颜色标签
```

---

## 5. Model/View 性能优化

| 技术要点 | 方法 | 适用场景 |
|----------|------|----------|
| 懒加载 | 只提供可见行的数据 | 10万+行数据 |
| 批量更新 | beginResetModel/endResetModel | 全量刷新 |
| 局部更新 | dataChanged信号指定范围 | 单行/单列更新 |
| 代理过滤 | QSortFilterProxyModel | 排序/过滤 |
| fetchMore | 分批加载数据 | 数据库/网络数据 |

```cpp
/* 使用QSortFilterProxyModel实现过滤 */
QSortFilterProxyModel *proxy = new QSortFilterProxyModel(this);
proxy->setSourceModel(model);
proxy->setFilterKeyColumn(0);  // 按Name列过滤
proxy->setFilterCaseSensitivity(Qt::CaseInsensitive);

view->setModel(proxy);

// 过滤操作
proxy->setFilterFixedString("ali");  // 只显示包含"ali"的行
```

---

## 6. 总结

Model/View架构是Qt处理数据驱动型UI的核心框架：

- **Model**负责数据存储与访问——继承QAbstractTableModel/QAbstractListModel
- **View**负责布局与交互——QTableView/QListView/QTreeView
- **Delegate**负责单元格渲染与编辑——继承QStyledItemDelegate

理解这套架构，是开发专业级Qt应用程序的必经之路。无论是数据管理系统、监控仪表盘还是配置工具，Model/View都能提供优雅的数据展示方案。

---

*本文首发于个人技术博客，欢迎交流讨论。*
