# 算法题模板

Acwing算法基础课模板

<!--more-->

## 排序

### 快速排序

```C++
void quick_sort(int q[], int l, int r)
{
	if (l >= r) return;  
    
    int i = l - 1, j = r + 1, x = q[ l + r >> 1 ];
    while(i < j)
    {
        do i++ ; while(q[i] < x);
        do j-- ; while(q[j] > x);
        if(i < j) swap(q[i],q[j]);	
    }
    
    quick_sort(q, l, j);	//只会出现i == j 或者 i = j + 1 情况
    quick_sort(q, j+1, r);
}
```

### 归并排序

```C++
void merge_sort(int a[], int l, int r)
{
    if(l>=r) return;
    
    int mid = l+r >> 1;
    merge_sort(a,l,mid);
    merge_sort(a,mid+1,r);
    
    int k = 0, i = l, j = mid+1;
    while(i <= mid && j <= r)
    {
        if(a[i] <= a[j]) tmp[k++] = a[i++];
        else tmp[k++] = a[j++];
    }
    while(i <= mid) tmp[k++] = a[i++];
    while(j <= r) tmp[k++] = a[j++];
    
    for(i = l, j = 0; i <= r; i++, j++) a[i] = tmp[j];
}
```



## 二分

### 整数二分

```C++
//先写check函数，想一下check如何更新更新区间，如果l = mid, 需要+1
int bsearch_1(int l, int r)
{
	while(l < r)
	{
		int mid = l+r >> 1;
		if(check(mid)) r = mid;
		else l = mid + 1;
	}
	return l;
}
int bsearch_2(int l, int r)
{
	//区别在于当l = mid, 需要+1
	while(l < r)
	{
		int mid = l + r + 1 >> 1;
		if(check(mid)) l = mid;
		else r = mid - 1;
	}
	return l;
}
```

### 浮点数二分

```C++
double bsearch_3(double l, double r)
{
	//eps 尽量比题目要求再高出两位，减少误差
	while(r-l > eps)
	{
		double mid = (l + r) / 2;
		if(check(mid)) r = mid;
		else l = mid;
	}
	return l;
}
```



## 高精度

### 高精度加法

```c++
vector<int> add(vector<int>& a, vector<int>& b)
{
    vector<int> c;
    int t = 0;
    for(int i = 0; i < a.size() || i < b.size(); i++)
    {
        if(i < a.size()) t += a[i];
        if(i < b.size()) t += b[i];
        c.push_back(t % 10);
        t /= 10;
    }
    if(t) c.push_back(t);
    
    return c;
}
```



### 高精度减法

```c++
vector<int> sub(vector<int> &A, vector<int> &B)
{
    vector<int> C;
    for (int i = 0, t = 0; i < A.size(); i ++ )
    {
        t = A[i] - t;
        if (i < B.size()) t -= B[i];
        C.push_back((t + 10) % 10);
        if (t < 0) t = 1;
        else t = 0;
    }

    while (C.size() > 1 && C.back() == 0) C.pop_back(); //去除前导0
    return C;
}
```



### 高精度乘法

```C++
vector<int> mul(vector<int> &A, int b)
{
    vector<int> C;
    int t = 0;
    for (int i = 0; i < A.size() || t; i ++ )
    {
        if (i < A.size()) t += A[i] * b;
        C.push_back(t % 10);
        t /= 10;
    }

    while (C.size() > 1 && C.back() == 0) C.pop_back();
    return C;
}
```



### 高精度除法

```C++
vector<int> div(vector<int>& A, int B, int& r)
{
    vector<int> c;
    r = 0;
    //从最高位算
    for(int i = A.size() - 1; i >= 0 ;i--)
    {
        r = r * 10 + A[i];
        c.push_back(r / B);
        r %= B;
    }

    reverse(c.begin(), c.end());
    while(c.size() > 1 && c.back() == 0) c.pop_back();
    return c;
}
```



## 差分和前缀和

### 前缀和

