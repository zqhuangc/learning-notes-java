### BF 和 FK



### BM



### KMP

 核心原理是利用一个“部分匹配表”

```java
移动位数 = 已匹配的字符数 - 对应的部分匹配值（最少移动一位）
```
"部分匹配值"就是"前缀"和"后缀"的最长的共有元素的长度。以"ABCDABD"为例，
"前缀"和"后缀"。  
 * "前缀"指除了最后一个字符以外，一个字符串的全部头部组合；  
 * "后缀"指除了第一个字符以外，一个字符串的全部尾部组合。



```
public static int kmp(char[] a, int n, char[] b, int m) {
	int[] next = getNexts(b, m);
	int j = 0;
	for (int i = 0; i < n; ++i) {
		while (j > 0 && a[i] != b[j]) { // 一直找到a[i]和b[j]
			j = next[j - 1] + 1;
		}
		if (a[i] == b[j]) {
			++j;
		}
		if (j == m) { // 找到匹配模式串的了
			return i - m + 1;
		}
	}
	return -1;
}
```

