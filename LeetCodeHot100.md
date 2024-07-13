# Hot100

1. 两数之和

   哈希值来快速找到target-x

2. 字母异位词分组

   把单词的字母排序（不覆盖原单词），然后存入map（相同字符组成的存入一组中即可）。

   `char[] array=str.toCharArray();`

   `Array.sort(array);`

   `list.add()`  `list.get()` `map.put()` `map.get()`

3. 最长连续序列数字（要求O(n))

   先记录有哪些数字，然后从第一个开始。一直往上找数字并标记，直到找不到为止（此时断开）。所有的数字都要往大的方向找一遍

4. 移动零

   双指针，所有不是0的都往前挪就好。最后填充0

5. 盛水最多的容器

   双指针、向内侧更新。和接雨水的区别是，接雨水内部物理会挤压空间

6. 三数之和

   排序，然后选择一个作为select。转化为两数之和（左右指针，小移左，大移右）

7. 接雨水

   单调递减栈，每个高于左侧的都会填上当前层的水。（之后就不用再考虑这些了，因为最新的这个成了围栏）

   `Deque<Integer> stack=new LinkedList<>()` `stack.isEmpty()` `stack.peek()` 

   + Java中栈用Deque更好。（虽然也可以Stack）

8. 无重复字符的最长子串

   双指针，用map记录窗口内该字符存在的最右侧为止即可。

9. 找到所有异位词

   滑动窗口，窗口大小是固定的

   `Arrays.equals(sCount,pCount)`其中`sCount` `pCount`是数组（用来记录窗口内目前各个字符的个数，一致时说明时异位词）

10. 和为K的子数组

    前缀和+哈希。首先求得前缀和，此时要获得结果就需要之前，前面有多少个前缀和为（i的前缀和-target）。

    `map.getOrDefault(pre,0)`

11. 滑动窗口最大值

    用单调队列保存窗口内的顺序，由于只看最大的，所以优先存储最右侧最大的（因为左侧的要么不是最大，即便一样大，之后也会被出队列）出窗口的话只需要看最大那些的在不在窗口区域内即可

    双端队列`deque.peekLast()` `deque.pollLast()` `deque.offerLast()` `deque.peekFirst()`

12. 最小覆盖串

    右边界向右移动，如果满足了条件就开始收缩，直到不满足。然后继续右边界右移。

    类似9.异位词。不过窗口大小不固定，需要用两个while进行移动

    `map.containsKey()`

13. 最大和的子数组

    1. 可以用前缀和：当前前缀和-前面的最小前缀和。
    2. 动态规划：dp存储当前元素为结尾的和最大的子数组值。`dp[i]=max{dp[i-1]+nums[i],nums[i]}`即考虑当前元素独立（只是当前独立，可以和后面的合并）还是加入前面的子数组中（要加入肯定是前面以i-1为结尾的最大子数组为正，否则不如自己大）

14. 合并区间

    ```java
    Array.sort(intervals,new Comparator<int[]>(){
        public int compare(int[] interval1,int[] interval2){
            return interval1[0]-interval2[0];
        }
    })// 比较的元素是int[]类型（中的0下标）
    ```

15. 轮转数组

    三次反转（整体，前部，后部）

    `Collection.reverse( list.subList(k,len) )`

    同理：`Collections.sort(List<T> list, Comparator<? super T> c)` 排序

    ​	`Collections.shuffle(List<?> list)` 洗牌

    ​	`Collections.max(Collection<? extends T> coll) `  `Collections.min(Collection<? extends T> coll) `最大最小

    ​		`Collections.replaceAll(List<T> list, T oldVal, T newVal)` 替换old变为new

16. 除自身以外数组的乘积（不用除法）

    用L，R数组分别从左右计算积累乘积即可。每个ans都从i处分割成两部分的乘积

17. 缺失的第一个正数，要求O（n）且常数空间（即原地）

    1. 去除负数并标记为范围外数字（数组大小为范围内）
    2. 下标表示范围内的数字，如果最后为负数则代表下标出现过。
    3. 遍历数组，并把范围内的数字作为下标，反转数组元素为负。

18. 矩阵置0，原地算法

    遍历，用两个数组记录哪些行列需要为0最后设置

19. 螺旋矩阵

    没啥好说的，四条边分别处理即可

20. 搜索二维矩阵（特殊增长规则）

    1. 每行都二分查找

       ```java
       while(low<high){
           int mid=(low+high)/2;
           int num=nums[mid];
           if(num==target){
               return mid;
           }
           if(num>target){
               high=mid-1;
           }else{
               low=mid+1;
           }
         return left;
       }
       ```

       

    2. Z型查找

21. 相交链表

    求长度差，对齐。

22. 反转链表

    + 链表的处理最好都自己加上一个头节点

23. 回文链表

    扔进List

24. 环形链表