```c++
//构造二维前缀和
s[i][j] = s[i - 1][j] + s[i][j - 1] - s[i - 1][j - 1] + a[i][j];
//获取二维前缀和
s[x2][y2] - s[x1 - 1][y2] - s[x2][y1 - 1] + s[x1 - 1][y1 - 1];
```

### 差分

```c++
// a[i] = b[1] + b[2] + ... + b[i]
void insert(int l, int r, int c)
{
    a[l] += c;
    a[r + 1] -= c; 
}
// 二维差分
void insert(int x1, int y1, int x2, int y2, int c)
{
    a[x1][y1] += c;
    a[x2 + 1][y1] -= c;
    a[x1][y2 + 1] -= c;
    a[x2 + 1][y2 + 1] += c;
}
b[i][j] = b[i][j] + b[i - 1][j] + b[i][j - 1] - b[i - 1][j - 1];
```

## 双指针

```c++
//需要找到一些性质(单调性) 使得O(n^2)变为O(n)
for(int i = 0, j = 0; i < n; i++)
{
    while(j < i && check(i, j)) j++;
}
```

## 位运算

```c++
求第k位 n >> k & 1;
返回最后一位1的位置 lowbit(n) n & -n ;
```

## 离散化

```c++
//区间和
vector<int> alls; //存储所有离散化的值
sort(alls.begin(), alls.end()); //排序
alls.erase(unique(alls.begin(), alls.end()), alls.end()); //去重

int find(x)
{
    int l = 0, r = alls.size() - 1;
    while(l < r)
    {
        int mid = (l + r) >> 1;
        if(alls[mid] >= x) r = mid; 
        else l = mid + 1;
	}
    return r + 1; //便于求前缀和
}
```

## 区间合并

```c++
void merge(vector<pii> &segs)
{
    vector<pii> res;
    
    sort(segs.begin(), segs.end());
    int st = -2e9, ed = -2e9;
    for(auto seg : segs)
    {
        if(ed < seg.first)
        {
            if(st != -2e9) res.push_back({st, ed});
            st = seg.first, ed = seg.second;
		}
        else ed = max(ed, seg.second);
	}
    if(st != -2e9) res.push_back({st,ed});
    
    segs = res;
}
```

## 数据结构

### 单链表

```C++
//主要应用 邻接表，存储图和树
int head, e[N], ne[N], idx;
void init()
{
    head = -1, idx = 0;
}
void add_to_head(int x)
{
    e[idx] = x, ne[idx] = head, head = idx, idx ++;
}
//在下标为k的节点后添加
void add(int k, int x)
{
    e[idx] = x, ne[idx] = ne[k], ne[k] = idx, idx ++;
}
//在下标为k的节点后删除
void remove(int k)
{
    ne[k] = ne[ne[k]];
}
```

### 双链表

```C++
//l[i] 节点i左边的点是   r[i] 节点i右边的点是
int e[N], l[N], r[N], idx;
void init()
{
    r[0] = 1, l[1] = 0;
    idx = 2;
}
//在下标为k的节点右边插入
void add(int k, int x)
{
    e[idx] = x;
    r[idx] = r[k];
    l[idx] = k;
    l[r[k]] = idx;
    r[k] = idx;
    idx ++;
}
//在下标为k的节点左边插入
void add(int l[k], int x);


//删除下标为k的节点
void remove(int k)
{
    l[r[k]] = l[k];
    r[l[k]] = r[k];
}
```

### 栈

```C++
//tt表示栈顶
int stk[N], tt;
//插入
stk[++tt] = x;
//弹出
tt--;
//栈顶的值
stk[tt];
//判断栈是否为空
if(tt > 0) {};
```

### 队列

``` c++
int q[N], hh = 0, tt = -1;
//插入
q[++tt] = x;
//弹出
hh++
//判空
if(hh <= tt) not empty
//取出队头元素
q[hh];
```

### 单调栈

```C++
//常见模型：找出每个数左边离它最近的比它大/小的数
int tt = 0;
for(int i = 0; i < n; i++)
{
    while(tt && stk[tt] >= x) tt--;
    if(tt) {}
    else {}
    stk[++tt] = x;
}
```

### 单调队列

