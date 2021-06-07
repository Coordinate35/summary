|作者|版本号|时间|
|:-|:-|:-|
|Coordinate35| v1.0.0| 2017-06-15|

# LeetCode 88,91,76题解

此题解纯属个人观点，如有错误，欢迎指出。

## (LeetCode 88.Merge Sorted Array)

### 题目

Given two sorted integer arrays *nums1* and *nums2*, merge *nums2* into *nums1* as one sorted array.

**Note:**
You may assume that *nums1* has enough space (size that is greater or equal to *m* + *n*) to hold additional elements from *nums2*. The number of elements initialized in *nums1* and *nums2* are *m* and *n* respectively.

大意就是给定两个已经排好序的数组arr1,arr2，保证第一个数组的可用空间能够装下两个数组所有的现有有效元素。目标是让你将两个数组合并成一个排好序的数组，并把所有的元素放在第一个数组里面。

### 题解

#### 方法一

一个最直接能想到的办法就是，开辟一个和arr1元素数量一样长的数组arr3,先把arr1中的所有元素拷贝到arr3,同时清空arr1。然后两个指针p2, p3分别指向arr2,arr3的起始位置。每次比较p2,p3指向元素的大小，取小的那个压入arr1的末尾。直到p2, p3中有一个指针到达末尾了，就让另一个还没结束的数组中剩余的元素全部进入arr1。这种方法空间复杂度是O(n), 时间复杂度是O(n + m);

伪代码：

```pascal
algorithm merge(arr1, arr2) is
	for i <- 1 to arr1_length do 
		arr3 <- arr1[i];
	p1 <- p2 <- p3 <- 1;
	
	while ((p2 ≠ arr2_length) and (p3 ≠ arr3_length)) do 
		if (arr2[p2] > arr3[p3]) then
			arr1[p1] <- arr3[p3];
			p3 <- p3 + 1;
		else 
			arr1[p1] <- arr2[p2];
			p2 <- p2 + 1;
		p1 <- p1 + 1;
		
	if (p2 ≠ arr2_length) then 
		rest_arr <- arr2;
		rest_p <- p2;
		rest_length <- arr2_length
	else 
		rest_arr <- arr3;
		rest_p <- p3;
		rest_length <- arr3_length
	while (rest_p ≠ rest_length) do
		arr1[p1] <- rest_arr[rest_p];
		rest_p <- rest_p + 1;
		p1 <- p1 + 1;
		
	return arr1;
```

#### 方法二

上面这种做法，需要耗费O(n)的空间，有没有更高效的办法呢？有！

试想数组的优势是什么？可以随机访问。同时，目前给定的数据是已经排好序的。那我们能不能，不从数组的开始进行合并， 而是从数组的末尾开始合并呢？

做法就是，p1指向 arr1的最后一个元素，p2指向arr2的最后一个元素。p指向arr1空间末尾。每次从p1,p2中取比较大的那个元素压到arr1[p]的位置。这样，我们的空间复杂度就是O(1),时间复杂度还是O(m+n)

传进来的arr1的空间已经保证了可以容纳两个数组的所有元素，最坏的情况下，即使arr2的数据全部进入arr的末尾，也不会污染arr1的数据，所以这种做法是可行的。

```pascal
algorithm merge(arr1, arr2) is 
	p1 <- arr1_length;
	p2 <- arr2_length;
	p <- arr1_length + arr2_length;
	
	while ((arr1[p1] > 0) and (arr2[p2] > 0)) do
		if (arr1[p1] > arr2[p2]) then
			arr1[p] <- arr1[p1];
			p1 <- p1 - 1;
		else 
			arr1[p] <- arr2[p2]
			p2 <- p2 - 1;
		p <- p - 1;
		
	if (p1 > 0) then 
		rest_p <- p1;
		rest_arr <- arr1;
	else 
		rest_p <- p2;
		rest_arr <- arr2;
	while (p > 0) do
		arr1[p] <- rest_arr[rest_p]
		p <- p - 1;
		rest_p <- rest_p - 1;
		
	return arr1;
```

## (LeetCode 91.Decode Ways)

### 题目

A message containing letters from `A-Z` is being encoded to numbers using the following mapping:

```
'A' -> 1
'B' -> 2
...
'Z' -> 26

```

Given an encoded message containing digits, determine the total number of ways to decode it.

For example,
Given encoded message `"12"`, it could be decoded as `"AB"` (1 2) or `"L"` (12).

The number of ways decoding `"12"` is 2.

大意就是给定了字母A-Z和数字1-26的对应法则，给出一串数字，算出有多少种可能的转换成字母串的方式。

### 题解

定义状态变量 decode_ways[i] 表示（那我们实际要求的就是decode_ways[s的长度],decode_ways[i]的初始值都是0），s[1..i] 这个数字串转换成字母串的方式的数量。观察这个题目，数字会变转换成字母有两种可能：

1. 一个数字转换成一个字母，比如"2"转换成"B"
2. 两个数字一起转换成字母，比如"26"转换成"Z"

按照所求的s的长度来划分阶段，那么很明显，当前阶段所求的decode_ways[i]只和上一阶段的decode_ways[i - 2]和decode_ways[i - 1]有关。

再来定义一下合法数字(valid number)的概念:

> 合法数字村在的形式是一个字符串,叫它s_num。s_num可能有一位，也可能有两位。
>
> 1. 若只有一位：只要s_num[1]不为'0'就是合法的
> 2. 若有两位：只要s_num[1]不为'0'，且把s_num转化成数字num之后，满足 1 <= num <= 26

