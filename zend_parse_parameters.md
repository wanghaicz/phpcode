# zend_parse_parameters详解

自动生成的PHP函数周围包含了一些注释，这些注释用于自动生成代码文档和vi、Emacs等编辑器的代码折叠。函数自身的定义使用了宏PHP_FUNCTION()，该宏可以生成一个适合于Zend引擎的函数原型。逻辑本身分成语义各部分，取得调用函数的参数和逻辑本身。为了获得函数传递的参数，可以使用`zend_parse_parameters()`API函数。下面是该函数的原型：
```
zend_parse_parameters(int num_args TSRMLS_DC, char *type_spec, …);
```

第一个参数是传递给函数的参数个数。通常的做法是传给它ZEND_NUM_ARGS()。(ZEND_NUM_ARGS()来表示对传入的参数“有多少要多少”)这是一个表示传递给函数参数总个数的宏。第二个参数是为了线程安全，总是传递TSRMLS_CC宏。第三个参数是一个字符串，指定了函数期望的参数类型，后面紧跟着需要随参数值更新的变量列表。因为PHP采用松散的变量定义和动态的类型判断，这样做就使得把不同类型的参数转化为期望的类型成为可能。例如，如果用户传递一个整数变量，可函数需要一个浮点数，那么`zend_parse_parameters()`就会自动地把整数转换为相应的浮点数。如果实际值无法转换成期望类型（比如整形到数组形），会触发一个警告。
下表列出了可能指定的类型。我们从完整性考虑也列出了一些没有讨论到的类型。
| 类型指定符 | 对应的C类型 | 描述 |
|:======:|:======|:======|
| l | long | 符号整数 |
| d | double | 浮点数 |
| s | char \*,int | 二进制字符串，长度 |
| b | zend_bool | 逻辑型（1或0） |
| r | zval * | 资源（文件指针，数据库连接等） |
| a | zval * | 联合数组 |
| o | zval * | 任何类型的对象 |
| O | zval * | 指定类型的对象。需要提供目标对象的类类型 |
| z | zval * | 无任何操作的zval |
 
为了容易地理解最后几个选项的含义，你需要知道zval是Zend引擎的值容器。无论这个变量是布尔型，字符串型或者其他任何类型，其信息总会包含在一个zval联合体中。下面的是或多或少在C中的zval, 以便我们能更好地理解接下来的代码。
```c
typedef union _zval {
  long lval;
  double dval;
  struct {
  char *val;
  int len;
  } str;
  HashTable *ht;
  zend_object_value obj;
} zval;
```
 
在我们的例子中，我们用基本类型调用`zend_parse_parameters()`，以本地C类型的方式取得函数参数的值，而不是用zval容器。为了让`zend_parse_parameters()`能够改变传递给它的参数的值，并返回这个改变值，需要传递一个引用。
```c
if (zend_parse_parameters(argc TSRMLS_CC, "sl", &str, &str_len, &n) == FAILURE)
return;
```

注意到自动生成的代码会检测函数的返回值FAILUER(成功即SUCCESS)来判断是否成功。如果没有成功则立即返回，并且由`zend_parse_parameters()`负责触发警告信息。因为函数打算接收一个字符串l和一个整数n，所以指定 ”sl” 作为其类型指示符。s需要两个参数，所以我们传递参考char * 和 int (str 和 str_len)给`zend_parse_parameters()`函数。无论什么时候，记得总是在代码中使用字符串长度str_len来确保函数工作在二进制安全的环境中。不要使用strlen()和strcpy()，除非你不介意函数在二进制字符串下不能工作。二进制字符串是包含有nulls的字符串。二进制格式包括图象文件，压缩文件，可执行文件和更多的其他文件。”l” 只需要一个参数，所以我们传递给它n的引用。
回到转换规则中来。下面三个对self_concat()函数的调用使str, str_len和n得到同样的值：
```c
self_concat("321", 5);
self_concat(321, "5");
self_concat("321", "5");
```
str 指向字符串"321"，str_len等于3，n等于5。
