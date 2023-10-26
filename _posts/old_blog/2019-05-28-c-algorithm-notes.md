---
layout: post
title: C++算法大全
date: 2019-05-28
author: Mr Chen
# cover: '/assets/assets/img/shan.jpg'
#cover_author: 'rogov'
#cover_author_link: 'https://unsplash.com/@rogovca'
tags: 
- algorithm
---



> 以内内容为互联网收集，非原创。

## 算法分类

常见的排序算法分为两类：

- **比较类排序：**通过比较来决定元素间的相对次序，由于其时间复杂度不能突破O(nlogn)，因此也称为非线性时间比较类排序。

- **非比较类排序：**不通过比较来决定元素间的相对次序，它可以突破基于比较排序的时间下界，以线性时间运行，因此也称为线性时间非比较类排序。 

<!--more-->

![算法分类](/assets/img/old_blog/algorithm/算法分类.png)

## 算法复杂度

![算法复杂度](/assets/img/old_blog/algorithm/算法复杂度.png)

- **时间复杂度：**对排序数据的总的操作次数。反映当n变化时，操作次数呈现什么规律。
- **空间复杂度：**是指算法在计算机内执行时所需存储空间的度量，它也是数据规模n的函数。 


## 经典排序

### 冒泡排序

冒泡排序是一种简单的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果它们的顺序错误就把它们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。 

#### 算法描述
- 比较相邻的元素。如果第一个比第二个大，就交换它们两个；
- 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对，这样在最后的元素应该会是最大的数；
- 针对所有的元素重复以上的步骤，除了最后一个；
- 重复步骤1~3，直到排序完成。

#### 动图演示

![插入排序](/assets/img/old_blog/algorithm/冒泡排序.gif)

#### 代码实现

~~~cpp
void BubbleSort(int* src, int count)
{
    if(!src || count <= 1) {
        return;
    }
    for (int i(0); i < count; ++i) {
        for (int j(0); j < count - i - 1; ++j) {
            if (src[j] > src[j + 1]) {
                int tmp = src[j + 1];
                src[j + 1] = src[j];
                src[j] = tmp;
            }
        }
    }
}
~~~

### 选择排序（Selection Sort）

选择排序(Selection-sort)是一种简单直观的排序算法。它的工作原理：首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。 

#### 算法描述

n个记录的直接选择排序可经过n-1趟直接选择排序得到有序结果。具体算法描述如下：