所以状态转移方程就很明显了：

```
decode_ways[i] = decode_ways[i - 1] * (s[i] is valid number ? 1 : 0)
				+ decode_ways[i - 2] * (s[i - 1 .. i] is valid number ? 1 : 0)
```

再来看边界条件：

1. 如果s[1]存在:

   1. 如果s[1]是合法数字，则decode_ways[1] = 1，否则decode_ways[1] = 0;

   如果s[1]不存在：直接返回 0；

2. 如果s[2]存在：

   1. 如果s[2]是个合法数字，则decode_ways[2] = decode_ways[1]；
   2. 如果s[1..2]是个合法数字，则decode_ways[2]自增1；

   如果s[2]不存在：直接返回decode_ways[1];

确定好边界，一直算到decode_ways[s的长度]返回就行。

伪代码：

```pascal
algorithm num_decodings(s) is
	if s[1] exists then
		if s[1] is valid number then
			decode_ways[1] <- 1;
		else 
			decode_ways[1] <- 0;
	else 
		return 0;
	if s[2] exists then
		if s[2] is valid number then
			decode_ways[2] <- decode_ways[1];
		if s[1..2] is valid number then
			decode_ways[2] <- decode_ways[2] + 1;
	else
		return decode_ways[1];
	
	for i <- 3 to s_length do
		decode_ways[i] <- decode_ways[i - 1] * (s[i] is valid number ? 1 : 0) 
						+ decode_ways[i - 2] * (s[i - 1 .. i] is valid number ? 1 : 0);
						
	return decode_ways[s_length];
```

## (LeetCode 76.Minimum Window Substring)

### 题目

Given a string S and a string T, find the minimum window in S which will contain all the characters in T in complexity O(n).

For example,
**S** = `"ADOBECODEBANC"`
**T** = `"ABC"`

Minimum window is `"BANC"`.

**Note:**
If there is no such window in S that covers all characters in T, return the empty string `""`.

If there are multiple such windows, you are guaranteed that there will always be only one unique minimum window in S.

大意就是，给定两个字符串S,T。在S中找到一个最短的字符串（窗口），使得这个窗口，包含T字符串中的所有出现的字符，包括数量。要求复杂度是O(n),数据保证最短的窗口在S中只出现一次。

### 题解

用 window_start 和 window_end 来标记窗口的前后边界。一开始，window_start 和 window_end 都为 1，表示从字符串的最开始开始。然后 window_end 不断的往后移，到达某个位置的时候，出现了第一个合法窗口。然后，window_start 开始往右移，直到缩小到一个最小的合法窗口。记录下窗口的边界值 min_window_start 和 min_window_end。然后 window_start 往后移一位，使窗口变得不合法。接着再将 min_window_end 往后滑，直到出现一个合法的窗口，然后将 window_start 往右收缩窗口达到一个最小合法窗口。如果当前窗口比之前得到的最小窗口还要小，就更新最小窗口。如此循环，直到最终得到的临时最小合法窗口的的window_end到达S的末尾。

判断窗口是否合法的办法是：我们创建两个字典，expected_char 和 appeared_char。他们每个元素的初始值都是0。

expected_char 的键是字母，值是该字母在T中出现的次数。

appeared_char 的键是字母，值是该字母在当前窗口中出现的次数。

同时使用一个变量 appeared_number 用来标记在字符串 T 中，出现在当前窗口的字符的个数(同一个字符出现两次按两个字符计算，但是如果这个字符出现在 T 中出现的次数小于在窗口中出现的次数，按在 T 中出现的次数计算)，也就是说，如果appeared_number和 T 的长度相等，就认为这个窗口是合法的。这样，就能快速的判断当前的窗口是否是合法的。

伪代码：

```pascal
algorithm min_window(s, t) is
	window_start <- 1;
	window_end <- 1;
	min_window_start <- 0;
	min_window_end <- s_length + 1;
	appeared_number <- 0;
	
	for i <- 1 to t_length do
		expected_char[t[i]] <- expected_char[t[i]] + 1;
		
	for window_end <- 1 to s_length do
		if expected_char[s[window_end]] ≠ 0 then
			if (appeared_char[s[window_end]] < expected_char[s[window_end]]) then
				appeared_number <- appeared_number + 1;
			appeared_char[s[window_end]] <- appeared_char[s[window_end]] + 1;
		
		if t_length = appeared_number then 
			while expected_char[s[window_start]] = 0 or (expected_char[s[window_start]] > 0 and expected_char[s[window_start]] < appeared[s[window_start]]) do
				if (expected_char[s[window_start]] ≠ 0) then
					appeared_char[s[window_start]] <- appeared_char[s[window_start]] - 1;
                window_start <- window_start + 1;
            if window_end - window_start < min_window_end - min_window_start then
            	min_window_start <- window_start;
            	min_window_end <- windwo_end;
            appeared_number <- appeared_number - 1;
            appeared_char[s[window_start]] <- appeared_char[s[window_start]] - 1;
            window_start <- window_start + 1;
            
    return s[min_window_start .. min_window_end];
```

由于 window_end 和 window_start 最多都只可能把 S 遍历一边，所以判断该算法的事件复杂度是线性的。空间复杂度和S、T长度没有关系，和S、T中字符的种类有关系，所以是O(1)