```C++
//常见模型：求出滑动窗口的最大值/最小值

int hh = 0, tt = -1;
for(int i = 0; i < n; i++)
{
    while(hh <= tt && q[hh] < i - k + 1) hh++;  //判断队头是否滑出窗口
    while(hh <= tt && a[q[tt]] >= a[i]) tt--;
    q[++tt] = i;
}
```

### KMP

```C++
//求next的过程
for(int i = 2, j = 0; i <= n; i++)
{
    while(j && p[i] != p[j + 1]) j = next[j];
    if(p[i] == p[j + 1]) j++;
    next[i] = j;
}
//匹配过程
for(int i = 1, j = 0; i <= m; i++)
{
    //j 总往前错一位
    while(j && s[i] != p[j + 1]) j = next[j];
    if(s[i] == p[j + 1]) j++;
    if(j==m)
    {
        //匹配成功后的逻辑
    }
}
```

### Trie数

```C++
int son[N][26], cnt[N], idx;

// 0号点既是根节点，又是空节点
// son[][]存储树中每个节点的子节点
// cnt[]存储以每个节点结尾的单词数量

void insert(char* str)
{
    int p = 0;
    for(int i = 0; str[i]; i++)
    {
        int u = str[i] - 'a';
        if(!son[p][u]) son[p][u] = ++idx;
        p = son[p][u];
    }
    cnt[p]++;
}
int query(char *str)
{
    int p = 0;
    for(int i = 0; str[i]; i++)
    {
        int u = str[i] - 'a';
        if(!son[p][u]) return 0;
        p = son[p][u];
	}
    return cnt[p];
}
```

### 并查集

```C++
//作用:
//1.将两个集合合并
//2.询问两个元素是否在一个集合中
//朴素并查集
int p[N];
int find(int x)
{
    if(p[x] != x) p[x] = find(p[x]);
    return p[x];
}
//合并操作
p[find(a)] = find(b);


//维护size的并查集
int p[N], size[N];
int find(int x)
{
    if(p[x] != x)  p[x] = find(p[x]);
    return p[x];
}
//合并操作
size[find(b)] += size[find(a)];
p[find(a)] = find(b);
```

### 堆

```C++
int h[N], size;
for(int i = n/2; i; i--) down(i);

//和左右孩子进行比较
void down(int u)
{
    int t = u;
    if(t * 2 <= size && h[t * 2] < h[u]) t = u * 2;
    if(t * 2 + 1 <= size && h[t * 2 + 1] < h[u]) t = u * 2 + 1;
    if(t != u)
    {
        swap(h[t], h[u]);
        down(t);
    }
}

void up(int u)
{
    while(u / 2 && h[u / 2] > h[u])
    {
        swap(h[u / 2], h[u]);
        u = u / 2;

    }
}
```

### 哈希表

```C++
//拉链法
int h[N], e[N], ne[N], idx;
void insert(int x)
{
    int k = (x % N + N) % N;
    e[idx] = x, ne[idx] = h[k], h[k] = idx++; 
}
bool find(int x)
{
    int k = (x % N + N) % N;
    for (int i = h[k]; i != -1; i = ne[i])
        if (e[i] == x)
            return true;
    return false;
}
```



## STL

### vector

```C++
vector<int> a;
//所有容器均有的操作
a.size();
a.empty();

a.clear();
front()/back();
push_back()/pop_back();
begin()/end();
支持比较运算;

```

### pair

```C++
pair<int, int> p;
p = make_pair(123, 345)/p = {123, 345};
p.first/p.second;
```

### string

```C++
clear();
sub_str(int startindex, int len);
c_str() 返回字符数组的启示地址;
```

### queue

```C++
push()/pop();
front()/back();
//清空
q = queue<int>();
```

### priority_queue

```C++
push()/pop();
top();
//默认是大根堆，实现小根堆
1.插入负数；
2.priority_queue<int, vector<int>, greater<int>> heap;
```

### stack

```C++
push()/pop();
top();
```

### deque

```C++
//速度较慢
front()/back();
push_back()/pop_back();
push_front()/pop_front();
begin()/end();
```

### set / multiset

