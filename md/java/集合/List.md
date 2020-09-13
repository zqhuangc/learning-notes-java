+ AbstractList-->ArrayList： 
  - 数据结构：数组
  - 容量：默认初始容量10 ，数组满后再添加扩容为1.5倍，
  - 是否同步：否
  - 底层原理：各项指定位置操作基本基于方法Arrays.copyOf->System.arraycopy，删改for循环equals判断
  - 增加，直接加在最后，超界扩容
+ Vector： 
  - 数据结构：与ArrayLiist相同
  - 是否同步：同步，
  - 底层原理：操作方法为 synchronized，与ArrayLiist相似
  - 容量：默认初始容量10，扩容长度若未设定新增长度，则扩为原来的2倍

+ AbstractSequentialList-->
LinkedList：
  - 数据结构：私有静态内部类Node，前后Node引用
  - 底层：查询指定位置根据与中间索引的比较从前后查找，有直接从前后添加的方法

ArrayList和LinkedList的分析，可以理解List的3个特性 
1. 是按顺序查找 
2. 允许存储项为空 
3. 允许多个存储项的值相等 


### 栈（先入后出）继承Vector
最大容量：最长Integer.MAX_VALUE  
栈是一种很重要的数据结构，如下一些操作都应用到了栈。 
* 符号匹配（左括号则入栈，右括号则出栈）
* 中缀表达式转换为后缀表达式
* 计算后缀表达式
* 实现函数的嵌套调用
* HTML和XML文件中的标签匹配
* 网页浏览器中已访问页面的历史记录