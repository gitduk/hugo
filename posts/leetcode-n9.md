+++
title = "回文数"
date = "2023-12-27"
tags = ["leetcode"]

+++



给你一个整数 `x` ，如果 `x` 是一个回文整数，返回 `true` ；否则，返回 `false` 。

回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。

- 例如，`121` 是回文，而 `123` 不是。




**示例 1：**

```
输入：x = 121
输出：true
```

**示例 2：**

```
输入：x = -121
输出：false
解释：从左向右读, 为 -121 。 从右向左读, 为 121- 。因此它不是一个回文数。
```

**示例 3：**

```
输入：x = 10
输出：false
解释：从右向左读, 为 01 。因此它不是一个回文数。
```

 

- **提示：**
  - `-231 <= x <= 231 - 1`



## 题解

```rust
impl Solution {
    pub fn is_palindrome(x: i32) -> bool {
        if 0 <= x && x <= 9 { return true }
        if x == 10 { return false }
        if x < 0 { return false }

        let length = x.to_string().len() / 2;

        let mut left = x;
        let mut right = 0;
        for i in 0..length {
            right = right * 10 + left % 10;
            left = left / 10;
        }

        println!("{}|{}", left, right);

        return right == left || right == left / 10
    }
}
```