```C++
//set没有重复元素 multiset可以有
insert();
find();//不存在返回end迭代器
count(); //某个数个数
erase(); //可以删除所有数，也可以删除单个迭代器
lower_bound()/ upper_bound(); //返回 >= x最小的数 / 返回 > x最小的数  没有返回end()
```

### map / multimap

```C++
insert();
erase();
find();
lower_bound()/ upper_bound();
```



### unorder_set / unorder_multiset / unorder_map / unordered_multimap 

```C++
// 所有操作O(1)
// 不支持lower_bound() upper_end(); being() end()
```



### bitset

```C++
bitset<10000> b;
~ | & ^;
<< >>;
==  !=;
count();
any() //判读是否至少有一个1
none() //判断是否全为0
set(k, v)
reset()  //将所有数取反
flip() //取反
```



## 动态规划

- 无后效性，动态规划要求求解的子问题不受后续阶段的影响。换言之，动态规划对状态空间的遍历构成一张有向无环图，遍历该有向无环图就是一个拓扑序，有向无环图中每个节点对应不同的状态，图中的边对应状态之间的转移。

### 背包问题

- 01背包

  ```C++
  //考虑前i个物品，且总体积不大于j的所有选法
  int v[N], w[N];
  int f[N][N];
  for(int i = 1; i <= n; i++)
  {
      for(int j = 0; j <= m; j++)
      {
          f[i][j] = f[i - 1][j];
          if(j >= v[i]) f[i][j] = max(f[i][j], f[i - 1][j - v[i]] + w[i]);
      }
  }
  
  //优化
  int f[N];
  for(int i = 1; i <= n; i++)
  {
      for(int j = m; j >= w[i]; j--)
      {
          f[j] = max(f[j], f[j - v[i]] + w[i]);
      }
  }
  
  ```

- 完全背包

  ```C++
  for(int i = 1;  i <= n; i++)
      for(int j = 0; j <= m; j++)
          for(int k = 0; k * v[i] <= j; k++)
              f[i][j] = max(f[i][j], f[i - 1][j - k * v[i]] + k * w[i])
  
  /*优化
  f[i, j] = max(f[i - 1][j], f[i - 1][j - v] + w, f[i - 1][j - 2*v] + 2*w + .....)
  f[i, j -v] = max(          f[i - 1][j - v], f[i - 1][j - 2* v]......)
  */
              
  for(int i = 1; i <= n; i++)
      for(int j = 0; j <= m; j++)
      {
          f[i][j] = f[i - 1][j];
          if(j >= v[i]) f[i][j] = max(f[i][j], f[i][j - v[i]] + w[i]);
      }
  //优化(区别于01背包的遍历顺序)
  for(int i = 1; i <= n; i++)
  {
      for(int j = v[i], j <= m; j++)
      {
          f[j] = max(f[j], f[j - v[i]] + w[i]);
  	}
  }
  ```

- 多重背包

  ```C++
  //将多重背包转化为01背包(二进制优化)
  while(k <= s)
  {
      cnt++;
      v[cnt] = a * k;
      w[cnt] = b * k;
      s -= k;
      k *= 2;
  }
  if(s > 0)
  {
      cnt++;
      v[cnt] = a * s;
      w[cnt] = b * s;
  }
  n = cnt;
  for(int i = 1; i <= n; i++)
  {
      for(int j = m; j >= w[i]; j--)
      {
          f[j] = max(f[j], f[j - v[i]] + w[i]);
      }
  }
  ```

- 分组背包问题

  ```C++
  for(int i = 0; i < n; i++)
  {
      for(int j = m; j >= 0; j--)
      {
          for(int k = 0; k < s[i]; k++)
              if(v[i]][k] <= j) f[j] = max(f[j], f[j - v[i][k]] + w[i][k]);
      }
  }
  ```

### 线性DP

- 数组三角形

