# 字符数组、字符串操作

```java
//字串转字符数组
String str;
char[] array=str.toCharArray();
//字符数组排序
Arrays.sort(array);
//数组的长度
int[] height;
int size=height.length;//可直接.size()
//字符串
String s;
int n=s.length();
char ch=s.charAt(right);//获取字符
//数组比较
int[] sCount=new int[26];
int[] pCount=new int[26];
Arrays.equals(sCount,pCount);//注意s
//转为List
int[] res;
Arrays.asList(res);
//二维数组一次拷贝一行
tmp[i]=matrix[i].clone();
```

# Map

```java
//从map中查找key值
map.getOrDefault(key,new ArrayList<String>());//第一个参数是要找的值，第二个参数是如果没有找到则返回的值
//看看有没有Key=ch的
map.containsKey(ch);
map.get(ch);
```

# Set

```java
//注意是 contain-s()
num_set.contains(num-1)
```

# Deque

```java
Deque<Integer> deque=new LinkedList<Integer>
//从双端队列的尾部移除并返回一个元素
deque.pollLast();
//放入尾部
deque.offerLast(i);
//拿到但不移除
deque.peek();
deque.peekFirst();//和peek没区别
deque.peekLast();
```

