---
title: bool 表达式解析
tags: 每日一题
typora-root-url: ../../dongyifeng.github.io
---

> 给定一个只由 0（假）、1 (真)、&（逻辑与）、|（逻辑或）、^（s）五种字符组成的字符串 express，再给定一个布尔值 desired。返回 express 能有多少种组合方式，可以达到 desired 的结果。
>
> 【举例】
>
> Express = “1^0|0|1”，desired = false
>
> 只有 1^((0|0)|1) 和 (1^0|(0|1)) 的组合可以得到 false ，返回 2
>
> 无组合可以得到 false ，返回 0



以下 a = 0 ；b = 1

验证 express 正确性：

1. 长度必须是奇数：a ^ b 或者 a ^ b | a
2. 偶数位是 a 或者 b，奇数位是运算符



![](/images/assets/screenshot-20220916-172557.png)

可能性分析：

![](/images/assets/screenshot-20220916-221110.png)

```python
# 校验 exp 正确性
def is_valid(exp):
    n = len(exp)
    # exp 长度必须奇数
    if n % 2 == 0: return False

    # 偶数位必须是 0 或者 1
    for i in range(0, n, 2):
        if exp[i] != "0" and exp[i] != "1":
            return False

    # 奇数位必须是 ^ | &
    for i in range(1, n, 2):
        if exp[i] != "^" and exp[i] != "|" and exp[i] != "&":
            return False

    return True


# 主函数
def express01(exp, desired):
    if not exp: return 0

    if not is_valid(exp): return 0

    return process(exp, desired, 0, len(exp) - 1)


# L R 一定不要压中逻辑符号
def process(exp, desired, L, R):
    # base_case 当 L == R 时，没有操作符只是 0 或者 1
    if L == R:
        if desired:
            return 1 if exp[L] == "1" else 0
        else:
            return 1 if exp[L] == "0" else 0

    res = 0
    if desired:
        # 逐个尝试操作符
        for i in range(L + 1, R, 2):
            if exp[i] == "|":
                res += process(exp, True, L, i - 1) * process(exp, True, i + 1, R)
                res += process(exp, True, L, i - 1) * process(exp, False, i + 1, R)
                res += process(exp, False, L, i - 1) * process(exp, True, i + 1, R)
            elif exp[i] == "^":
                res += process(exp, True, L, i - 1) * process(exp, False, i + 1, R)
                res += process(exp, False, L, i - 1) * process(exp, True, i + 1, R)
            elif exp[i] == "&":
                res += process(exp, True, L, i - 1) * process(exp, True, i + 1, R)
    else:
        # 逐个尝试操作符
        for i in range(L + 1, R, 2):
            if exp[i] == "|":
                res += process(exp, False, L, i - 1) * process(exp, False, i + 1, R)
            elif exp[i] == "^":
                res += process(exp, False, L, i - 1) * process(exp, False, i + 1, R)
                res += process(exp, True, L, i - 1) * process(exp, True, i + 1, R)
            elif exp[i] == "&":
                res += process(exp, False, L, i - 1) * process(exp, True, i + 1, R)
                res += process(exp, True, L, i - 1) * process(exp, False, i + 1, R)
                res += process(exp, False, L, i - 1) * process(exp, False, i + 1, R)

    return res
```



将暴力递归改为动态规划

有三个变量：desired, L, R。但是 desired 只有两个值，我们可以使用两个 dp 分别对应 desired =True 或者desired = False

![](/images/assets/screenshot-20220916-230255.png)

```python
def express01_v2(exp, desired):
    if not exp: return 0

    if not is_valid(exp): return 0

    n = len(desired)
    dp_true = [[0] * n for _ in range(n)]
    dp_false = [[0] * n for _ in range(n)]
    # 处理 base case
    for i in range(0, n, 2):
        dp_true[i][i] = 1 if exp[i] == "1" else 0
        dp_false[i][i] = 0 if exp[i] == "1" else 1

    for row in range(n - 3, -1, -2):
        for col in range(row + 2, n, 2):

            for i in range(row + 1, col, 2):
                if exp[i] == "|":
                    dp_true[row][col] += dp_true[row][i - 1] * dp_true[i + 1][col]
                    dp_true[row][col] += dp_true[row][i - 1] * dp_false[i + 1][col]
                    dp_true[row][col] += dp_false[row][i - 1] * dp_true[i + 1][col]
                elif exp[i] == "^":
                    dp_true[row][col] += dp_true[row][i - 1] * dp_false[i + 1][col]
                    dp_true[row][col] += dp_false[row][i - 1] * dp_true[i + 1][col]
                elif exp[i] == "&":
                    dp_true[row][col] += dp_true[row][i - 1] * dp_true[i + 1][col]

                if exp[i] == "|":
                    dp_false[row][col] += dp_false[row][i - 1] * dp_false[i + 1][col]
                elif exp[i] == "^":
                    dp_false[row][col] += dp_false[row][i - 1] * dp_false[i + 1][col]
                    dp_false[row][col] += dp_true[row][i - 1] * dp_true[i + 1][col]
                elif exp[i] == "&":
                    dp_false[row][col] += dp_false[row][i - 1] * dp_true[i + 1][col]
                    dp_false[row][col] += dp_true[row][i - 1] * dp_false[i + 1][col]
                    dp_false[row][col] += dp_false[row][i - 1] * dp_false[i + 1][col]

    return dp_true[0][-1] if desired else dp_false[0][-1]
```