25. 环形链表II

    可以用hash存储。`Set<ListNode> visited = new HashSet<ListNode>();`

26. 合并两个有序链表

27. 两数相加（存储在链表）

28. 删除链表的倒数第N个节点

    first先走n个，再一起走。

29. 两两交换链表中的节点

    递归调用交换节点方法。

30. K组一个反转链表

    1. 先确定有k个
    2. 记录本组开头，和下一组开头
    3. 保存pre、start、next三组的关键节点

31. 随机链表的复制

32. 排序链表

33. 合并K个升序链表

    `PriorityQueue<ListNode> pq=new PriorityQueue<>((a,b)->(a.val-b.val));`堆

    `pq.offer()` `pq.isEmpty()` `pq.poll()`

34. LRU缓存

35. 二叉树中序遍历

36. 二叉树最大深度

    ```java
    if(root=null){
        return 0;
    }
    int leftHeight=maxDepth(root.left);//递归
    int rightHeight=maxDepth(root.right);
    return Math.max(leftHeight,rightHeight)+1;
    ```

37. 翻转二叉树

38. 对称二叉树

    ```java
    if(left==null && right==null){
        return true;
    }
    if(left==null || right==null){
        return false;
    }
    if(left.val!=right.val){
        return false;
    }
    check(left.left,right.right) && check(left.right,right.left);
    ```

39. 二叉树直径(两个节点之间的长度)

    计算左右子树的最高高度（即深度），然后加在一起。

40. 二叉树层序遍历

    `Queue<TreeNode> queue=new LinkedList<TreeNode>();` `queue.size()` `queue.offer()` `queue.poll()`

    ```java
    while(queue.isEmpty()){
        List<Integer> level=new ArrayList<Integer>();
        int curLevSize=queue.size();//上一层的节点数
        for(int i=1;i<=curLevSize;++i){
            TreeNode node=queue.poll();//队头出队
            level.add(node.val);//其实就是遍历输出到一层一层
            if(node.left!=null){
                queue.offer(node.left);
            }
            if(node.right!=null){
                queue.offet(node.right);
            }
        }
    }
    ```

41. 有序数组转二叉搜索树

    二分，建树，递归

42. 验证二叉搜索树

    递归判断，保存lower和upper用于限制大小。左子更新upper、右子更新lower

    `return isValidBST(node.left,lower,node.val)&&isValidBST(node.right,node.val,upper);`

43. 二叉树搜索第K小的元素

    中序遍历并计数即可

44. 二叉树的右视图

    1. 层序遍历可以
    2. 深度遍历，ans的大小即已经遍历到的层数。如果此层的深度标记<=ans.size （视深度从0还是从1开始）说明不是第一次到这一层（先遍历右节点，不符合右视）不用加入ans。（也即如果从0计，深度标记==size则说明最深的一层，加入）

45. 二叉树展开为链表

    

46. 从前中序列中构造二叉树

    

47. 路径总和III

    

48. 二叉树的最近公共祖先

    ```java
    if(root==p || root== q|| root==null) return root;
    TreeNode left=lowestCommonAncestor(root.left,p,q);//left不空表示在左子树上找到了p或q
    TreeNode right=lowestCommonAncestor(root.right,p,q);
    
    if(left!=null && right!=null) return root;//都不空，说明是公共祖先。再往上不会出现左右都不空的情况了。
    if(left==null) return right;
    
    return left;//不空说明有目标在子树上
    ```

49. 二叉树中的最大路径和

50. 岛屿数量

    深度优先搜索，每搜完一课树就说明这棵树被隔断了（四面环水）。数组标记被搜索过的节点。

51. 腐烂的橘子

    多源头BFS，最短路径，用层次遍历。一开始的若干烂橘子就是根。一个queue用于层次遍历，一个map用于存储每个位置是第几轮被污染的。

    坐标可以用一维表示，除法和取余即可得到坐标

52. 课程表

    拓扑，用dfs判断图里是否有环

    dfs遍历邻接边。visit[v]=1表示遍历中，visit[v]=2表示遍历完成。

    ```java
    visited[v]=1;//搜索中
    for(int w:edges.get(v)){//遍历邻接边
        if(visited[w]==0){
            dfs(w);
            if(!valid){
                return;
            }
        }else if(visited[w]==1){//表示v所需要的点中有w，且w已经遍历过了(即v是依赖w的，但此时w又直接或间接依赖v)（成环了）。
          	valid=false;
            return;
        }
    }
    visited[v]=2;//表示完成了深度搜索（搜索过了）
    //1表示本轮dfs搜索过了，有环
    //2表示被其他节点开始的dfs搜过了，不用再搜
    ```

    当然也可以BFS遍历

53. 实现前缀树

54. 全排列

    回溯，又一个技巧，把选中的元素与当前下标==**交换**==（此时下标及以前的表示已经选过的元素）回溯的时候再换回来`Collection.swap(list,first,another)`