- 初始状态：无序区为R[1..n]，有序区为空；
- 第i趟排序(i=1,2,3…n-1)开始时，当前有序区和无序区分别为R[1..i-1]和R(i..n）。该趟排序从当前无序区中-选出关键字最小的记录 R[k]，将它与无序区的第1个记录R交换，使R[1..i]和R[i+1..n)分- 别变为记录个数增加1个的新有序区和记录个数减少1个的新无序区；
- n-1趟结束，数组有序化了。

#### 动图演示

![插入排序](/assets/img/old_blog/algorithm/选择排序.gif)

#### 代码实现

~~~cpp
void SelectSort(int* src, int count)
{
    if (!src || count <= 1) {
        return;
    }
    for (int i(0); i < count; ++i) {
        int min_num = i;
        int min_value = src[i];
        for (int j(i + 1); j < count; ++j) {
            if (src[j] < min_value) {
                min_num = j;
                min_value = src[j];
            }
        }
        if (min_num != i) {
            int tmp = src[min_num];
            src[min_num] = src[i];
            src[i] = tmp;
        }
    }
}
~~~


### 插入排序

插入排序（Insertion-Sort）的算法描述是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

#### 算法描述

一般来说，插入排序都采用in-place在数组上实现。具体算法描述如下：

- 从第一个元素开始，该元素可以认为已经被排序；
- 取出下一个元素，在已经排序的元素序列中从后向前扫描；
- 如果该元素（已排序）大于新元素，将该元素移到下一位置；
- 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置；
- 将新元素插入到该位置后；
- 重复步骤2~5。

#### 动图演示

![插入排序](/assets/img/old_blog/algorithm/插入排序.gif)

#### 代码实现

~~~C
void InsertSort(int *src, int count)
{
    if(!src || count <= 1) {
        return;
    }
    int pre_index(0), cur_value(0);
    for (int i(1); i < count; ++i) {
        pre_index = i - 1;
        cur_value = src[i];
        while (pre_index >= 0 && src[pre_index] > cur_value) {
            src[pre_index + 1] = src[pre_index];
            --pre_index;
        }
        src[pre_index + 1] = cur_value;
    }
}
~~~


### 希尔排序（Insertion Sort）

1959年Shell发明，第一个突破O(n2)的排序算法，是简单插入排序的改进版。它与插入排序的不同之处在于，它会优先比较距离较远的元素。希尔排序又叫缩小增量排序。

#### 算法描述
先将整个待排序的记录序列分割成为若干子序列分别进行直接插入排序，具体算法描述：

- 选择一个增量序列t1，t2，…，tk，其中ti>tj，tk=1；
- 按增量序列个数k，对序列进行k 趟排序；
- 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序。仅增量因子为1 时，整个序列作为一个表来处理，表长度即为整个序列的长度。

#### 动图演示

![插入排序](/assets/img/old_blog/algorithm/希尔排序.gif)

#### 代码实现

~~~cpp
void ShellSort(int *src, int count)
{
    int i, j, k, group;
    for (group = count / 2; group > 0; group /= 2)//增量序列为n/2,n/4....直到1
    {
        for (i = 0; i < group; ++i) {
            for (j = i + group; j < count; j += group) {
                //对每个分组进行插入排序
                if (src[j - group] > src[j]) {
                    int temp = src[j];
                    k = j - group;
                    while (k >= 0 && src[k] > temp) {
                        src[k + group] = src[k];
                        k -= group;
                    }
                    src[k + group] = temp;
                }
            }
        }
    }
}
~~~

### 归并排序（Merge Sort）

归并排序是建立在归并操作上的一种有效的排序算法。该算法是采用分治法（Divide and Conquer）的一个非常典型的应用。将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。若将两个有序表合并成一个有序表，称为2-路归并。 

#### 算法描述

把长度为n的输入序列分成两个长度为n/2的子序列；
对这两个子序列分别采用归并排序；
将两个排序好的子序列合并成一个最终的排序序列。

#### 动图演示

![插入排序](/assets/img/old_blog/algorithm/归并排序.gif)

#### 代码实现

~~~cpp
void Merge(int* src_left, int left_count, int* src_right, int right_count, int *dest)
{
    int left_index(0), right_index(0), index(0);
    while (left_index < left_count && right_index < right_count) {
        if (src_left[left_index] < src_right[right_index]) {
            dest[index] = src_left[left_index];
            ++left_index;
        } else {
            dest[index] = src_right[right_index];
            ++right_index;
        }
        ++index;
    }
    while (left_index < left_count) {
        dest[index++] = src_left[left_index++];
    }
    while (right_index < right_count) {
        dest[index++] = src_right[right_index++];
    }
}

void MergeSort(int* src, int count)
{
    if (!src || count <= 1) {
        return;
    }
    // 首先拆分成 2 个数组
    int middle = count / 2;
    int left_count = middle;
    int right_count = count - middle;

    int *src_left = new int[left_count];
    int *src_right = new int[right_count];
    for (int i(0); i < left_count; ++i) {
        src_left[i] = src[i];
    }
    for (int i(0); i < right_count; ++i) {
        src_right[i] = src[i + middle];
    }
    // 递归 归并排序
    MergeSort(src_left, left_count);
    MergeSort(src_right, right_count);

    Merge(src_left, left_count, src_right, right_count, src);

    delete[]src_left;
    delete[]src_right;
}
~~~

#### 算法分析

归并排序是一种稳定的排序方法。和选择排序一样，归并排序的性能不受输入数据的影响，但表现比选择排序好的多，因为始终都是O(nlogn）的时间复杂度。代价是需要额外的内存空间。


### 快速排序

快速排序的基本思想：通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，则可分别对这两部分记录继续进行排序，以达到整个序列有序。

#### 算法描述

- 先从数列中取出一个数作为key值；
- 将比这个数小的数全部放在它的左边，大于或等于它的数全部放在它的右边；
- 对左右两个小数列重复第二步，直至各区间只有1个数；

#### 动图演示

![快速排序](/assets/img/old_blog/algorithm/快速排序.gif)

#### 代码实现

~~~cpp
void QuikSort(int* src, int left, int right)
{
    if (!src || left >= right) {
        return;
    }
    int i(left), j(right);
    int key = src[left];
    while (i < j) {
        // 从右至左查找第一个小于 key 的值
        while (i < j && src[j] >= key) {
            --j;
        }
        if (i < j) {
            src[i] = src[j];
            ++i;
        }

        // 从左至右查找第一个大于 key 的值
        while (i < j && src[i] < key) {
            ++i;
        }
        if (i < j) {
            src[j] = src[i];
            --j;
        }
    }
    src[i] = key;
    QuikSort(src, left, i - 1);
    QuikSort(src, i + 1, right);
}
~~~


### 堆排序

堆排序（Heapsort）是指利用堆这种数据结构所设计的一种排序算法。堆积是一个近似完全二叉树的结构，并同时满足堆积的性质：即子结点的键值或索引总是小于（或者大于）它的父节点。

#### 算法描述

将初始待排序关键字序列(R1,R2….Rn)构建成大顶堆，此堆为初始的无序区；
将堆顶元素R[1]与最后一个元素R[n]交换，此时得到新的无序区(R1,R2,……Rn-1)和新的有序区(Rn),且满足R[1,2…n-1]<=R[n]；
由于交换后新的堆顶R[1]可能违反堆的性质，因此需要对当前无序区(R1,R2,……Rn-1)调整为新堆，然后再次将R[1]与无序区最后一个元素交换，得到新的无序区(R1,R2….Rn-2)和新的有序区(Rn-1,Rn)。不断重复此过程直到有序区的元素个数为n-1，则整个排序过程完成。

#### 动图演示

![堆排序](/assets/img/old_blog/algorithm/堆排序.gif)

#### 代码实现

~~~cpp
// 构建大顶堆
void HeapBuild(int* src, int root, int count)
{
    if (!src) {
        return;
    }
    int left = root * 2 + 1;
    if (left < count) {
        int max_num = left; // 先默认左节点为最大值的下标
        int right = left + 1;
        if (right < count) {
            if (src[right] > src[max_num]) {
                max_num = right;
            }
        }
        if (src[root] < src[max_num]) {
            // 交换比父节点大的最大子节点
            int tmp_val = src[root];
            src[root] = src[max_num];
            src[max_num] = tmp_val;
            // 从此次最大子节点的那个位置开始递归建堆
            HeapBuild(src, max_num, count);
        }
    }
}

void HeapSort(int* src, int count)
{
    if (!src || count <= 1) {
        return;
    }
    // 从最后一个非叶子节点的父节点开始建堆，这样才能保证顺序
    for (int i(count / 2); i >= 0; --i) {
        HeapBuild(src, i, count);
    }

    // 将已经建好的大顶堆进行排序，1、将最前和最后两个元素对调；2、重新建堆；3、重复第一步
    // j 表示数组此时的长度，因为 count 长度已经建过了
    for (int j(count - 1); j > 0; --j) {
        // 交换首尾元素，将最大值交换到数组的最后位置
        int tmp_val = src[0];
        src[0] = src[j];
        src[j] = tmp_val;
        // 去除最后元素重新建堆
        HeapBuild(src, 0, j);
    }
}
~~~


### 计数排序

计数排序不是基于比较的排序算法，其核心在于将输入的数据值转化为键存储在额外开辟的数组空间中。 作为一种线性时间复杂度的排序，计数排序要求输入的数据必须是有确定范围的整数。

#### 算法描述

- 找出待排序的数组中最大和最小的元素；
- 统计数组中每个值为i的元素出现的次数，存入数组C的第i项；
- 对所有的计数累加（从C中的第一个元素开始，每一项和前一项相加）；
- 反向填充目标数组：将每个元素i放在新数组的第C(i)项，每放一个元素就将C(i)减去1。

#### 动图演示

![计数排序](/assets/img/old_blog/algorithm/计数排序.gif)

#### 代码实现

~~~cpp
void CountSort(vector<int>& src_vec, int max_val)
{
    if (src_vec.empty()) {
        return;
    }
    // 新建一个数组
    vector<int> count(max_val + 1, 0);
    vector<int> tmp(src_vec);
    for (auto val : src_vec) {
        count[val]++;
    }
    for (int i(1); i <= max_val; ++i) {
        count[i] += count[i - 1];
    }
    for (int i = src_vec.size() - 1; i >= 0; --i) {
        src_vec[count[tmp[i]] - 1] = tmp[i];
        count[tmp[i]]--;
    }
}
~~~

#### 算法分析

计数排序是一个稳定的排序算法。当输入的元素是 n 个 0到 k 之间的整数时，时间复杂度是O(n+k)，空间复杂度也是O(n+k)，其排序速度快于任何比较排序算法。当k不是很大并且序列比较集中时，计数排序是一个很有效的排序算法。


### 桶排序

桶排序是计数排序的升级版。它利用了函数的映射关系，高效与否的关键就在于这个映射函数的确定。桶排序 (Bucket sort)的工作的原理：假设输入数据服从均匀分布，将数据分到有限数量的桶里，每个桶再分别排序（有可能再使用别的排序算法或是以递归方式继续使用桶排序进行排）。

#### 算法描述

- 设置一个定量的数组当作空桶；
- 遍历输入数据，并且把数据一个一个放到对应的桶里去；
- 对每个不是空的桶进行排序；
- 从不是空的桶里把排好序的数据拼接起来。 

#### 图片演示

![桶排序](/assets/img/old_blog/algorithm/桶排序.png)

#### 算法分析

现在假设我有一堆蛋，包括麻雀蛋、鸡蛋、恐龙蛋，现在我要将这几种蛋排序下序；
有点常识就知道，这三种类别的蛋大小是不一样的,而且每一类蛋的大小也是有区别的，现在我对这三种蛋进行排序，我是这样排的：
准备三个桶，把同一类别的蛋放到同一个桶中，然后对每一个桶内的蛋进行排序，然后按顺序从三个桶中取出相应蛋排序；
即，从放有麻雀蛋的桶里取出所有麻雀蛋，因为已经有序，直接取出即可，然后再将鸡蛋取出，最后取出恐龙蛋，排序完成；
桶排序即先将大小相近的蛋（或“同一类蛋”）放入到同一个桶中，然后对桶内的蛋进行排序，由于每个桶内蛋的数量比较有限，所以排序效率较高


桶排序是一种时间复杂度为O（n）的排序方法，但并不意味着它有多优秀，上帝打开一扇门的同时也会关上一扇窗
桶排序的局限如下：
1）桶排序耗用较大的辅助空间，所需要的辅助空间一般与被排序的数列的最大值与最小值有关；
2）听到“桶”，很容易联想到哈希表，因为哈希表的冲突解决方法之一开链法也相当于一个个桶,事实上，这两种桶的原理是基本相同的；
3）并非所有数列都适合桶排序,桶排序适合分布比较均匀的数列，如1，2，3，6，4，像0，999，558，10000这种就不太适合
4）桶排序是一种稳定的排序算法
5）桶排序需要维护一个链表，但也有一类简单的桶排序算例如下，所有不同键值是放在不同的桶当中的，所以不用维护链表，当然，下面这种方法也不再稳定

#### 代码实现

~~~cpp
void BucketSort(vector<int>& vec)
{
    int length = vec.size();
    vector<int> buckets(length, 0);//准备一堆桶，容器的下标即待排序数组的键值或键值经过转化后的值
                                   //此时每个桶中都是没有放蛋的，所以都是0

    for (int i = 0; i < length; ++i) {
        buckets[vec[i]]++;//把每个蛋放入到对应的桶中
    }

    int index = 0;
    for (int i = 0; i < length; ++i) {//把蛋取出，空桶则直接跳过
        for (int j = 0; j < buckets[i]; j++) {
            vec[index++] = i;
        }
    }
}
~~~


### 基数排序

基数排序(Radix Sort)是桶排序的扩展，它的基本思想是：将整数按位数切割成不同的数字，然后按每个位数分别比较。 
具体做法是：将所有待比较数值统一为同样的数位长度，数位较短的数前面补零。然后，从最低位开始，依次进行一次排序。这样从最低位排序一直到最高位排序完成以后, 数列就变成一个有序序列。

#### 图文说明

通过基数排序对数组{53, 3, 542, 748, 14, 214, 154, 63, 616}，它的示意图如下：

![基数排序](/assets/img/old_blog/algorithm/基数排序.png)

在上图中，首先将所有待比较树脂统一为统一位数长度，接着从最低位开始，依次进行排序。 
1. 按照个位数进行排序。 
2. 按照十位数进行排序。 
3. 按照百位数进行排序。 
排序后，数列就变成了一个有序序列。

为什么要从个数开始比较，低位是有序的，高位中如果有相同的值，则只需在保持稳定的前提下对高位进行排序，结果自然有序

#### 代码实现

~~~cpp
//radix sort
//基数排序也是基于一种假设，假设所有数都是非负的整数
//基数排序的基本思路是从低位至高位依次比较每个数的对应位，并排序；对应位的比较采用计数排序也可以采用桶排序；
//基数排序是一种稳定的排序方法，不稳定的话也没法排序，因为某一位相同并不代表两个数相同； 
#include<iostream>
#include<vector> 
using namespace std;

void countSort(vector<int>& vec,int exp)
{   //计数排序
	vector<int> range(10,0);
 
	int length=vec.size();
	vector<int> tmpVec(length,0);
 
	for(int i=0;i<length;++i)
	{
		range[(vec[i]/exp)%10]++;
	}
 
	for(int i=1;i<range.size();++i)
	{
		range[i]+=range[i-1];//统计本应该出现的位置
	}
 
	for(int i=length-1;i>=0;--i)
	{
		tmpVec[range[(vec[i]/exp)%10]-1]=vec[i];
		range[(vec[i]/exp)%10]--;
	}
	vec=tmpVec;
}
 
void radixSort(vector<int>& vec)
{
	int length=vec.size();
	int max=-1;
	for(int i=0;i<length;++i)
	{
        //提取出最大值
		if(vec[i]>max)
			max=vec[i];
	}
	
	//提取每一位并进行比较，位数不足的高位补0
	for(int exp=1;max/exp>0;exp*=10)
		countSort(vec,exp);
}
~~~


