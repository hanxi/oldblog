---
layout: post
title: "[原创]构造函数的语意"
tagline: "The Semantics of Constructors"
description: "编译器在何时自动合成构造函数？成员初始化，NRV"
tags: ["C++", "对象模型", "读书笔记", " NRV "]
categories: ["技术"]
---
{% include JB/setup %}

### 默认构造函数
默认构造函数在编译器需要的时候被编译器产生出来。

编译器会自动生成的四种有用的默认构造函数：
#### 带有default constructor 的member class object（组合结构）
如果一个类没有任何构造函数，而它含有一个对象成员，且该成员有默认构造函数，那么编译器需要为该类合成一个默认构造函数，不过合成操作只在构造函数真正需要被调用的时候才会发生。
被合成的默认构造函数只满足编译器的需要，而不是程序的需要。如：

<pre class="prettyprint lang-cpp">
class Foo {public: Foo(), Foo(int) ...};
class Bar {public: Foo foo; char* str;} // 组合

void foo_bar()
{
    Bar bar; // Bar::foo成员必须在此处初始化
    if (str) {} ...
}
</pre>

被合成的默认构造函数看起来可能时这样的：
<pre class="prettyprint lang-cpp">
inline
Bar::Bar()
{
    //C++伪代码
    foo.Foo:Foo();
}
</pre>
生成的代码中没有提供str的初始化操作，需要程序员自己完成。如果程序员定义构造函数如下：
<pre class="prettyprint lang-cpp">
Bar::Bar() { str=0;}
</pre>
编译的做法是：如果类内含一个或多个对象成员，那么类的每一个构造函数必须调用每一个对象成员的默认构造函数。编译器会扩张已存在的构造函数，在user code之前安插一些代码。上面的构造函数会扩张成下面的样子：
<pre class="prettyprint lang-cpp">
Bar::Bar() {
    foo.Foo:Foo();
    str=0;
}
</pre>
如果有多个对象成员都要求构造初始化操作，C++语言要求以对象在类中的声明次序来调用各个构造函数。

#### 带有default constructor 的base class
类似于第一种组合情形。如果基类含有默认构造函数，而派生类没有，则编译器会自动为派生类合成一个默认构造函数用于完成对基类的初始化。
如果派生自多个基类，则按声明次序依次调用。
如果派生类中包含其他类对象成员，则在所有的base class Constructor 都被调用之后调用类对象成员的构造函数。
#### 带有一个virtual function 的 class
编译器会做下面两个扩张：

1. 产生一个vtbl，里面存放类的所有虚函数的地址。

2. 在每个类对象中，添加一个额外的vptr指针，内含一个相关的类vtbl的地址。

#### 带有一个virtual base class 的 class
virtual base class 的实现法在不同的编译器之间有极大差异，然而，每种实现法的共通点是：必须让virtual base class在其每一个派生类对象中的位置能在执行期准备妥当。
一般在派生类对象中安插一个指向virtual base classes的指针，所有经由引用或指针来存取一个virtual base class 的操作都可以通过相关指针完成。

总结：
在合成的默认构造函数中，只有base class subobjects和member class objects会被初始化，所以其他的非静态成员数据，如整数，整数指针，整数数组等等都不会被初始化，这些初始化操作需要程序员自己完成。

### 拷贝构造函数
有三种情况会以一个对象的内容作为另一个对象的初值：

1. 对一个对象做明确的初始化操作

2. 当对象被当做参数交给某个函数时

3. 当函数传回一个类对象时

不使用逐位拷贝的情况
#### 类中含有类对象成员，而该成员的类声明有一个拷贝构造函数
<pre class="prettyprint lang-cpp">
class Word {
public:
    Word(const String& );
    ~Word();
    // ...
private:
    int cnt;
    String str;
};

class String {
public:
    String(const char*);
    String(const String&);
    ~String();
    // ...
};

Word noun("book");
void foo()
{
    Word verb = noun;
    // ...
}
</pre>
在这中情况下，编译器会合成出一个拷贝构造函数以便调用member class String object的拷贝构造函数：
<pre class="prettyprint lang-cpp">
inline Word:Word( const Word& wd)
{
    str.String::String(wd.str);
    cnt = wd.cnt;
}
</pre>
在这被合成出来的拷贝构造函数中，如整数，指针，数组等等的nonclass members 也都会被复制。

#### 类继承子一个基类而基类存在一个拷贝构造函数
这里不举例了，和第一种情况类似。

#### 类声明了一个或多个虚函数
<pre class="prettyprint lang-cpp">
class ZooAnimal {
public:
    ZooAnimal();
    virtual ~ZooAnimal();

    virtual void animate();
    // ..
private:
    // data ...
};

