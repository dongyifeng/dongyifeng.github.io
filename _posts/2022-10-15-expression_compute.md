---
title: 公式计算
tags: 每日一题
typora-root-url: ../../dongyifeng.github.io
---

> 给定一个字符串str，str 表示一个公式，公式里可能有整数，加减乘除，返回公式的计算结果。
>
> 【举例】
>
> str = “3-6*9+43\*70\*8-6” , 返回 24023
>
> str = “3+1*4” , 返回 7 



分析：

由于乘除的计算优先级高于加减，因此：第一遍遍历优先计算乘除，第二遍遍历再计算加减。

```python
def expression_compute(string):
    stack = []
    i = 0
    while i < len(string):
        item = string[i]
        # 遇到数字
        if '0' <= item <= '9':
            num, j = get_num(string, i)
            stack.append(num)
            i = j
        # 遇到乘除：立即计算
        elif item == "*" or item == "/":
            num, j = get_num(string, i + 1)
            stack.append(compute(stack.pop(), item, num))
            i = j
        # 遇到加减：暂时缓存
        elif item == "+" or item == "-":
            stack.append(item)
            i += 1
    
    # 计算加减
    i = 0
    res = stack[0]
    # 注意从前向后计算
    while i < len(stack) - 1:
        operator = stack[i + 1]
        num2 = stack[i + 2]
        res = compute(res, operator, num2)
        i += 2
    return res


# 从 string 的 index 位置开始获取数字
# 返回：num, i。计算结果，后续要处理的位置
def get_num(string, index):
    num = 0
    for i in range(index, len(string)):
        item = string[i]
        if item < '0' or item > '9': break

        num = num * 10 + int(item)

    return num, i + 1 if i == len(string) - 1 else i

# 运算
def compute(num1, operator, num2):
    if operator == "+": return num1 + num2
    if operator == "-": return num1 - num2
    if operator == "*": return num1 * num2
    if operator == "/": return int(num1 / num2)
  
  
print(expression_compute("14-95+62/50"))
```



> 给定一个字符串str，str 表示一个公式，公式里可能有整数，加减乘除和左右括号，返回公式的计算结果。
>
> 【举例】
>
> str = “48*((70-65)-43)+8\*1” , 返回 -1816 
>
> str = “3+1*4” , 返回 7 
>
> str = “3+(1*4)” , 返回 7
>
> 【说明】
>
> 1. 可以认为给定的字符串一定是正确的公式，即不需要对 str 做公式有效性检查。
> 2. 如果是负数，就需要用括号括起来，比如”4*(-3)“ 。但如果负数作为公式的开头或者括号部分的开头，则可以没有括号，比如：”-3\*4“ 和 ”(-3\*4)“ 都是合法的。
> 3. 不用考虑计算过程中发生的溢出的情况



运算规则：

- 优先计算括号内公式
- 再计算乘算
- 再计算加法



在计算括号内公式时：遇到 ”(“ 时，递归调用，再遇到 “)” 在进行计算，每次调用 que 中只存储当前括号内的公式。

![](/images/assets/screenshot-20221015-090401.png)



```python
def expression_compute(string):
    return process(string, 0)[0]

def get_num(string, index):
    num = 0
    for i in range(index, len(string)):
        item = string[i]
        if item < '0' or item > '9': break

        num = num * 10 + int(item)

    return num, i + 1 if i == len(string) - 1 else i

# 计算 string 公式
# 优先括号内的公式，再计算乘除，再计算加减
def process(string, index):
    que = []
    i = index
    while i < len(string):
        # (：递归调用
        if string[i] == "(":
            num, j = process(string, i + 1)
            i = j
            que.append(num)
        # 遇到数字
        elif "0" <= string[i] <= "9":
            num, j = get_num(string, i)
            que.append(num)
            i = j
        # ) 计算括号内的公式
        elif string[i] == ")":
            res = sub_expression_compute(que)
            return res, i + 1
        else:
            que.append(string[i])
            i += 1

    # 计算没有括号，剩下的公式
    return sub_expression_compute(que), i

# 计算 array 内的公式
# 优先计算乘除，再计算加减
def sub_expression_compute(array):
    stack = []
    i = 0
    # 计算乘除
    while i < len(array):
        if array[i] == "*" or array[i] == "/":
            stack.append(compute(stack.pop(), array[i], array[i + 1]))
            i += 2
        else:
            stack.append(array[i])
            i += 1

    # 计算加减
    i = 0
    res = stack[0]
    while i < len(stack) - 1:
        operator = stack[i + 1]
        num2 = stack[i + 2]
        res = compute(res, operator, num2)
        i += 2
    return res

# 运算
def compute(num1, operator, num2):
    if operator == "+": return num1 + num2
    if operator == "-": return num1 - num2
    if operator == "*": return num1 * num2
    if operator == "/": return int(num1 / num2)
  
print(expression_compute("48*((70-65)-43)+8*1"))
print(expression_compute("3+1*4"))
print(expression_compute("3+(1*4)"))
```

