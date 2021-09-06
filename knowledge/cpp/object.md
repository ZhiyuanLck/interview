# 对象语义

## 数据语义

### 虚基类

```cpp
class A {};
class B : virtual public A {};
class C : virtual public A {};
class D : public B, public C {};
```

A, B, C, D的大小分别为1, 8, 8, 16

A大小为1是因为编译器要保证任何两个对象都拥有独一无二的地址

### 静态成员

```cpp
class A {
  static int x;
};

int A::x = 1;

int main() {
  A a, *p = &a;
  a.x = 2;  // 1. 等同于A::x = 2
  p->x = 3; // 2. 等同于A::x = 3
}
```

上面两种存取方式的成本一致，汇编代码是一样的

```assembly
mov     DWORD PTR A::y[rip], 2
mov     DWORD PTR A::y[rip], 3
```

静态成员在class内部声明，在外部定义，具有内部链接，存储在data段，只在class作用域内可见，通过指针或对象存取静态成员的成本相同

### 非静态成员

```cpp
class A {
public:
  int x = 10;
};

int main() {
  A a;
  A *p = &a;
  a.x = 1;  // 1
  p->x = 2; // 2
}
```

在上面这个例子中，通过指针存取非静态成员成本略微大一点

```assembly
# a.x = 1 直接将1存入rbp-12指向的内存中
mov     DWORD PTR [rbp-12], 1

# p->x = 2 先将rbp-8指向的指针地址保存到rax中，然后将2复制到该地址指向的内存
mov     rax, QWORD PTR [rbp-8]
mov     DWORD PTR [rax], 2
```

多层次的继承由于alignment的原因会造成空间浪费。

多态造成的额外负担

- 虚表的引入
- 每个类对象有一个虚表指针vptr指向虚表
- 构造函数中为vptr设置初值
- 析构函数中重设vptr的值

多重继承虚表实际上只有一张，如果重写了第二个或更后面的基类的虚函数，会生成对应的thunk来修正this指针的地址然后调用对应的虚函数([查看VTT](https://stackoverflow.com/questions/6258559/what-is-the-vtt-for-a-class))

多重继承对非静态成员的存取不会有额外的成本，因为偏移是早就计算好的

class中如果有virtual base class subobject则将class分割为两部分，一个不变区域和一个共享区域。

- 不变区域拥有固定的offset，这部分的数据可以直接存取
- 共享区域即virtual base class subobject，这一部分的数据会因为每次的派生操作而有变化，只能间接存取
