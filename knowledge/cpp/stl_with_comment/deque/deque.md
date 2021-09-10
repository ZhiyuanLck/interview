# deque

## 迭代器

通过四个指针进行迭代

```cpp
_Elt_pointer _M_cur;    // 指向当前元素
_Elt_pointer _M_first;  // 指向当前缓冲区的第一个元素
_Elt_pointer _M_last;   // 指向当前缓冲区最后一个元素
_Map_pointer _M_node;   // 指向当前缓冲区
```

当迭代器超出当前缓冲区的范围时，更新`_M_first, _M_last, _M_node`

```cpp
void _M_set_node(_Map_pointer __new_node) _GLIBCXX_NOEXCEPT {
  _M_node = __new_node;
  _M_first = *__new_node;
  _M_last = _M_first + difference_type(_S_buffer_size());
}
```

自增和自减操作首先判断是否超出缓冲区需要更新指针

```cpp
_Self& operator++() _GLIBCXX_NOEXCEPT {
  ++_M_cur;
  if (_M_cur == _M_last) {
    _M_set_node(_M_node + 1);
    _M_cur = _M_first;
  }
  return *this;
}

_Self& operator--() _GLIBCXX_NOEXCEPT {
  if (_M_cur == _M_first) {
    _M_set_node(_M_node - 1);
    _M_cur = _M_last;
  }
  --_M_cur;
  return *this;
}
```

迭代器需要移动n位时，先计算相对于`_M_first`的绝对偏移，判断迭代器移动后是否超出缓冲区。+, -, -=, []操作符都是基于+=操作符

```cpp
_Self& operator+=(difference_type __n) _GLIBCXX_NOEXCEPT {
  const difference_type __offset = __n + (_M_cur - _M_first);
  // 偏移值小于缓冲区大小
  if (__offset >= 0 && __offset < difference_type(_S_buffer_size()))
    _M_cur += __n;
  // 偏移值大于缓冲区大小
  else {
    // 超出当前缓冲区，计算map偏移
    const difference_type __node_offset = __offset > 0
      ? __offset / difference_type(_S_buffer_size())
      : -difference_type((-__offset - 1) // offset < 0, -__offset - 1为移动后反过来的从0开始的下标
        / _S_buffer_size()) - 1; // offset小于0时map至少向前移动一个node
    _M_set_node(_M_node + __node_offset);
    // 类似取余操作
    // 正向偏移用offset减去偏移的缓冲区大小
    // 反向偏移则用偏移的缓冲区大小减去offset
    _M_cur = _M_first + (__offset - __node_offset
    * difference_type(_S_buffer_size()));
  }
  return *this;
}
```

迭代器之间的距离计算

```cpp
template<typename _Tp, typename _Ref, typename _Ptr>
inline typename _Deque_iterator<_Tp, _Ref, _Ptr>::difference_type
operator-(const _Deque_iterator<_Tp, _Ref, _Ptr>& __x,
const _Deque_iterator<_Tp, _Ref, _Ptr>& __y) _GLIBCXX_NOEXCEPT {
  return typename _Deque_iterator<_Tp, _Ref, _Ptr>::difference_type
    (_Deque_iterator<_Tp, _Ref, _Ptr>::_S_buffer_size())
    * (__x._M_node - __y._M_node - 1) + (__x._M_cur - __x._M_first)
    + (__y._M_last - __y._M_cur);
}
```

## 内存管理

`deque`通过`_Deque_base`类来管理内存。

`_Deque_base`中使用`_Deque_impl`封装了Deque的实现，其中记录了指向map的指针，map的大小以及指向开头和结尾的迭代器

```cpp
_Map_pointer _M_map;
size_t _M_map_size;
iterator _M_start;
iterator _M_finish;
```

初始化map时，计算所需要的节点数量，确定`_M_map_size`

```cpp
this->_M_impl._M_map_size = std::max((size_t) _S_initial_map_size, size_t(__num_nodes + 2));
```

然后从中间开始为节点分配空间
