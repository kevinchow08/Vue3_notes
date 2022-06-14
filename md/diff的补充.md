## 位运算

位运算符将它的操作数视为 32 位元的二进制串（0 和 1 组成）而非十进制八进制或十六进制数。

例如：十进制数字 9 用二进制表示为 1001，位运算符就是在这个二进制表示上执行运算，但是返回结果是标准的 JavaScript 数值。

基本位运算如下:

|  Operator   | Usage  | Description  |
|  ----  | ----  | ----  |
| 按位与 AND  | a & b | 在 a,b 的位表示中，每一个对应的位都为 1 则返回 1， 否则返回 0. |
| 按位或 OR  | a \| b | 在 a,b 的位表示中，每一个对应的位，只要有一个为 1 则返回 1， 否则返回 0. |
| 按位非 NOT  | ~ a | 反转被操作数的位 |
| 按位异或 XOR  | a ^ b | 在 a,b 的位表示中，每一个对应的位，两个不相同则返回 1，相同则返回 0. |
| 左移 shift  | a << b | 将 a 的二进制串向左移动 b 位，右边移入 0. |
| 算术右移  | a >> b | 把 a 的二进制表示向右移动 b 位，丢弃被移出的所有位。 |

结合之前的代码来看:
```js
// 位运算
// 这则是一个判断shapeFlag是否包含ShapeFlags.COMPONENT这个权限
// eg: 1111 & 0110 => true
if (shapeFlag & ShapeFlags.COMPONENT) {
    processComponent(n1, n2, container);
} else if (shapeFlag & ShapeFlags.ELEMENT) {
    processElement(n1, n2, container, anchor);
}

export const enum ShapeFlags {
  ELEMENT = 1, // 元素 000001
  FUNCTIONAL_COMPONENT = 1 << 1, // 函数式组件 000010
  STATEFUL_COMPONENT = 1 << 2, // 普通组件 000100
  TEXT_CHILDREN = 1 << 3, // 孩子是文本 001000
  ARRAY_CHILDREN = 1 << 4, // 孩子是数组 010000
  SLOTS_CHILDREN = 1 << 5, // 组件插槽  100000
  COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT 	// 组件, 两种权限的组合
}
```
关于ShapeFlags的定义和使用: 实际代码中的 `| &` 是做权限必备的一个操作, `|`一般是用来组合权限. `&` 来判断当前节点的shapeFlag是否包含定义的某个权限(ShapeFlags.which)

下面是两道位运算的题目:

```js
// 计算一个32位二进制数的1的个数
var hammingWeight = function(n) {
    var count = 0
    for(let i = 0; i < 32; i++) {
        // if (1 & n) {
        //     count++
        // }
        // n >>= 1
        if(n & (1 << i)) {
            count++
        }
    }
    return count
};
```

```js
// 判断n是否为2的幂: 二进制数字,只有最高位为1时,才满足条件
// 16
// 10000 => true
// 16-1 = 15
// 01111 => false
// 16&15 == 0

var isPowerOfTwo = function(n) {
    return n > 0 && (n & (n - 1)) === 0
};
```

## 最长递增子序列

给你一个整数数组 nums (eg: `[10, 9, 2, 5, 3, 7, 100]`) ，找到其中最长严格递增子序列的长度。

key: 
+ 利用动态规划的思路去解决: 设置一个dp数组, `dp[i]` 则表示为前 i 个元素，以第 i 个数字结尾的最长上升子序列的长度. 初始化dp, 设置长度为`nums.length`. 值为 `1`.
+ 首先明确一点, `nums[i]`的最长递增子序列,一定都在`nums[i]`的左侧. 我们从小到大计算 dp 数组的值，在计算 dp[i] 之前，我们已经计算出 dp[0…i−1] 的值，则状态转移方程为：
`dp[i]=max(dp[j])+1`, 其中 `0≤j<i` 且 `nums[j] < nums[i]`

+ 具体详细讲解可参见: [视频讲解](https://www.bilibili.com/video/BV19b4y1R7K3?spm_id_from=333.337.search-card.all.click&vd_source=7e26628ab4ba9f5e2f2ee6543c0bc288)

代码如下:
```js
function lengthOfLIS(nums) {
    var resultLen = 1
    var dp = new Array(nums.length).fill(1)
    for(var i = 1; i < nums.length; i++) {
        for(var j = 0; j < i; j++) {
            if(nums[j] < nums[i]) {
                // 注意, 只要满足nums[j] < nums[i], dp[i]会重新计算
                // 所以dp[i]是在不断更新中的, 但会进行大小的比较
                dp[i] = Math.max(dp[i], dp[j] + 1)
            }
        }
        resultLen = Math.max(dp[i], resultLen)
    }
    return resultLen
}
```

方法二: 贪心 + 二分查找(降低时间复杂度)



```js
function lengthOfLIS(nums) {
   var tail = [nums[0]]
    for(var i = 0; i < nums.length; i++) {
        if(nums[i] > tail[tail.length - 1]) {
            tail.push(nums[i])
        } else {
            var left = 0
            var right = tail.length - 1
            while(left < right) {
                var mid = (left + right) >> 1
                if(nums[i] > tail[mid]) {
                    left = mid + 1
                } else {
                    right = mid
                }
            }
            tail[left] = nums[i]
        }
    }
    return tail.length
}
```

上文中只是获取到最长递增子序列的长度, 但在vue3中, 需要得到这个子序列所对应的索引, 具体该如何做 ??