- 最长上升子序列

  ```C
  for(int i = 0; i < n; i++)
  {
      f[i] = 1;
      for(int j = 0; j < i; j++)
      {
          if(f[i] > f[j]) f[i] = max(f[i], f[j] + 1)
      }
  }
  
  //求路线
  for(int i = 0; i < n; i++)
  {
      f[i] = 1;
      g[i] = 0;
      for(int j = 0; j < i; j++)
      {
          if(f[i] > f[j]) 
              if(f[i] < f[j] + 1)
              {
                  f[i] = f[j] + 1;
                  g[i] = j;
              }
      }
  }
  for(int i = 0, len = f[k]; i < len; i++)
  {
      printf("%d", g[k]);
      k = g[k];
  }
  ```

- 最长上升子序列II

  ```C
  for(int i = 0; i < n; i++)
  {
      int l = 0, r = len;
      while(l < r)
      {
          int mid = (l + r + 1) / 2;
          if(q[mid] < a[i]) l = mid;
          else r = mid - 1;
      }
      len = max(len, r + 1);
      q[r + 1] = a[i];
  }
  ```

  

- 最长公共子序列

  ```C++
  for(int i = 1; i <= n; i++)
  {
      for(int j = 1; j <= m; j++)
      {
          f[i][j] = max(f[i - 1][j], f[i][j - 1]);
          if(a[i] == b[j]) f[i][j] = max(f[i][j], f[i - 1][j - 1] + 1);
      }
  }
  
  ```

- 编辑距离

  ```C++
  for(int i = 0; i <= m; i++) f[0][i] = i;
  for(int i = 0; i <= n; i++) f[i][0] = i;
  
  for(int i = 1; i <= n; i++)
  {
      for(int j = 1; j <= m; j++)
      {
          f[i][j] = min(f[i - 1][j], f[i][j - 1]);
          if(a[i] == b[j]) f[i][j] = min(f[i][j], f[i - 1][j - 1]);
          else f[i][j] = min(f[i][j], f[i - 1][j - 1] + 1);
  	}
  }
  ```

### 区间DP

- 石子合并

  ```C++
  for(int len = 2; len <= n; len++)
  {
      for(int i = 1; i + len - 1<= n; i++)
      {
          int l = i, r = i + len - 1;
          f[i][j] = 1e8;
          for(int k = i, k < r; k++)
          {
              f[l][r] = min(f[l][r], f[l][k] + f[k + 1][r] + s[r] - s[l - 1]);
          }
      }
  }
  ```

### 计数DP

- 整数划分

  ```C++
  //背包做法
  f[i][j] = f[i - 1][j] + f[i - 1][j - i] + f[i - 1][j - 2i] +... f[i - 1][j - si];
  f[i][j - 1] =           f[i - 1][j - 1] + f[i - 1][j - 2i] +... f[i - 1][j - si];
  
  f[i][j] = f[i - 1][j] + f[i][j - 1];
  
  for(int i = 1; i <=n ;i++)
      for(int j = i; j <= n; j++)
      {
          f[j] = (f[j] + f[j - 1]) % mod;
      }
  
  //计数dp
  //集合：所有总和为i,并且恰好表示成j个数的和的方案
  for(int i = 1; i <= n; i++)
  {
      for(int j = 1; j <= i; j++)
      {
          f[i][j] = (f[i - 1][j] + f[i - j][j]) % mod
      }
  }
  return f[n][1] + .....f[n][n]; 
  ```

### 数位DP

- 数字统计

  ```C++
  
  ```



## 搜索和图论

### DFS

### BFS

### 邻接表

```C++
int h[N], e[N], ne[N], idx;
init()
{
    memset(h, -1, sizeof h);
    idx = 0;
}

void add(int a, int b)
{
    e[idx] = b, ne[idx] = h[a], h[a] = idx, idx++;
}
```

### 拓扑排序

```C++
bool toposort()
{
    int hh = 0, tt = -1;
    for(int i = 1; i <= n; i++)
    {
        if(!d[i]) q[++tt] = i;
	}
    while(hh <= tt)
    {
        int t = qq[hh++];
        for(int i = h[t]; i != -1; i = ne[i])
        {
            int j = e[i];
            d[j]--;
            if(d[j] == 0) q[++tt] = j;
		}
    }
    return tt == n - 1;
}
```

### 单源最短路