55. 子集

    track中当前元素：添加/不添加。用cur记录当前的位置（都决定了就是一个结果）回溯。

56. 电话号码的字母组合

57. 组合总和

    `list.add()` `list.remove()`。 回溯问题中常常用传入的idx表示当前要抉择的元素。

58. 括号生成

    `StringBuilder sb` `sb.append(char)` `sb.deleteCharAt()` `sb.length()`

59. 单词搜索

    + 涉及到四个方向时，常常也需要进行辩解判断

    标记为访问，然后在它的基础上四个方向。再回溯（未访问）。如果没有访问标记则导致永远遍历不完

60. 分割回文串

    f\[i]\[j]表示这段区域是否是回文串。如果是，可以从j处切除/不切由此产生的回溯

61. N皇后

    三个变量存储，列、对角、反对角。行直接是用传入的参数（每次dfs+1）表示。循环来遍历一行中的那些列`dfs(i+1)`即表示开始选择第i+1行的棋子落哪里

62. 搜索插入位置

63. 搜索二维矩阵

    看成一维即可，而为坐标可以用一个数字表示出来（反之也可以从数字求出二维坐标）

64. 查找第一个和最后一个target

    ```java
    //找最左侧
    while(low<=high){// 小于等于
        int mid=(low+high)/2;
        if(nums[mid]==target){
            left=mid;
            high=mid-1;//继续收缩high即可
        }else if(nums[mid]>target){
            high=mid-1;
        }else{
            low=mid+1;
        }
    }
    ```

65. 搜索旋转数组

    

66. 寻找旋转排序数组中的最小值

    

67. 寻找两个正序数组的中位数：（对k的二分）

    > 第一次删除的是两个列表和的四分一，第二次删除则是八分之三，第三次是删的十六分之七，依次递归的话，只是无限逼近二分之一也就是第k小数。

    1. 转化为找到两个数组中的第k小的数值
    2. 从两个数组中都找到第k/2个元素。
    3. 假设k1<k2。则比k1小的那k/2个元素一定不是第k小的元素（另一部分则不一定，存在k2是最大值的可能）。舍弃
       + 实际子数组可能不足k/2个。则取数组最后的元素为准
    4. 于是转化为找剩下数组中的第k-k/2个元素（实际不一定是k/2）。
    5. 直到变为以下情况
       1. 有一个数组全被淘汰
       2. 找最小的那个元素

    + 为了方便处理，可以传入数组的顺序（以保证只可能nums1为空）

68. 字符串解码

    `Character.isDigit(cur)`

69. 每日温度

    1. 暴力解

    2. 单调栈

       > 温度递减单调栈（存储没有answer的，但不存储温度而存储下标）。每个新的右边元素往左（底）比较。如果比顶大，则设置顶的answer。
       >
       >  同时，移除顶部。继续往左（底）比较。注意stk的判空即可

70. 柱状图中的最大矩形

    1. 暴力：两层循环，每轮固定一个宽度（最大宽度只有一个矩形，最小宽度有n个矩形）。计算在该宽度下的最大矩形

    2. 单调栈法：对每根柱子都考虑，以它为高度的矩形。最大是多少。此时要求两边的柱子必须高于它并尽力向外扩展（用单调栈求）

    > ```txt
    > //关于是什么栈的【未严肃验证】简单判别法：在找后侧的情况下，直到满足要求才会停，反之的情况就是出栈的情况
    > //比如。找右侧比它大。则找到比他大的停止，也即栈内小于它的则出栈。
    > //比如：找比他左侧小的。停止是比他小的停，则比他大的要出。
    > //但是栈的左右（顶底）方向不确定（自己理一下，和遍历方向有关）
    > //一般“它”指的就是后来的。怎么能把它置于后遍历的位置怎么遍历整个
    > ```

71. 数组中第K个最大元素：用堆

72. 前k个高频元素

    ```java
    //对Map的逆排序
    List<Map.Entry<Integer,Integer>> list=new ArrayList<>();
    list.sort(Map.Entry.comparingByValue(Comparator.reverseOrder()));
    ```

73. 买卖股票的最佳时机

    1. 暴力

    2. 一个变量记录当前节点之前中的最低价，然后可以计算出今天卖最多能赚多少。再遍历计算出来的那最多能赚多少数组即可
       1. 可以继续优化，直接在计算当天能赚多少的时候就顺便求max

74. 跳跃游戏

75. 跳跃游戏II

    从左往右进行判断，如果可以抵达position。则说明这个是一步可以到达position中最左的那个。但是每轮只能走一步。所以需要外层while来判断position是否已经到达数组头了。

76. 划分字母区间

    同一字母最多出现在一个片段中。用数组记录字符的最后一个位置在哪。

    遍历，当遍历到的位置和目前段中所有字符的最后位置end一致时即可切开。（end是不断更新的）

