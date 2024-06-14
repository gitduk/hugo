+++
title = "两数之和"
date = "2023-12-15"
tags = ["leetcode"]

+++



给定一个整数数组 `nums` 和一个整数目标值 `target`，请你在该数组中找出 **和为目标值** *`target`* 的那 **两个** 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。




**示例 1：**

```
输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。
```

**示例 2：**

```
输入：nums = [3,2,4], target = 6
输出：[1,2]
```

**示例 3：**

```
输入：nums = [3,3], target = 6
输出：[0,1]
```

 

**提示：**

- `2 <= nums.length <= 104`
- `-109 <= nums[i] <= 109`
- `-109 <= target <= 109`
- **只会存在一个有效答案**



## 题解

第一版

```rust
impl Solution {
    pub fn two_sum(nums: Vec<i32>, target: i32) -> Vec<i32> {
    for (i, v) in nums.iter().enumerate() {
        for (i2, v2) in nums[(i + 1)..].iter().enumerate() {
            if v + v2 == target {
                return vec![i as i32, (i + 1 + i2) as i32];
            }
        }
    }
    vec![]
    }
}
```



第二版

```rust
use std::ops::Index;

impl Solution {
    pub fn two_sum(nums: Vec<i32>, target: i32) -> Vec<i32> {

    let mut map: std::collections::HashMap<i32, i32> = std::collections::HashMap::new();
    for (i, v) in nums.iter().enumerate() {
        if map.contains_key(&(target - v)) {
            return vec![*map.index(&(target - v)), i as i32];
        }
        map.insert(*v, i as i32);
    }
    vec![]
    }
}
```



