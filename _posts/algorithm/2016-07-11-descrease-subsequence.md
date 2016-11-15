---
layout: post
title: 最长子序列
category: 算法
tags: Go
description: 求一个数组的最长递减子序列 比如{9，4，3，2，5，4，3，2}的最长递减子序列为{9，5，4，3，2}。
---

首先想到的是使用暴力方式破解，go语言的切片可以很方便地提供一个解决的方案。具体的思路就是每次遇到一个合适的元素，都考虑使用它和不使用它作为自序列元素的情况

```go
package main
import "fmt"

var LastResult [][]int
func main() {
    LastResult = make([][]int, 0)
//    origin := []int{9,8,7,6,5,4,3,2,1}
    origin := []int{1, 35, 43, 2, 3, 455, 1, 3, 6, 3, 46, 3, 4, 7345}
    var result []int
    getSubsequence(origin, result, 0)

    fmt.Println(LastResult)

    max := 0
    last := []int{}
    for _,v := range LastResult {
        if len(v) > max {
            last = v
            max = len(v)
        }
    }

    fmt.Println(max)
    fmt.Println(last)
}

func getSubsequence(origin, result []int, index int) {
    if index >= len(origin)  {
        if len(result) != 0 {
            LastResult = append(LastResult, result)
        }
        return
    }

    /*如果目标数大于结果数组中最后一个数字，则跳过*/
    already := len(result)
    if already != 0 && result[already - 1] <= origin[index] {
        getSubsequence(origin, result, index + 1)
        return
    }

    clone1 := append(result, origin[index])
    clone2 := append([]int{}, result...)
    getSubsequence(origin, clone1, index + 1)
    getSubsequence(origin, clone2, index + 1)
}
```

另外一种方法就是通过动态规划来处理，对于原始数组里面的每一个元素，依次记录以它为结尾的最长子序列的长度。随后我们只需要在这个长度数组中取一个最大的数字即可。

```go
func getSubsequence2(origin []int) int{
    b := make([]int, len(origin))        //生成一个数组b，b[i]标识以origin[i]结尾的最长子序列长度
    b[0] = 1
    for i := 1; i < len(origin); i ++ {
        for j := 0; j < i; j++ {
            if origin[i] < origin[j] && b[i] < b[j] + 1 {
                b[i] = b[j] + 1
            }
        }
    }

    fmt.Println(b)
    var result int
    for _, v := range b {
        if v > result {
            result = v
        }
    }

    fmt.Println(result)
    return result
}
```