77. 爬楼梯：斐波那契

    最终都是跳一步或者两步。则f(n)=f(n-1)+f(n-2)

78. 杨辉三角

    两侧填1，其余的下标j。则上一行的j-1和j之和

79. 打家劫舍

    ```java
    dp[i] = Math.max(dp[i - 2] + nums[i], dp[i - 1]);
    ```

    不用过多考虑i-1是否选中对i的影响

80. 完全平方数

    两个循环，外层dp[i]。内层一直更新minn（最小的数量）。minn和dp[i-k*k]

    注意两个循环的开始值是从1的

81. 单词拆分

    dp[i]决定于str[0...j...i]：dp[j]是否成立以及str[j...i]是否存在。

    ```java
    for(int i=1;i<=s.length();++i){
        for(int j=0;j<i;++j){
            if(dp[j]&&wordDictSet.contains(s.substring(j,i))){
                //表明[0..j]可以拆分，且[j..i]也是存在的。则0..i可拆
                dp[i]=true;
                break;
    ```

    

82. 最长递增子序列（非连续）

    注意dp初始值全1，

83. 乘积最大数组（有正有负）

    需要注意正负的情况，维护两个dp（初始值为数组本身）。一个存最大，一个存最小。

    每次计算时分别用前一个的最大和最小乘当前。

84. 分割等和子集

    **01背包**，先写二维

    ```java
    dp[0][nums[0]]=true;
            for (int i = 1; i < n; i++) {
                for (int j = 0; j <=target; j++) {
                    if(j>nums[i]){
                        //nums[i]放入与不放
                        dp[i][j]=dp[i-1][j-nums[i]]|dp[i-1][j];
                    }else {//必不能放入
                        dp[i][j]=dp[i-1][j];
                    }
                }
            }
    ```

    优化成一维。

    ```java
    for(int i=1;i<n;++i){
        int num=nums[i];
        for(int j=target;j>=num;--j){
            dp[j]=dp[j]|dp[j-num];
        }
    }//需要用到左上，只能从右下往左上遍历。否则会覆盖。
    ```

85. 最长有效括号

    

86. 不同路径

    二维动态，`f[i][j]=f[i-1][j]+f[i][j-1];`边界置1即可。

87. 最小路径和：` dp[i][j]=Math.min(dp[i-1][j]+grid[i][j],dp[i][j-1]+grid[i][j]);`

88. 最长回文字串

    外层为L，`dp[i][j]=dp[i+1][j-1]`以及考虑特殊情况：

    1. 相等时仅两个字符

    2.  不等则直接false。 dp\[i]\[j]表示的就是\[i...j]这个字串是不是回文

89. 最长公共子序列

    ```java
    dp[i][j]=dp[i-1][j-1]+1;// = 时
    dp[i][j]=Math.max(dp[i-1][j],dp[i][j-1]); //i，j不等时
    ```

90. 编辑距离

    ```java
    edit=dp[i-1][j-1]+1;//修改最后一个字符。
    //word1删除/添加/修改 三选一
    dp[i][j]=Math.min(dp[i-1][j]+1,Math.min(dp[i][j-1]+1,edit));
    ```

91. 只出现过一次的数

    其他只出现两次：两个相同的数异或会变0，0和一个数异或还是那个数

# 春招百题

1. 移掉K位数字变最小

   要使数字最小，就让靠前的数字越小越好。贪心，每次删一位的时候都让结果时最小的。再用单调栈加速过程。

   > 单调递增栈存储最终的结果，存在一种情况，为了形成递增栈，已经删除够k个元素了，则此时便不再删除（可能存在后续不删除的大于删除的，但是如果不删除那些小的，则会导致高位更大从而导致数字更大。比如1231567中23如果不删除则更大，所以一定优先删左侧的）

2. 去除重复字母（要求字典序最小）

   去除重复容易，主要是字典序最小（类比作数字最小，想到单调栈）。以下是单调栈进出的考虑：

   > 如果字符已经在栈里了，就不该再进（举例a：abca,如果a已经在栈里了，说明a后的字符都大于它。如果排出并加入后面的，显然序列变大
   >
   > 应该出去的：当前字母的字母序更小，且被排出字母在后面仍会出现。假如，想排出的字母m后面没了，那么它就不能被排出。否则违反仅去重

3. 拼接最大数【困难】

   两个数组都形成单调递减栈，然后双指针选取最大的元素加入队列中最后形成结果（这个结果未必是最大的，它需要和各种情况下的两组单调栈形成的结果比较）。

   用一个for循环来将k个元素分别分配给两个数组。然后分别形成单调栈。再参照上面的。注意：比较最好采用递归方式，不能只比较某一位（相同时要向后比较决定）

4. 



