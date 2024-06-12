---
title: Scaling Scrollable Lists in Qt
categories: [programming]
tags: [cpp, qt]
last_modified_at: 2024-06-11-T12:00:00-05:00
---

In a recent project, one of the tasks my team was assigned was implementing a
scrollable list area consisting of a preview image, title, and description text.
The project was in C++ using the Qt library. In previous features we wrote,
we decided that it was more worthwhile to create a custom widget to represent
each item in the scrollable region. This was fine for our previous use cases, as
the number of items in the scrollable region were very small, never to exceed 20.

This time we had to greatly consider the scalability of our design. Instead of populating
20 items in a list, we had to support tens of thousands! Our previous approach did not scale
at all, so it was time to go to the drawing board and see what other options we had.


## The Initial Design

The initial design we had used in the past was nice. We followed a Model-View approach
consisting of the following components:

- A struct representing the data to be displayed.

```cpp
struct Item {
    QString title;
    QString description;
};

Q_DECLARE_METATYPE(Item)
```

- A custom widget to display each item in the list

```cpp
class ItemWidget : public QWidget {
public:
    explicit ItemWidget(const Item& item, QWidget* parent = nullptr) : QWidget(parent) {
        const auto layout = new QHBoxLayout(this);
        const auto titleLabel = new QLabel(item.title, this);
        const auto descriptionLabel = new QLabel(item.description, this);
        layout->addWidget(titleLabel);
        layout->addWidget(descriptionLabel);
    }
};
```

- A view to display each `ItemWidget`

```cpp
class WidgetView : public QWidget {
    Q_OBJECT

public:
    explicit WidgetView(WidgetModel* model, QWidget* parent = nullptr) 
        : QWidget(parent), m_model(model) {
        
        m_layout = new QVBoxLayout(this);
        setLayout(m_layout);

        // Connect the model's signal to the view's update method
        connect(model, &WidgetModel::itemsChanged, this, &WidgetView::updateView);
    }

public slots:
    void updateView(const QList<Item>& items) {
        // Clear the current widgets
        clearLayout(m_layout);

        // Add new widgets based on the model's items
        for (const auto& item : items) {
            m_layout->addWidget(new ItemWidget(item));
        }
    }

private:
    void clearLayout(QLayout* layout) {
        while (QLayoutItem* item = layout->takeAt(0)) {
            delete item->widget();
            delete item;
        }
    }

    QVBoxLayout* m_layout{};
    WidgetModel* m_model{};
};
```

- And the model to notify the view of any changes to the backend

```cpp
class WidgetModel : public QObject {
    Q_OBJECT
public:
    explicit WidgetModel(QObject* parent = nullptr) : QObject(parent) {}

    void addItems(int count) {
        for (int i = 0; i < count; ++i) {
            m_items.append({
                QString("Title %1").arg(m_items.size() + 1),
                QString("Description %1").arg(m_items.size() + 1)});
        }
        emit itemsChanged(m_items);
    }

signals:
    void itemsChanged(const QList<Item>& items);

private:
    QList<Item> m_items;
};
```

The above design has a lot of strengths! Its implementation is high level,
making it easy to understand and use. Additionally, given that the individual
item is represented in a custom widget, it is also very easy to add additional
signals and slots when listening for events! It has one very large flaw though:
scalability. The moment the number of widgets begins to grow, it quickly
becomes apparent that this design is not efficient. Below is a sped up example
of loading in 5000 items after clicking a button. As you can see it takes a
surprisingly large amount of time to populate the list.

![Slow Load](/assets/img/blog/2024-06-11-scaling-lists-in-qt/slow-load.gif)

There are a few ways to help improve the performance of a widget-based model view
architecture.

#### Lazy Loading

Since we know there are only going to be N number of widgets in the view of the
user at a given point of time, we can simply generate only the visible widgets.
If the user scrolls, then we would append to the list the next
visible widgets. The con to this approach is we have to track where the user has scrolled.
Also, if the user is scrolling too quickly, they may experience lag while the widgets are
being generated.

#### Widget Pool

Since we know that creating widgets is an expensive operation, we can have a set number
of widgets already created. As the user scrolls, instead of creating additional widgets,
we would recycle the widgets and simply swap the data they display. This approach has
great scaling but requires a lot of extra set up. We need to track where the user has scrolled,
what data is currently visible, how to swap that data, and how to handle inserted data!
Unless you are dealing with very complex items in the list, this is a lot of overhead to
deal with. Additionally, you often lose a sense of "smooth scrolling" as it does take
effort to continuously swap data back and forth.

