# rCore-2023s-lab3 实验报告

## 实现的功能
本章加入了进程管理这一机制，要实现的系统调用有两个，一个是创建新进程的spawn，另一个是修改进程优先级的set_priority。虽然本章同样需要对之前的章节的内容进行兼容，但由于测试样例中是使用spawn进行的进程创建，因此我先进行了spawn的实现。spawn的实现参考了fork和exec的实现，主要的点在于初始化新进程的进程控制块。而后像过去一章的那样实现gettime、mmap和munmap系统调用，最后实现set_priority系统调用。在set_priority的实现中，我在进程控制块的inner中引入了stride和priority字段，set_priority调用会根据输入的优先级对priority字段进行赋值。在进程调度时，我采用了堆的方式，在所有的就绪进程组成的堆中找到最小stride的进程，然后更新该进程的stride值，将该进程fetch出执行。（在实现过程中，我将TaskManager抽象成rust中的Trait，并将原来默认的任务调度器的名称修改为FIFOTaskManager，新实现的任务调度器命名为StrideManager；同时由于使用了堆结构，需要对进程控制块进行PartialEq、PartialCmp、Eq、Ord等特征的实现）

## 实验练习题回答

+ 实际情况并不是轮到p1执行，因为发生了溢出，大概率是轮到p2继续执行。（在我实现`stride`进程调度的过程中，我刚开始把`BIG_STRIDE`设置的非常大，为usize的最大值，但当内核运行起来以后，会突然卡在其中某个位置，后来把`BIG_STRIDE`调小为`0xffffffff`，就不卡了。我分析可能是因为`BIG_STRIDE`设置的太大会导致pass在经过几次计算后突然溢出，从而导致一些莫名其妙的错误，使得内核卡在某次进程调度中。）

+ 因为pass的值是`BIG_STRIDE`的值除以优先级`Priority`，由于优先级大于等于2，所以一个pass的值一定小于等于`BIG_STRIDE / 2`，那么自然有 `PASS_MAX – PASS_MIN <= BigStride / 2`；

+ 由于每次的stride值等于原本的stride+pass，而一个pass小于等于`BIG_STRIDE / 2`，那么两个进程之间的stride差值就不会比`BIG_STRIDE / 2`大，如果两个进程之间的直接差值的绝对值大于`BIG_STRIDE / 2`，就说明发生了溢出，那么此时stride值大的那个进程，才是下一步应该选择的进程。更进一步，直接让两个进程的stride数相减，如果结果小于`BIG_STRIDE / 2`，则下一个该调度的进程是第二个进程，否则是第一个进程。具体代码如下：

    ```rust
    use core::cmp::Ordering;
    
    struct Pass(u64);
    
    impl PartialOrd for Pass {
        fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
            const BIG_STRIDE: u64 = u64::MAX;	//18_446_744_073_709_551_615u64
            
            (self.0 - other.0).cmp(BIG_STRIDE/2)
            // 当上面的比较结果为小于时，返回Ordering::Less，即后面的other优先级更大
            // 否则返回Ordering::Grater，表示前面的self的优先级更大
        }
    }
    
    impl PartialEq for Pass {
        fn eq(&self, other: &Self) -> bool {
            false
        }
    }
    ```

## 荣誉准则
1. 在完成本次实验的过程（含此前学习的过程）中，我参考过微信群内有关git仓库提交的的相关事项，但并没有与他人做过交流。
2. 此外，我也参考了以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：
[rust中alloc::BtreeMap的相关用法](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html#method.get_key_value)

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。