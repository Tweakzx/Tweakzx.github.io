---
title: "【Golang】自动机的实现"
author: "Tweakzx"
date: 2022-03-12T16:59:39+08:00
description: 遇到了一道用自动机解决的题目
categories: Go语言
tags: 
  - Go语言
  - 练习代码
image: 
draft: false
---

## 题目描述

请你来实现一个 myAtoi(string s) 函数，使其能将字符串转换成一个 32 位有符号整数（类似 C/C++ 中的 atoi 函数）。

函数 myAtoi(string s) 的算法如下：

1. 读入字符串并丢弃无用的前导空格
2. 检查下一个字符（假设还未到字符末尾）为正还是负号，读取该字符（如果有）。 确定最终结果是负数还是正数。 如果两者都不存在，则假定结果为正。
3. 读入下一个字符，直到到达下一个非数字字符或到达输入的结尾。字符串的其余部分将被忽略。
4. 将前面步骤读入的这些数字转换为整数（即，"123" -> 123， "0032" -> 32）。如果没有读入数字，则整数为 0 。必要时更改符号（从步骤 2 开始）。
5. 如果整数数超过 32 位有符号整数范围 [−231,  231 − 1] ，需要截断这个整数，使其保持在这个范围内。具体来说，小于 −231 的整数应该被固定为 −231 ，大于 231 − 1 的整数应该被固定为 231 − 1 。
6. 返回整数作为最终结果。

注意：

- 本题中的空白字符只包括空格字符 ' ' 。

- 除前导空格或数字后的其余字符串外，请勿忽略 任何其他字符。



示例 1：

> 输入：s = "42"
> 输出：42
> 解释：加粗的字符串为已经读入的字符，插入符号是当前读取的字符。
> 第 1 步："42"（当前没有读入字符，因为没有前导空格）
>                 ^
> 第 2 步："42"（当前没有读入字符，因为这里不存在 '-' 或者 '+'）
>                ^
> 第 3 步："42"（读入 "42"）
>                  ^
> 解析得到整数 42 。
> 由于 "42" 在范围 [-231, 231 - 1] 内，最终结果为 42 。

示例 2：

> 输入：s = "   -42"
> 输出：-42
> 解释：
> 第 1 步："   -42"（读入前导空格，但忽视掉）
>                   ^
> 第 2 步："   -42"（读入 '-' 字符，所以结果应该是负数）
>                     ^
> 第 3 步："   -42"（读入 "42"）
>                       ^
> 解析得到整数 -42 。
> 由于 "-42" 在范围 [-231, 231 - 1] 内，最终结果为 -42 。

示例 3：

> 输入：s = "4193 with words"
> 输出：4193
> 解释：
> 第 1 步："4193 with words"（当前没有读入字符，因为没有前导空格）
>                ^
> 第 2 步："4193 with words"（当前没有读入字符，因为这里不存在 '-' 或者 '+'）
>                ^
> 第 3 步："4193 with words"（读入 "4193"；由于下一个字符不是一个数字，所以读入停止）
>                  ^
> 解析得到整数 4193 。
> 由于 "4193" 在范围 [-231, 231 - 1] 内，最终结果为 4193 。




提示：

- 0 <= s.length <= 200
- s 由英文字母（大写和小写）、数字（0-9）、' '、'+'、'-' 和 '.' 组成

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/string-to-integer-atoi



## 题解

大家可以自行去看leetcode的官方题解，就是用一个自动机来解决这个问题。我第一次用Go来解决这个问题，所以决定记录下来

```go
type Automaton struct{
    state string
    sign  int 
    ans   int 
    table map[string][]string 
}

func(a *Automaton) init(){  //因为Go不能设置类型变量的初始值（我没找到相关代码），所以我用了初始化函数来代替
    a.state  =  "start"
    a.sign = 1
    a.ans = 0
    a.table = map[string][]string{
        "start" : []string{"start","signed","in_number","end"},
        "signed" : []string{"end","end","in_number","end"},
        "in_number" : []string{"end","end","in_number","end"},
        "end" : []string{"end","end","end","end"},
    }
}

func(a *Automaton) get(c byte){ //实现状态转移的方法
    a.state = a.table[a.state][a.get_col(c)]
    switch a.state{
        case "in_number": 
            a.ans = a.ans*10 + int(c-'0')
            if a.sign>0{
                a.ans = min(a.ans, math.MaxInt32)
            }else{
                a.ans = min(a.ans, -math.MinInt32)
            }
        case "signed":
            if c == '-'{
                a.sign = -1
            }
    }
}

func(a *Automaton) get_col(c byte ) int{//获得输入字符的类型
    switch {
        case c==' ': return 0
        case c=='+'||c=='-': return 1
        case '0'<=c&&c<='9': return 2
        default: return 3
    }
}

func(a *Automaton) get_ans() int{ //获得结果
    return a.sign * a.ans
}

func myAtoi(s string) int {
    a := &Automaton{}
    a.init()

    for i:=0; i<len(s); i++{
        a.get(s[i])
    }
    return a.get_ans()
}

func min(a,b int) int{
    if a<b{
        return a
    }
    return b
}
```