## Improved Design: Paint Events!

When painting items directly onto the view, there is less overhead compared to managing
individual widget instances for each item. This reduces memory consumption and improves
performance, especially for large datasets. In order to implement with the paint events,
we are going to slightly modify our initial design. Instead of having a Model-View architecture,
we are going to add an additional section called the Delegate. Qt refers to this design
pattern as the Model-View-Delegate, and it only requires slightly more code for a great
improvement of speed.

In the example below, we are only responsible for creating the model and the delegate.
For the view, we can leverage the `QListView` class.

- Implementing the Model

```cpp
class PaintModel : public QAbstractListModel {
    Q_OBJECT

public:
    explicit PaintModel(QObject* parent = nullptr) : QAbstractListModel(parent) {
        qRegisterMetaType<Item>("Item");
    }

    int rowCount(const QModelIndex& parent = QModelIndex()) const override {
        Q_UNUSED(parent);
        return m_items.size();
    }

    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        if (!index.isValid() || index.row() >= m_items.size())
            return QVariant();

        const auto& item = m_items.at(index.row());
        if (role == Qt::DisplayRole) {
            return QVariant::fromValue(item);
        }
        return QVariant();
    }

    void addItems(int count) {
        beginInsertRows(QModelIndex(), m_items.size(), m_items.size() + count - 1);
        for (int i = 0; i < count; ++i) {
            Item item;
            item.title = QString("Title %1").arg(m_items.size() + 1);
            item.description = QString("Description %1").arg(m_items.size() + 1);
            m_items.append(item);
        }
        endInsertRows();
    }

private:
    QList<Item> m_items;
};
```

- Implementing the Paint Delegate

```cpp
class PaintDelegate : public QAbstractItemDelegate {
    Q_OBJECT

public:
    explicit PaintDelegate(QObject* parent = nullptr) : QAbstractItemDelegate(parent) {}

    void paint(QPainter* painter, const QStyleOptionViewItem& option, const QModelIndex& index) const override {
        if (!index.isValid())
            return;

        // Cast the QVariant back to an Item struct
        Item item = index.data(Qt::DisplayRole).value<Item>();

        painter->save();

        // Draw background
        if (option.state & QStyle::State_Selected) {
            painter->fillRect(option.rect, option.palette.highlight());
        } else {
            painter->fillRect(option.rect, option.palette.base());
        }

        // Draw text
        painter->setPen(option.palette.color(QPalette::Text));
        painter->drawText(option.rect.adjusted(10, 5, -10, -25), Qt::AlignLeft | Qt::AlignVCenter, item.title);
        painter->drawText(option.rect.adjusted(10, 30, -10, -10), Qt::AlignLeft | Qt::AlignVCenter, item.description);

        painter->restore();
    }

    QSize sizeHint(const QStyleOptionViewItem& option, const QModelIndex& index) const override {
        Q_UNUSED(option);
        Q_UNUSED(index);
        return QSize(300, 70); // Adjust the size of each item as needed
    }
};
```

- Link to the `QListView`

```cpp
PaintModel paintModel;
const auto paintListView = new QListView;
paintListView->setModel(&paintModel);
paintListView->setItemDelegate(new PaintDelegate(paintListView));
```

It is fairly daunting to have to think about drawing items pixel by pixel in the paint event. It requires a lot of math,
and trial and error to get it right, especially if the widget to draw has a complicated design. That is perhaps one
of the greatest flaws of having to manually write paint events as a replacement for widgets. The speed increase
however is hardly comparable. Below is an example of me populating 100,000 new fields per click of the button. The paint event easily
scales into the millions, whereas the widget approach became shockingly slow when generating only 5,000 items.


![Scaling Paint Events](/assets/img/blog/2024-06-11-scaling-lists-in-qt/scaling-stress-test.gif)

## When to Use Each Approach

Use Widget-Based Approach When:
- Items require complex layouts or interactions.
- Each item has unique functionality or appearance.
- Memory and performance overhead are not significant concerns.

Use Paint-Based Approach When:
- Efficiency and scalability are paramount, especially for large datasets.
- Item appearance can be standardized or easily customized through painting.
- Interactivity requirements are modest and can be handled through other means (e.g., signals and slots).

For a complete example of the code in this blog, along with a side by side example of each approach,
please feel free to visit this GitHub Gist:
[https://gist.github.com/khurd21/f783bf70926846a3c79005bc4ff52f9a](https://gist.github.com/khurd21/f783bf70926846a3c79005bc4ff52f9a)
