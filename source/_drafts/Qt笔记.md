---
abbrlink: ''
categories:
- - Qt
date: '2024-07-04T21:31:38.696432+08:00'
tags:
- Qt
- C++
title: Qt笔记
updated: '2025-11-14T20:10:00.867+08:00'
---
# Qt笔记

## 信号

### 自定义信号

```cpp
signals:
    void sig_addOne(int value);
```

> 使用signals声明，返回值是void

### emit----发送信号

```cpp
emit　sig_add(socre++);
```

Qt的子线程无法直接修改ui,需要发送信号到ui线程进行修改

## 非基础类型参数注册

```cpp
qRegisterMetaType<Score>("Score");
qRegisterMetaType<string>("string");
```

## 槽函数的参数和信号参数的关系

> Qt槽函数的参数需要和信号的参数保持一致, 可以比信号的参数少, 但是不能顺序不同, 也不能比信号的参数多

## 信号重名如何处理

> 使用泛型解决信号重载问题

```cpp
connect(ui->comboBox, QOverload<int>::of(&QComboBox::currentIndexChanged),
            this, &Widget::onIndex);
connect(ui->comboBox, QOverload<const QString&>::of(&QComboBox::currentIndexChanged),
            this, &Widget::onIndexStr);
```

## 信号和槽函数关系

> - 信号是由对象在某个特定事件发生时发出的消息。信号本身并不执行任何操作，它仅仅是表示某种状态的变化或事件的发生。例如，当一个按钮被点击时，它会发出一个`clicked()`信号
> - 槽是一个函数，可以接收并处理信号。当一个信号被发出时，与该信号连接的槽函数就会被调用。槽函数可以是任何成员函数、全局函数或lambda表达式

## 复选框状态

> 未选中  0
>
> 半选中 1
>
> 选中 2
