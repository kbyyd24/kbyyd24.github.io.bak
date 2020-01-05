---
title: Sort Colors 解题思路
date: 2016-07-29
updated: 2016-07-29
---

# 题目

先把题目放上：

链接：[https://leetcode.com/problems/sort-colors](https://leetcode.com/problems/sort-colors/)

> Given an array with n objects colored red, white or blue, sort them so that objects of the same color are  adjacent, with the colors in the order red, white and blue.
>
> Here, we will use the integers 0, 1, and 2 to represent the color red, white, and blue respectively.
>
> ###### Note:
> You are not suppose to use the library's sort function for this problem.
>
> ###### Follow up:
> A rather straight forward solution is a two-pass algorithm using counting sort.
> First, iterate the array counting number of 0's, 1's, and 2's, then overwrite array with total number of 0's,  then 1's and followed by 2's.
>
> Could you come up with an one-pass algorithm using only constant space?

---

# 解题思路

拿到这个题目，第一个想到的就是遍历，计数，然后赋值。这个方法很容易想到，题目也给出了提示。

然而，能只用一次遍历就得到预期的数组吗？

当然可以。**按照要求，就是对数组进行遍历，找到`0`就放到前面去，找到`1`就放到中间，找到`2`就放到后面去。**有没有很眼熟？

没错，这就是一个**快速排序**，因为只有`0`，`1`，`2`这三种数，所以仅仅需要一次遍历就能完成，连赋值都变得简单了起来。 :smile: 

---

# 代码

```java
public class Solution {
	public void sortColors(int[] nums) {
		if (nums == null || nums.length < 2) return;
		int i = 0, j = 0, k = nums.length - 1;
		final int red = 0;
		final int white = 1;
		final int blue = 2;
		while (j <= k) {
			if (nums[j] < white) {
				nums[j++] = nums[i];
				nums[i++] = red;
			} else if (nums[j] > white) {
				nums[j] = nums[k];
				nums[k--] = blue;
			} else {
				j++;
			}
		}
	}
}

```