class Bear: public ZooAnimal {
public:
    Bear();
    void animate();
    virtual void dance();
    // ...
private:
    // data ...
};

Bear yogi;
ZooAnimal franny = yogi; // 切割行为
</pre>
当一个基类对象以派生类的对象做初始化操作时，其bptr复制操作也必须保证安全，合成出来的 ZooAnimal 拷贝构造函数会明确设定对象的vptr指向 ZooAnimal 的 vtbl。


#### 类派生子一个继承串链，其中有一个或多个虚基类
<pre class="prettyprint lang-cpp">
class Raccoon : public virtual ZooAnimal {
};
class RedPanda : public Raccoon {
};

Raccoon rocky;
Raccoon little_critter = rocky; // 简单的逐位拷贝足以完成

RedPanda red;
Raccoon little_critter = red; // 需要编译器完成vbcPtr(虚基类指针)的初始化
</pre>
为了正确的完成 little_critter 的初值设定，编译器必须合成一个拷贝构造函数，安插一些代码以设定vbcPtr的处置。

### 程序转换

#### 明确的初始化操作
<pre class="prettyprint lang-cpp">
X x0;
void foo_bar() {
    X x1(x0);
    X x2=x0;
    X x3 = X(x0);
}

// 可能的程序转换
void foo_bar() {
    X x1;
    X x2;
    X x3;

    x1.X::X(x0);
    x2.X::X(x0);
    x3.X::X(x0);
}
</pre>
####参数的初始化
<pre class="prettyprint lang-cpp">
void foo(X x0);
X xx;
foo(xx);

// 可能转换
void foo(X& x0);
X _tmp0;
_tmp0.X::X(xx);
foo(_tmp0);
</pre>

#### 返回值的初始化
<pre class="prettyprint lang-cpp">
X bar()
{
    X xx;
    // ...
    return xx;
}
X xx = bar();

bar().memfunc();

X ( *pf )();
pf = bar;

// 有可能的转换
void bar(X& _ret)
{
    X xx;
    xx.X::X();
    // ...
    _ret.X::X(xx);
    return;
}
X xx;
bar(xx);

X _tmp0;
(bar(_tmp0),_tmp0).memfunc();

void ( *pf ) (X&);
pf=bar;

</pre>

#### 在使用者层面做优化
<pre class="prettyprint lang-cpp">
X bar(const T &y, const T &z)
{
    X xx;
    // 以y和z来处理xx
    return xx;
}
</pre>
定义另一个构造函数，可以直接计算xx的值：
<pre class="prettyprint lang-cpp">
X bar(const T &y, const T &z)
{
    return X(y,z);
}
</pre>
C++伪代码
<pre class="prettyprint lang-cpp">
void bar (X & _ret,const T &y, const T &z) {
    _ret.X::X(y,z);
    return;
}
</pre>

#### 在编译器层面做优化(NRV-named return value)
<pre class="prettyprint lang-cpp">
X bar()
{
    X xx;
    // 处理xx
    return xx;
}

void bar(X & _ret)
{
    _ret.X:X();
    // 直接处理_ret
    return;
}
</pre>

总结：当一个函数以传值（by value）的方式返回一个类对象时，而该class有一个拷贝构造函数（不论时合成的还是明确定义的）时，这将导致深奥的程序转化（不论时函数定义还是使用），此外编译器也将拷贝构造函数的调用做优化，一个额外的第一参数（数值被直接存放其中）取代NRV。


### 成员初始化列表
什么情况下必须使用成员初始化列表：

1. 当初始化一个引用成员时。

2. 当初始化一个const成员时。

3. 当调用一个基类的构造函数，而它拥有一组参数时。

4. 当调用一个成员对象的构造函数，而它拥有一组参数时。

编译器生成的初始化代码的次序不是按照成员初始化列表的顺序，而是有类中成员声明次序决定。编译器生成的初始化代码会放在user code之前。生成的基类构造函数的调用代码在生成的成员初始化代码之前。
<pre class="prettyprint lang-cpp">
class FooBar:public X {
    int _fval;
public:
    int fval() {return _fval);
    FooBar(int val):_fval(val),X(fval()){}
};
// 会生成如下代码：
FooBar::FooBar(int val) {
    X::X(this, this->fval());
    _fval = val;
}

// 想要得到正确的结果，应该这样做：
class FooBar:public X {
    int _fval;
public:
    int fval() {return _fval);
    FooBar(int val):_fval(val),X(val){}
};
// 会生成如下代码：
FooBar::FooBar(int val) {
    X::X(this,val);
    _fval = val;
}
</pre>

简单的说，编译器会对初始化列表一一处理并可能重新排序，以反映出成员的声明次序，它会安插一些代码到构造函数体内，并置于任何user code之前。

