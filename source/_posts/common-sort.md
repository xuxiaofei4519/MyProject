---
title: 常用排序-GO实现
date: 2019-03-26 20:41:00
copyright: true
tags:
 - algorithm
categories:
 - algorithm
---

{% cq %} 
最近正好在学习go，因此用go实现常用的排序算法
{% endcq %}
<!-- more -->

#### 冒泡排序
```go
func BubblingSort(nums []int) []int {
	for i := 0; i < len(nums); i ++ {
		for j := 0; j < len(nums) - 1 - i; j ++ {
			if nums[j] > nums[j + 1] {
				tmp := nums[j]
				nums[j] = nums[j + 1]
				nums[j + 1] = tmp
			}
		}
	}
	return nums
}
```

#### 插入排序
```go
func InsertSort(nums []int) []int {
	return InsertSortByIndex(&nums, 0, len(nums) - 1)
}
// 指定插入排序的范围
func InsertSortByIndex(nums *[]int, left int, right int) []int {
	for p := left + 1; p <= right; p++ {
		tmp := (*nums)[p]
		j := p
		for ; j > left && tmp < (*nums)[j - 1]; j-- {
			(*nums)[j] = (*nums)[j - 1]
		}
		(*nums)[j] = tmp
	}
	return *nums
}
```

#### 选择排序
```go
func ChooseSort(nums []int) []int{
	for i := 0; i < len(nums) - 1; i ++  {
		minIndex := i
		for j := i + 1; j < len(nums); j ++  {
			if nums[j] < nums[minIndex] {
				minIndex = j
			}
		}
		temp := nums[i]
		nums[i] = nums[minIndex]
		nums[minIndex] = temp
	}
	return nums
}
```

#### 快速排序
```go
func QuickSort(nums *[] int)  {
	QuickSortInner(nums, 0, len(*nums) - 1)
}

const CUTOFF int = 10

func QuickSortInner(nums *[] int, left int, right int) {
	// 这里就是跳出递归条件，如果当分割数组太小的时候，使用插入排序能够提高百分之15的效率。
	if left + CUTOFF <= right{
		center := median3(nums, left, right)
		i, j := left, right - 1
		for {
			for {i++; if (*nums)[i] > center{break}}
			for {j--; if (*nums)[j] < center{break}}
			if i < j {
				SwapReference(nums, i , j)
			}else {
				// 肯定只有 j > i 的情况，一旦出现，则退出循环。
				break
			}
		}
		// 重新将center从right-1 放到i位置
		SwapReference(nums, i, right - 1)

		// 递归处理
		QuickSortInner(nums, left, i - 1)
		QuickSortInner(nums, i + 1, right)
	}else {
		InsertSortByIndex(nums, left, right)
	}
}

func median3(arr *[] int, left int, right int) int{
	center := (left + right) / 2
	if (*arr)[center] < (*arr)[left] {
		SwapReference(arr, center, left)
	}
	if (*arr)[right] < (*arr)[left] {
		SwapReference(arr, right, left)
	}
	if (*arr)[right] < (*arr)[center] {
		SwapReference(arr, center, right)
	}
	// 以上是保证三个值的大小顺序

	// 将center放在倒数第二个位置，防止i和j越界
	SwapReference(arr, center, right - 1)
	return (*arr)[right -1]
}

func SwapReference(arr *[]int, index1 int, index2 int) {
	tmp:= (*arr)[index1]
	(*arr)[index1] = (*arr)[index2]
	(*arr)[index2] = tmp
}
```

#### 归并排序
```go
func MergeSort(nums *[]int){
	numsLen := len(*nums)
	tmpArr := make([]int, numsLen)
	mergeSortInner(nums, &tmpArr,0, numsLen - 1)
}


func mergeSortInner(nums *[]int, tmpArray *[]int, left int, right int){

	if left < right {
		center := (left + right) / 2
		mergeSortInner(nums, tmpArray, left, center)
		mergeSortInner(nums, tmpArray, center + 1, right)
		merge(nums, tmpArray, left, center + 1, right)
	}
}

func merge(nums *[]int, tmpArray *[]int, leftpos int, rightpos int, rightEnd int){
	leftEnd := rightpos - 1
	tmpPos := leftpos
	numElement := rightEnd - leftpos + 1
	for leftpos <= leftEnd && rightpos <= rightEnd {
		if (*nums)[leftpos] <= (*nums)[rightpos] {
			(*tmpArray)[tmpPos] = (*nums)[leftpos]
			tmpPos ++
			leftpos ++
		}else {
			(*tmpArray)[tmpPos] = (*nums)[rightpos]
			tmpPos ++
			rightpos ++
		}
	}
	for leftpos <= leftEnd {
		(*tmpArray)[tmpPos] = (*nums)[leftpos]
		tmpPos ++
		leftpos ++
	}

	for rightpos <= rightEnd {
		(*tmpArray)[tmpPos] = (*nums)[rightpos]
		tmpPos ++
		rightpos ++
	}
	for i := 0; i < numElement; i ++ {
		(*nums)[rightEnd] = (*tmpArray)[rightEnd]
		rightEnd --
	}

}
```

#### 堆排序
```go
func HeapSort(nums *[] int)  {
	arrlen := len(*nums)
	// 构造堆
	for i := (arrlen / 2) - 1; i >= 0; i-- {
		percDown(nums, i, arrlen)
	}
	for i := arrlen - 1; i > 0; i-- {
		// 把最大的放到后面
		SwapReference(nums, 0, i)

		// 然后再下滤，这里要求输入i为数组长度，其实i是数组下标，
		// 但是下滤函数中n代表长度，循环会小于n，相当于 < arrlen - 1,这样会忽略我们交换到最后的元素
		percDown(nums, 0, i)
		// i - 1 目的是忽略最大的。
	}
}

// 下滤，根为最大即大顶堆
func percDown(nums *[] int, i int, n int){
	var childIndex int
	tmp := (*nums)[i]
	for ; leftChildIndex(i) < n; i = childIndex {
		childIndex = leftChildIndex(i)
		if childIndex != n - 1 && (*nums)[childIndex] < (*nums)[childIndex + 1]{
			childIndex ++
		}
		if tmp < (*nums)[childIndex] {
			(*nums)[i] = (*nums)[childIndex]
		}else {
			break
		}
	}
	(*nums)[i] = tmp
}

func leftChildIndex(i int) int{
	return 2 * i + 1
}
```