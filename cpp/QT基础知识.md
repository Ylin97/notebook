# QT基础知识

## 自定义信号与槽

QT自定义信号与槽时，信号函数只需要声明，但槽函数需要定义：

发送者：

```cpp
// mybutton.h
class MyButton : public QWidget
{
    Q_OBJECT
public:
    explicit MyButton(QWidget *parent = nullptr);

protected:
    void mousePressEvent(QMouseEvent *event) override;
    void paintEvent(QPaintEvent *event) override;
    
private:
    QPixmap pic;

signals:
    void clicked();
};

// mybutton.cpp
MyButton::MyButton(QWidget *parent) : QWidget(parent)
{
    pic.load(":/file-open2.png");
    setFixedSize(pic.size());
}

void MyButton::mousePressEvent(QMouseEvent *event)
{
    pic.load(":/file-open3.png");
    update();
    emit clicked();
}


void MyButton::paintEvent(QPaintEvent *event)
{
    QPainter painter(this);
    painter.drawPixmap(rect(), pic);
}
```

接收者：

```cpp
// widget.h
QT_BEGIN_NAMESPACE
namespace Ui { class Widget; }
QT_END_NAMESPACE

class Widget : public QWidget
{
    Q_OBJECT

public:
    Widget(QWidget *parent = nullptr);
    ~Widget();

private slots:
    void on_myBtn_clicked();
    
private:
    Ui::Widget *ui;
};

// widget.cpp
Widget::Widget(QWidget *parent)
    : QWidget(parent)
    , ui(new Ui::Widget)
{
    ui->setupUi(this);
    connect(ui->myBtn, &MyButton::clicked, this, &Widget::on_myBtn_clicked);
}

Widget::~Widget()
{
    delete ui;
}

void Widget::on_myBtn_clicked()
{
    qDebug() <<　"myButton is clicked!";
}
```

## QT事件

事件处理过程：事件派发 → 事件过滤 → 事件分发 → 事件处理。