```C++
//所有边权均为正数
//朴素Dijkstra O(n^2)
//适合稠密图
int g[N][N];
int dist[N];
bool st[N];

int dijkstra()
{
    memset(dist, 0x3f, sizeof dist);
    dist[1] = 0;
    for(int i = 0; i < n; i++)
    {
        int t = -1;
        for(int j = 1; j <= n; j++)
        {
            if(!st[j] && (t == -1 || dist[t] > dist[j]))
                t = j;
        }
        st[t] = 1;
        for(int j = 1; j <= n; j++)
        {
            dist[j] = min(dist[j], dist[t] + g[t][j]);
        }
    }
    if(dist[n] == 0x3f) return -1;
    return dist[n];   
}
```

```C++
//堆优化版的Dijkstra 
//适合稀疏图
//O(mlogn)
typedef pair<int, int> PII;
int h[N], w[N], e[N], ne[N], idx;
int dist[N];
bool st[N];

int dijkstra()
{
    memset(dist, 0x3f, sizeof dist);
    dist[1] = 0;
    st[1] = true;
    priority_queue<PII, vector<PII>, greater<PII>> heap;
    heap.push({0, 1});
    while(heap.size())
    {
        auto t = heap.top();
        heap.pop();
        int ver = t.second, dis = t.first;
        if(st[ver]) continue;
        st[ver] = true;
        for(int i = h[ver]; i != -1; i = ne[i])
        {
            int j = e[i];
            if(dist[j] > dis + w[i])
            {
                dist[j] = dis + w[i];
                heap.push({dist[j], j});
            }
        }
    }
    if(dist[n] == 0x3f3f3f3f) return -1;
    return dist[n];
}
```

```C++
//存在负权边情况
//Bellman- Ford算法
//O(nm)
int dist[N], backup[N];
struct Edge
{
    int a, b, w;
}edges[M];

int bellman_ford()
{
    memset(dist, 0x3f, sizeof dist);
    dist[1] = 0;
    for(int i = 0; i < k; i++)
    {
        //避免后效性
        memcpy(backup, dist, sizeof dist);
        for(int j = 0; j < m; j++)
        {
            int a = edges[j].a;
            int b = edges[j].b;
            int w = edges[j].w;
            dist[b] = min(dist[b], backup[a] + w);
        }
    }
    // 两个正无穷的点之间的边是负的情况
    if(dist[n] > 0x3f3f3f3f / 2) return -1;
    return dist[n];
}
```

```C++
//SPFA算法 是对Bellman-ford进行优化
一般O(m) 最坏O(nm)
//利用广度优先算法进行优化
int spfa()
{
    memset(dist, 0x3f, sizeof dist);
    dist[1] = 0;
    st[1] = true;
    
    queue<int> q;
    q.push(1);
    
    while(q.size())
    {
        int t = q.front();
        q.pop();
        st[t] = false;
        
        for(int i = h[t]; i != -1; i = ne[i])
        {
            int j = e[i];
            if(dist[j] > dist[t] + w[i])
            {
                dist[j] = dist[t] + w[i];
                if(!st[j])
                {
                    q.push(j);
                    st[j] = true;
                }
            }
        }
    }
    if(dist[n] == 0x3f3f3f3f) return -1;
    return dist[n];
}

```

```C++
//拓展 spfa判断负环
//维护一个cnt数组 cnt[i] >= n则有负环
bool spfa()
{
    memset(dist, 0x3f, sizeof dist);
    queue<int> q;
    for(int i = 1; i <= n; i++)
    {
        st[i] = true;
        q.push(i);
    }
    while(q.size())
    {
        int t = q.front();
        q.pop();
        st[t] = false;
        
        for(int i = h[t]; i != -1; i = ne[i])
        {
            int j = e[i];
            if(dist[j] > dist[t] + w[i])
            {
                dist[j] = dist[t] + w[i];
                cnt[j] = cnt[t] + 1;
                if(cnt[j] >= n) return true;
                if(!st[j])
                {
                    q.push(j);
                    st[j] = true;
                }        
            }
        }
    }
    return false;
    
}
```

### 多源汇最短路

