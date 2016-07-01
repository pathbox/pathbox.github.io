---
layout: post
title: 基本排序的总结
date:   2016-06-21 17:42:06
categories: algorithm
image: /assets/images/post.jpg
---

### 基本排序的总结


```ruby

  class SevenSort

    #............................................................
    # 1.冒泡排序
    # for(i=0;i<len;i++)
    # for(j=1;j<len-i;j++)
    # 复杂度：O(N^2)
    # 性能：极低。 倒序是最糟糕的情况
    # 示例解释：两层循环，内层循环随每次比较减少一次(因为每次比较都得到了一个有序确定的数,下一次比较就不需要再参与排序了)，两两比较，将较大的数交换到后面
    def bubble_sort(a)
      len = a.size
      flag = len
      loop do
        break if flag.zero?
        k = flag
        flag = 0
        (1...len).each do |j|
          if less(a[j-1] , a[j])
            tmp = a[j]
            a[j] = a[j-1]
            a[j-1] = tmp
            flag = j # 如果a[j]和a[j-1]没有任何交换操作,所以已经完全排序好了.这时flag = j这句不会执行,使得flag = 0.从而触发break
          end
        end
      end
      a
    end

    #............................................................
    # 2.选择排序
    # 复杂度: O(N^2)  倒序是最糟糕的情况
    # 和冒泡排序对比: 循环次数一样,交换次数减少。
    # 示例解释：这种方法类似我们人为的排序习惯：从数据中选择最小的同第一个值交换，在从省下的部分中, 选择最小的与第二个交换，这样往复下去。
    def select_sort(a)
      temp = 0
      pos = 0
      len = a.length
      (0...len-1).each do |i|
        temp = a[i]
        pos = i
        j = i + 1
        (j...len).each do |j|
          if less(temp, a[j])
            temp = a[j]
            pos = j
          end
        end
        a[pos] = a[i]
        a[i] = temp
      end
      a
    end

    #............................................................
    # 3.插入排序
    # 复杂度: O(N^2)  best: O(N)
    # 特点：所需时间取决于输入中元素的初始顺序。很大且其中的元素已经有序的数组进行插入排序将会比对随机顺序的数组或是逆序数组进行排序要快得多
    # 插入排序对于小数组,排序性能很高。比如 长度为 5-15位的数组
    def insert_sort(a)
      len = a.size
      (1...len).each do |i|
        j = i
        while(j>0)
          if less(a[j-1], a[j])
            tmp = a[j]
            a[j] = a[j-1]
            a[j-1] = tmp
          end
          j -= 1
        end
      end
      a
    end

    #............................................................
    # 4.希尔排序
    # 复杂度: O(NlogN)
    # 希尔排序比插入排序快很多,数组越大优势越大.不需要用到额外的内存空间。但是 比快速排序慢
    # 然后再选择一个更小的增量，再将数组分割为多个子序列进行排序...最后选择增量为1，即使用直接插入排序，使最终数组成为有序。
    # 增量的选择：在每趟的排序过程都有一个增量，至少满足一个规则 增量关系 d[1] > d[2] > d[3] >..> d[t] = 1
    # (t趟排序)；根据增量序列的选取其时间复杂度也会有变化，这个不少论文进行了研究，在此处就不再深究；本文采用首选增量为n/2,以此递推，每次增量为原先的1/2，直到增量为1
    # 希尔排序通过将比较的全部元素分为几个区域来提升插入排序的性能。这样可以让一个元素可以一次性地朝最终位置前进一大步。然后算法再取越来越小的步长进行排序，
    # 算法的最后一步就是普通的插入排序，但是到了这步，需排序的数据几乎是已排好的了（此时插入排序较快）
    def shell_sort(a)
      len = a.size
      d = len / 2
      while(d >=1 )
        (d...len).each do |i|
          j = i - d
          tmp = a[i]
          while( j >= 0 && a[j] > tmp)
            a[j+d] = a[j]
            j -= d;
          end
          a[j+d] = tmp if j != i - d #j+d = old(j)因为前面j-=d　所以现在j+d回来得到的是最开始的j的索引
        end
        d = d/2
      end
      a
    end

    #............................................................
    # 5. 堆排序
    # 复杂度: O(NlogN)
    # 堆排序(Heapsort)是指利用堆积树（堆）这种数据结构所设计的一种排序算法，它是选择排序的一种。
    # 可以利用数组的特点快速定位指定索引的元素。堆分为大根堆和小根堆，是完全二叉树。

    def heap_sort(arr)
     (arr.length - 1).downto(0).each do |i|
       arr[i], arr[0] = arr[0], arr[i]
       min_heap_fixdown(arr, 0, i)
     end
     arr
    end

    def min_heap_fixdown(arr, i, n)
      left = 2 * i + 1
      while(left < n)
       if left + 1 < n and arr[left] > arr[left + 1]
       left += 1
      end
      if arr[left] < arr[i]
        arr[left], arr[i] = arr[i], arr[left]
        i = left
        left = 2 * i + 1
      else
        break
      end
    end

    #............................................................
    # 6. 归并排序
    # 复杂度: O(NlogN)
    def merge_sort(a)
    end

    #............................................................
    # 7. 快速排序
    # 复杂度: O(NlogN)
    # 非常高效的排序算法
    def quick_sort(a)
      return a if a.size < 2
      left, right = a[1..-1].partition{ |y| y <= a.first }  # 以第一个元素为第一次的比较元素x
      quick_sort2(left) + [ a.first ] + quick_sort2(right)  # 左边部分继续递归排序, 右边部分继续递归排序。直到 a.size < 2.即直到a.size 为 1
    end

    #更详细的解释方法
    def adjust_array(a, l, r)
      i,j = l,r
      x = a[l] # x存放挖坑的值
      while i < j
        while i < j && a[j] >= x
          j -= 1 #从右向左找小于x的数填a[l], 没找到则向左走, a[j] < x时,不进行这个循环,跳出循环,此时如果i<j, 则a[i]=a[j](a[i]的值在x那里放着,不用担心被覆盖)
        end
        if i < j
          a[i] = a[j]  # 找到a[j] < x的值, 将更小的数a[j]放置到左边, a[j]的值赋给a[i]后,变成坑
          i += 1       # i 往右走一步,为下一次查找准备
        end
        while i < j && a[i] < x #从左向右找大于x的数填, 没找到则向右走, a[i] >= x时,不进行这个循环,跳出循环
          i += 1
        end
        if i < j
          a[j] = a[i]  # 找到了a[i] >= x的值, 将更大的数放置到右边,将a[i]的值放到之前形成的a[j]这个坑里,a[i]变为坑
          j -= 1       # j 往左走一步,为下一次查找准备
        end
      end
      a[i] = x         # 一次完整的查找过程完了,回填 x的值。当i==j的时候,表示一次完整的查找结束了。此时i的位置,就是x的值得合适的位置
      i                # 返回i的位置.此时i的左边是小于a[i]的数,右边是大于a[i]。然后进行下一次递归
    end

    def quick_sort(a, l, r)
      if l < r
        i = adjust_array(a, l, r)  # 先挖坑填数法调整a[]
        quick_sort3(a, l, i - 1)   # 递归调用
        quick_sort3(a, i + 1, r)
      end
      a
    end

    private

    def less(a , b)
      a > b ? true : false
    end
  end

#ary = [3,6,7,1,2,9,5,8,1,4]
#sorter = SevenSort.new
#puts sorter.bubble_sort(ary)
#puts sorter.select_sort(ary)
#puts sorter.insert_sort(ary)
#puts sorter.shell_sort(ary)
#puts sorter.quick_sort(ary,0,ary.size)
#puts sorter.heap_sort(ary)

```













