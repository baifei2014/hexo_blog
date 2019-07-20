---
layout: blog
title: n数求和问题
date: 2019-07-19 17:54:13
tags: [数组, 双指针]
---

## 题目描述
给定一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？找出所有满足条件且不重复的三元组。

注意：答案中不可以包含重复的三元组。
```php
例如, 给定数组 nums = [-1, 0, 1, 2, -1, -4]，

满足要求的三元组集合为：
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```
## 解题思路
对于这类型题，最直接的想法就是穷举法，找出所有组合判断结果，如果这样处理，很可能就会超时，还有哪些方法可以处理呢？下面就讲下如何利用双指针去解题。
首先，应该对于这类求和问题，第一步首先要将数组处理成有序的，这样会减少复杂度，如果无序，在遍历的时候还要考虑回溯。第二个，要明白，没法避免要遍历数组，但是在循环时，不能让所有值都是变化的，要确定每次循环的不变量，所以，我们可以在从头开始遍历时，固定第一个元素不变，然后过滤剩下的元素，这里问题是三元组，所以，还有两个元素的变量，如何在剩下的元素过滤出满足的二元组呢？根据求和的特征关系，我们可以采用双指针法。定义好左右指针，当求和值小时，右移左指针，当求和值大时，左移右指针，这样，就可以遍历出所有满足条件的结果了。代码实现如下：
```java
public static List getResult(int[] nums) {
    int length = nums.length;
    int start;
    int end;
    int tmp;
    Arrays.sort(nums);
    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < length-2; i++) {
        while (i > 0 && i < length-2  && nums[i] == nums[i-1]) {
            i++;
        }
        start = i + 1;
        end = length - 1;
        while (start < end) {
            tmp = nums[i] + nums[start] + nums[end];
            if (tmp == 0) {
                List<Integer> list = new ArrayList<>();
                list.add(nums[i]);
                list.add(nums[start]);
                list.add(nums[end]);
                result.add(list);
                while (start < end && nums[start] == nums[start+1]) {
                    start++;
                }
                while (start < end && nums[end] == nums[end-1]) {
                    end--;
                }
                start++;
                end--;
            }
            if (tmp > 0)
                end--;
            if (tmp < 0)
                start++;
        }
    }
    return result;
}
```
以上就是整个实现过程，通过这种方法，我们就可以获取到所有满足的三元组。

以下就是我们使用的测试用例：
```java
public static void main(String args[]) {
    int[] nums = { -1, 0, 1, 2, -1, -4};
    List<List<Integer>> result = new ArrayList<>();
    result = getResult(nums);
    for (int i = 0; i < result.size(); ++i) {
        for (int j = 0; j < result.get(i).size(); ++j) {
            System.out.println(result.get(i).get(j));
        }
    }
}
```
就会输出正确的输出结果，下图就是在leetcode上的提交记录。

{% asset_img leetcode-threesum.png 三数求和 %}

三数求和看完了，下面看下四数求和问题。

## 题目描述
给定一个包含 n 个整数的数组 nums 和一个目标值 target，判断 nums 中是否存在四个元素 a，b，c 和 d ，使得 a + b + c + d 的值与 target 相等？找出所有满足条件且不重复的四元组。

注意：

答案中不可以包含重复的四元组。

示例：
```java
给定数组 nums = [1, 0, -1, 0, -2, 2]，和 target = 0。

满足要求的四元组集合为：
[
  [-1,  0, 0, 1],
  [-2, -1, 1, 2],
  [-2,  0, 0, 2]
]
```

## 解题思路
和三数之和类似，不过这里变量更多了，不过还是可以利用双指针法，首先固定两端不变量，在两端区间左右设置指针，遍历求和，这里也要考虑如何避免重复，下面是解题代码：
```java
public List<List<Integer>> fourSum(int[] nums, int target) {
    int length = nums.length;
    int start;
    int end;
    int tmp;
    Arrays.sort(nums);
    List<List<Integer>> result = new ArrayList<>();
    for (int i = 0; i < length - 3; ++i) {
        while (i > 0 && i < length - 3 && nums[i] == nums[i-1]) {
            i++;
        }
        for (int j = i + 3; j < length; ++j) {
            while (j < length-1 && nums[j] == nums[j+1]) {
                j++;
            }
            start = i + 1;
            end = j - 1;
            while (start < end) {
                tmp = nums[start] + nums[end] + nums[i] + nums[j];
                if (tmp == target) {
                    List<Integer> list = new ArrayList<>();
                    list.add(nums[i]);
                    list.add(nums[start]);
                    list.add(nums[end]);
                    list.add(nums[j]);
                    result.add(list);
                    while (start < end && nums[start] == nums[start+1]) {
                        start++;
                    }
                    while (start < end && nums[end] == nums[end-1]) {
                        end--;
                    }
                    start++;
                    end--;
                }
                if (tmp < target) {
                    start++;
                }
                if (tmp > target) {
                    end--;
                }
            }
        }
    }
    return result;
}
```

以上代码可以通过测试用例测试，这里有一个测试用例如下：
```java
public static void main(String[] args) {
    int[] nums = { 0,4,-5,2,-2,4,2,-1,4 };
    int target = 12;
    List<List<Integer>> result = new ArrayList<>();
    result = getResult(nums, target);
    for (int i = 0; i < result.size(); ++i) {
        for (int j = 0; j < result.get(i).size(); ++j) {
            System.out.println(result.get(i).get(j));
        }
    }
}
```

执行以上用例，就会输出正确的输出结果，下图就是在leetcode上的提交记录。

{% asset_img leetcode-foursum.png 四数求和 %}

## 小结
以上就是n数求和问题的解法，这里给的只是一种解法，利用双指针移动寻找正确组合，还有很多别的更高效率的解法，欢迎讨论。

## 问题来源
- [三数之和](https://leetcode-cn.com/problems/3sum)
- [四数之和](https://leetcode-cn.com/problems/4sum)