```C++
//Floyd  
//O(n^3)
void floyd()
{
    for(int k = 1; k <= n; k++)
    {
        for(int i = 1; i <= n; i++)
        {
            for(int j = 1; j <= n; j++)
            {
                d[i][j] = min(d[i][j], d[i][k] + d[k][j]);
            }
        }
    }
}

```

### 最小生成树

```C++
//Prim
//适合稠密图
//O(n^2)
int prim()
{
    memset(dist, 0x3f, sizeof dist);
    int res = 0;
    for(int i = 0; i < n; i++)
    {
        int t = -1;
        for(int j = 1; j <= n; j++)
        {
            if(!st[j] && (t == -1 || dist[t] < dist[j]))
                t = j;
            if(i && dist[t] == INF) return INF;
            if(i) res += dist[t];
            
            for(int j = 1; j <= n; j++) 
                dist[j] = min(dist[j], g[t][j]);
            st[t] = true;
        }
    }
    return res;
}
```

```C++
//Kruskal
//适合稀疏图
//O(mlogm)
struct Edges
{
    int a, b, w;
    operator <(const Edges& W) const
    {
        return w < W.w;
    }
}edegs[m];

sort(edegs, edges + m);
for(int i = 1; i <= n; i++) p[i] = i;

int res = 0, cnt = 0;
for(int i = 0; i < m; i++)
{
    int a = edges[i].a;
    int b = edges[i].b;
    int w = edges[i].w;
    a = find(a), b = find(b);
    if(a != b)
    {
        res += w;
        cnt++;
        p[a] = b;
    }
}
if(cnt < n - 1) return -1;
else return res;
```

### 二分图

```C++
//染色法 
//O(n + m)
//二分图 当且仅当图中不含奇数环
bool flag = false;
bool dfs(int u, int c)
{
    for(int i = h[u]; i != -1; i = ne[i])
    {
        int j = e[i];
        if(!color[j])
        {
            return dfs(j, 3 - c);
        }
        else if(color[j] == c) return false;
    }
    return true;
}
for(int i = 0; i <= n; i++)
{
    if(!color[i])
    {
        if(!dfs(i, 1))
        {
            flag = false;
            break;
        }
    }
}
```

```C++
//匈牙利算法
//O(mn)
bool find(int x)
{
    for(int i = h[x]; i != -1; i = ne[i])
    {
        int j = e[i];
        if(!st[j])
        {
            st[j] = true;
            if(match[j] == 0 || find(match[j]))
            {
                match[j] = x;
                return true;
            }
        }
    }
    return false;
}

for(int i = 1; i <= n1; i++)
{
    memset(st, false, sizeof st);
    if(find(i)) res++;
    printf("%d\n", res);
}
```



## 线段树

```C++
//线段树是一种二叉搜索树，将一段区间划分为若干单位区间，每一个节点都存储着一个区间
//支持单点修改，区间求和，区间求最值，区间修改
struct Node {
    int l, r;
    int v;
} tree[N * 4];
//建树
void build(int u, int l, int r) {
    tree[u] = {l, r};
    if(l == r) return;
    int mid = l + r >> 1;
    build(u << 1, l, mid);
    build(u << 1 | 1, mid + 1, r);
}
//更新
void push_up(int u) {
    tree[u].v = max(tree[u << 1].v, tree[u << 1 | 1]. v);
}
//查询
int query(int u, int l, int r) {
    if(tree[u].l >= l && tree[u].r <= r) return tree[u].v;
    int mid = tree[u].l + tree[u].r >> 1;
    int v = 0;
    if(l <= mid) v = query(u << 1, l, r);
    if(r > mid) v = max(v, query(u << 1 | 1, l, r));
    return v;
}
//修改
void modify(int u, int x, int v) {
    if(tree[u].l == x && tree[u].r == x) tree[u].v = v;
    else {
        int mid = tree[u].l + tree[u].r >> 1;
        if(x <= mid) modify(u << 1, x, v);
        else modify(u << 1 | 1, x, v);
        tree[u].v = max(tree[u << 1].v , tree[u << 1 | 1].v);
    }
}
```









