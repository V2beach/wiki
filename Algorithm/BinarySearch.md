# Binary Search

单调问题是适用二分的必要前提，问题场景需要是某个函数上的任意一段单调位置。

写2226这类问题基本靠蒙边界AC。

```cpp
    while (left < right){
        int mid = right - (right - left) / 2;
        if (isOK){
            left = mid;
        }else{
            right = mid - 1;
        }
    }
	return left;
```

用这个板子对了，但用

```cpp
    while (left < right){
        int mid = left + (right - left) / 2;
        if (isOK){
            left = mid + 1;
        }else{
            right = mid;
        }
    }
	return left;
```

结果不对，会大1。

```cpp
// 适用于二分搜最优值
while (left < right)
{
   mid = left+(right-left)/2; 
   // 或者 right-(right-left)/2，取决于下面是否可能会有死循环

   if (mid是一个可行解) 
   {
 二分区间砍半去掉不够优秀的那部分，注意保留mid本身;
   }
   else
   {
 二分区间砍半去掉不可行的那部分，注意不保留mid本身;
   }
}
return left;  // 双闭区间，最终收敛解一定就是最优解
```

（贴一下群主的完整板子，退出时left一定==right，那返回left、right、mid都可以的吧？2226里right可mid不可。）

left+(right-left)/2相当于向下取整，right-(right-left)/2意思是向上取整。

首先用(right - left) / 2而不用加法(right + left) / 2是为了不溢出。

然后所谓开闭区间是啥？大体理解的是，每次循环搜索的区间。

> 二分查找算法有64种写法(?)。cpp官方模块的源码用的是左闭右开区间。

用cpp的lower_bound和刚做的2226分糖作为例子捋一下吧，**说起来华为考的那个分糖我也没做出来。**

lower_bound:

[1,2,3,5,5,5,6,7,9], target=5, 找满足x>=target的下界(第一个>=target的位置)。

按群主的板子，如果用left+(right-left)/2更新：

left = 0, right = 8, 双闭区间的意思就是，第一次搜索index[0, 8], 比对的是mid = 4, 发现5>=5满足，找的是下界则丢掉右边，按上面的板子来做right = mid，查找[0, 4]区间，mid = 2, 发现3>=5不满足，丢掉左边，查找[3, 4]区间，mid = 3，发现5>=5满足，丢掉右边，此时right = mid = 3，left < right不满足，返回left为3，找到了正确结果...

那么如果用right-(right-left)/2更新，第一次还是4，第二次也是2，第三次mid会更新为4，left一直是3，right一直是4，死循环了。

这个是找下界，下面这个题是找上界，可能上界就需要mid向上取整吧。

另外，left和right是否保留mid的问题，在这个例子里也体现不明显。可以看下一俩三四五的视频，学一个、熟练推导一个固定的板子，很多时候不是靠记忆是靠自己复现算法的推导思路。

2226.max candies allocated to k children（自检标准——再让我重新做这个题，我知道怎么匹配板子吗？）
几个piles的candies平均分给k个孩子，[5,8,6], k = 3, max = 5，三堆分别分成5, 5/3, 5/1。

下面这俩hard有点搞我心态了。

2145的check函数想了很久，但真的不难，n台电脑因必须并行，所以可以看作完全一样，只需要把battery>=电脑运行时长t的只给一台电脑使用，battery<t的全都加起来只要>剩下的n'台电脑*t就可以了。（即所有battery>t的电池都算作t求和）

这里要养成一个思维习惯就是，check函数不要想这个最优解为题，就是一个给确定输入输出的函数，比如这里的check，只需要考虑是一个“判断有一群电脑需要同时运行t时长，手里的batteries够不够满足这n台电脑？”的问题。**无论这个t是小到0，还是大得离谱，t是给定的，写这个check函数，我要考虑的只是能不能满足t。**

1648这种就需要想猜的目标是什么，我知道了能用二分都一下想不到猜哪个值。猜各种颜色的球最多卖的次数？看了一下，群主跟我想的差不多，就是猜最后一次卖出的价值v，要让这个v最大，卖的数量<orders也最大，v最大时总值就最大。就猜v，最后手动补足数量=orders。（因为v如果再-1就一定>orders了，所以肯定能补齐）