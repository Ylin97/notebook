# C++迭代器小结

[toc]

## 迭代器适配器

标准库头文件`iterator`中定义了额外几种迭代器：插入迭代器、流迭代器、反向迭代器和移动迭代器。

### 1. 插入迭代器

它们被绑定到一个容器上，可用来向容器插入元素。插入器有三种类型，差异在于元素插入的位置：

  - `back_inserter`创建一个使用`push_back`的迭代器（尾插）。
  - `front_inserter`创建一个使用`push_front`的迭代器（头插）。
  - `inserter`创建一个使用`insert`的迭代器。

  ```cpp
  // 插入迭代器通常用于算法，如std::copy, std::transform等
  std::vector<int> vec1;
  std::vector<int> vec2;
  std::vector<int> vec3;
  std::list<int> lst = {1, 2, 3, 4, 5};

  std::copy(lst.cbegin(), lst.cend(), std::back_inserter(vec1));
  std::copy(lst.cbegin(), lst.cend(), std::front_inserter(vec2));
  std::copy(lst.cbegin(), lst.cend(), std::inserter(vec, vec3.begin()));

  // 也可以使用单独使用插入迭代器向容器中添加元素（一般没啥用，因为容器都自带了push和insert方法）
  auto it = std::back_inserter(vec1);
  *it = 6; // 将6插入到vec1的尾部
  ```

  > 对于插入迭代器`it`，由于`*it`、`it++`和`++it`不会对`it`做任何事情且都会返回`it`本身，所以`*it = v`、`it++ = v`和`++it = v`都等价于`it = v`，即往容器中插入元素`v`。建议写`*it = v`这种与其他迭代器使用一致的形式。

### 2. 流迭代器

它们被绑定到输入或者输出流上，可用来遍历所有关联的IO流。

流迭代器主要用于从输入流（如`std::cin`）读取数据或向输出流（如`std::cout`）写入数据。它们通常适用于以下场合：

- **从输入流读取数据**：
当你需要从标准输入（`std::cin`）或其他输入流中读取一系列数据时，可以使用流迭代器。这在处理用户输入或从文件中读取数据时非常有用。
- **向输出流写入数据**：
类似地，当你需要向标准输出（`std::cout`）或其他输出流写入一系列数据时，流迭代器可以简化这个过程。
- **与算法结合使用**：
流迭代器可以与C++标准库中的算法（如`std::sort`、`std::copy`等）结合使用，以实现对数据的高效处理。例如，你可以使用`std::copy`将数据从一个容器复制到输出流，或者从输入流复制到另一个容器。
- **处理连续数据流**：
在处理需要连续读取或写入大量数据的场景时，流迭代器提供了一种方便的方式来处理这些操作，而不需要手动管理内存或数据的逐个处理。
- **简化I/O操作**：
流迭代器可以简化I/O操作，使得代码更加简洁和易于维护。它们隐藏了底层的I/O操作细节，让你可以专注于算法逻辑。
- **与容器操作结合**：
当你需要将容器的内容输出到文件或从文件读取到容器时，流迭代器提供了一种直接的方法来实现这一点，而不需要手动编写循环来处理每个元素。

```cpp
// 使用流迭代器从标准输入中读取数据
std::vector<int> numbers1;
std::copy(std::istream_iterator<int>(std::cin), std::istream_iterator<int>(), std::back_inserter(numbers1));

// 或者可以用以下形式:
std::istream_iterator<int> in_iter(cin), eof; 
std::vector<int> numbers2(in_iter, eof);
// eof与std::istream_iterator<int>()一样，都是调用默认构造函数，它们的值与尾后迭代器相同 

// 使用流迭代器将容器数据批量输出数据
std::ostream_iterator<int> out_iter(std::cout, " ");
std::copy(numbers1.cbegin(), numbers1.cend(), out_iter);

std::copy(numbers2.cbegin(), numbers2.cend(), std::ostream_iterator<int>(std::cout, " "));

// 与算法结合
std::istream_iterator<int> in(cin), eof;
std::cout << accumulate(in, eof, 0) << endl;
```

> 1. 流迭代器使用`>>`来读取数据，使用`<<`来输出数据，所以可以为任何定义了输入运算符`>>`的类型创建`istream_iterator`，为任何定义了输出运算符`<<`的类型创建`ostream_iterator`。
>
> 2. 同插入迭代器，虽然`*out`、`++out`和`out++`不会对输出流迭代器产生任何影响，但是还是建议写成`*out = v`这种形式。
>
> 3. 流迭代器是一种方便的工具，它们使得某些类型的I/O操作更加简洁，但它们并不提供性能上的优势。在决定是否使用流迭代器时，应该考虑代码的可读性、可维护性和特定场景下的需求。

### 3.反向迭代器

它们向后而不是向前移动，除了`forward_list`之外的标准库容器都有反向迭代器。

不可能在流中反向移动，所以不能从一个`forward_list`或一个流迭代器创建反向迭代器。反向迭代器会反向遍历元素，如果要想正向遍历，则**必须调用`base`成员函数来获取对应的正向迭代器**。

```cpp
string line("FIRST,MIDDLE,LAST");

auto rcomma = find(line.crbegin(), line.cend(), ',');
cout << string(line.crbegin(), rcomma) << endl;; // 将打印 TSAL!
// 如果要正向打印，必须使用base成员函数。其中，rcomma和rcomma.base()指向不同的元素！
cout << string(rcomma.base(), line.cend()) << endl;
```

