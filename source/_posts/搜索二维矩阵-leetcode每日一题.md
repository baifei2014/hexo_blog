---
title: 搜索二维矩阵(leetcode每日一题)
tags: [leetcode, golang, 中等难度]
date: 2021-03-07 10:21:40
---

### 题目描述

编写一个高效的算法来判断 m x n 矩阵中，是否存在一个目标值。该矩阵具有如下特性：

- 每行中的整数从左到右按升序排列。
- 每行的第一个整数大于前一行的最后一个整数。

**示例1:**

{% asset_img mat1.jpeg 示例图1 %}

```
输入：matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 3
输出：true
```

**示例2:**

{% asset_img mat2.jpeg 示例图2 %}

```
输入：matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 13
输出：false
```

**提示:**
- m == matrix.length
- n == matrix[i].length
- 1 <= m, n <= 100
- -104 <= matrix[i][j], target <= 104

### 题目分析

根据题干中的描述，我们要在二维矩阵中搜索目标值，而且矩阵的值是有序的。这样就很简单了，可以转换思路，将二维矩阵中查找目标值转变为一维查找，然后利用熟悉的二分查找解决。

二维转为一维，那一维的头就是0，一维数组的尾就是二维矩阵元素的个数，也就是`m*n`。头尾相加除以2二分取中间位置得到整数mid，这时如何转换为二维矩阵中的元素呢？很好解决，拿mid除以n，商是元素所在行，余数是所在列。这样逐渐二分，就能判断二维矩阵中是不是存在目标值。

### 代码实现

```golang
func searchMatrix(matrix [][]int, target int) bool {
	i := 0
	j := (len(matrix) * len(matrix[0])) - 1
	var mid, x, y int
	if matrix[0][0] > target {
		return false
	}
	if matrix[len(matrix)-1][len(matrix[0])-1] < target {
		return false
	}
	for i <= j {
		mid = (i + j) / 2
		x = mid / len(matrix[0])
		y = mid % len(matrix[0])
		if matrix[x][y] > target {
			j = mid - 1
			continue
		}
		if matrix[x][y] < target {
			i = mid + 1
			continue
		}
		if matrix[x][y] == target {
			return true
		}
	}
	return false
}
```

**执行结果:**

{% asset_img mat3.jpeg 示例图3 %}


### 小结

在二维数组里搜索值，将思路转换为一维数组中查找，利用熟悉的二分法，解决起来思路就很清晰了，这道题很容易就解决了。


### 引用

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/search-a-2d-matrix
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