### 4.移动迭代器

它们通过移动语义来移动其中的元素，即当使用移动迭代器时，元素会被“移动”而不是被“复制”，这样可以提高效率，特别是在处理大型对象或资源密集型对象时。

移动迭代器的主要应用场景包括：

- **优化资源管理**：当容器需要释放其元素时，使用移动迭代器可以避免不必要的复制，从而节省资源和时间。
- **提高性能**：在处理临时对象或即将销毁的对象时，移动迭代器可以避免复制操作，直接转移对象的所有权。
- **算法实现**：在实现某些算法时，如`std::move`，移动迭代器提供了一种方式来遍历容器并移动其元素。

移动迭代器的使用通常涉及到`std::move_iterator`，这是一个模板类，它接受一个迭代器类型作为模板参数，并提供一个转换构造函数，该构造函数接受一个迭代器对象。使用`std::move_iterator`时，它会将传入的迭代器转换为一个移动迭代器。

```cpp
int main() {
  std::vector<int> src = {1, 2, 3, 4, 5};
  std::vector<int> dest;

  // 使用移动迭代器将src中的元素移动到dest中
  std::move(std::move_iterator<std::vector<int>::iterator>(src.begin()),
            std::move_iterator<std::vector<int>::iterator>(src.end()),
            std::back_inserter(dest));

  // 打印dest中的元素
  for (int num : dest) {
      std::cout << num << " ";
  }
  std::cout << std::endl;

  // src中的元素现在都是未定义的，因为它们已经被移动了
  return 0;
}
```

## 迭代器分类

以下是 C++ 标准库中五种主要迭代器的特点及常用算法总结：

------

### 1. 输入迭代器（Input Iterator）

#### 特点:

- **单向访问**：支持单次遍历，不能回退。
- **只读访问**：只能读取元素值，不能修改。
- **一次性读取**：每个元素只能被访问一次，多次访问可能会失效。
- **操作支持**：支持 `operator*` 和 `operator++`。
- 通常用来读取数据流。

#### 常用算法:

- 数据读取类算法：`std::copy`（输入范围）、`std::find`、`std::accumulate`。
- 典型用例：文件流（`std::istream_iterator`）。

------

### 2. 输出迭代器（Output Iterator）

#### 特点:

- **单向访问**：仅支持写操作，不能回退。
- **一次性写入**：每个位置只能被写入一次，不能再读取或多次写入。
- **只写访问**：不能读取已有值。
- **操作支持**：支持 `operator*` 和 `operator++`。

#### 常用算法:

- 数据写入类算法：`std::copy`（输出范围）、`std::transform`、`std::generate`。
- 典型用例：`std::back_inserter`，`std::ostream_iterator`。

------

### 3. 前向迭代器（Forward Iterator）

#### 特点:

- **单向多次访问**：可以从头到尾单向遍历，同一元素可以多次访问。
- **读写支持**：支持读取和修改元素值。
- **恒定时间访问**：对所有元素的访问开销相同。
- **操作支持**：支持 `operator*`、`operator++`。

#### 常用算法:

- 数据遍历和处理类算法：`std::replace`, `std::replace_copy`, `std::unique`。
- 典型用例：链表（`std::forward_list`）。

------

### 4. 双向迭代器（Bidirectional Iterator）

#### 特点:

- **双向访问**：可以向前和向后遍历。
- **读写支持**：支持读取和修改元素值。
- **操作支持**：除了 `operator++`，还支持 `operator--`。

#### 常用算法:

- 需要反向遍历的算法：`std::reverse`, `std::prev_permutation`, `std::rotate`。
- 典型用例：双向链表（`std::list`）。

------

### 5. 随机访问迭代器（Random Access Iterator）

#### 特点:

- **随机访问**：支持常数时间的任意位置访问。

- **读写支持**：支持读取和修改元素值。

- **操作支持**：
  - 支持随机偏移访问：`operator+`, `operator-`, `operator[]`。
  - 支持比较操作：`operator<`, `operator>` 等。

#### 常用算法:

- 需要索引或直接访问的算法：`std::sort`, `std::binary_search`, `std::nth_element`。
- 典型用例：数组和向量（`std::vector`, `std::deque`）。

------

### 总结对比表

| **特性/迭代器** | **输入迭代器** | **输出迭代器** | **前向迭代器** | **双向迭代器** | **随机访问迭代器** |
| --------------- | -------------- | -------------- | -------------- | -------------- | ------------------ |
| **遍历方向**    | 单向           | 单向           | 单向           | 双向           | 双向               |
| **多次访问**    | 否             | 否             | 是             | 是             | 是                 |
| **读/写**       | 只读           | 只写           | 读写           | 读写           | 读写               |
| **回退能力**    | 无             | 无             | 无             | 有             | 有                 |
| **随机访问**    | 无             | 无             | 无             | 无             | 有                 |
| **典型用例**    | 流、集合       | 插入器         | 链表           | 双向链表       | 数组、向量         |

------

### 使用技巧

1. 如果算法只需遍历和读取数据，输入迭代器已经足够。
2. 如果需要写入目标范围，输出迭代器适用。
3. 需要多次访问同一数据，使用前向迭代器及以上。
4. 需要反向遍历或回退操作，使用双向迭代器。
5. 需要索引访问或高效排序，使用随机访问迭代器。
