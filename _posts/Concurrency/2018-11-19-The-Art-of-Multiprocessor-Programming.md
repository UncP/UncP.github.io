## Chapter 2

### Good `Lock` Algorithm

1. **Mutual exclusion**: critical section 不会发生重叠，即 $CS_{A}^{k}→CS_{B}^{j}$ or $CS_{B}^{j}→CS_{A}^{k}$
2. **Freedom From Deadlock**: 如果某个线程尝试加锁，那么某个线程会成功加锁。如果一个线程尝试加锁，但是始终不成功，那么一定是其他线程在完成无尽的 critical section
3. **Freedom From Starvation**: 所有尝试加锁的线程最终都会成功（Lockout Freedom）

&nbsp;

### 2-Thread Solutions

#### `LockOne` Class

```java
class LockOne implements Lock {
  private boolean[] flag = new boolean[2]; // thread-local index, 0 or 1

  public void lock() {
    int i = ThreadID.get();
    int j = 1 - i;
    flag[i] = true;        // write true
    while (flag[j]) {}     // read
  }

  public void unlock() {
    int i = ThreadID.get();
    flag[i] = false;
  }
}
```

满足1，

> 假设不满足1，那么 $lock_{A} → lock_{B}$ 成立，可以得出 $write_{A}(flag[A] = true) → read_{A}(flag[B] == false) → write_{B}(flag[B] = true) → read_{B}(flag[A] == false)$，即 $write_{A}(flag[A] = true) → read_{B}(flag[A] == false)$ ，矛盾，write true 操作后所有 read 操作都应该为 true，所以一定有 $unlock_{A}$ 操作发生在 $lock_{A}$ 和 $lock_{B}$ 之间。

但是不满足2，

> 两个线程在 read 操作之前都进行了 write true 操作，所以 read 结果一直为真。

&nbsp;

#### `LockTwo` Class

```java
class LockTwo implements Lock {
  private int victim;

  public void lock() {
    int i = ThreadID.get();
    victim = i;            // let the other go first
    while (victim == i) {} // wait
  }

  public void unlock() {}
}
```

满足1，

> $CS_{A}: write_{A}(victim = A) → read_{A}(victim == B)$
>
> $CS_{B}: write_{B}(victim = B) → read_{B}(victim == A)$
>
> 假设不满足1，那么 $lock_{A} → lock_{B}$ 成立，可以得出 $write_{A}(victim = A) → write_{B}(victim = B) → read_{A}(victim == B)$，但是 $victim == B$ 与 $CS_{B}$ 矛盾。

但是不满足2，

> 如果线程 B 在线程 A 之后执行，那么 B 将死锁。

&nbsp;

`LockOne` 与 `LockTwo` 互为补充。

&nbsp;

#### The Peterson Lock

```java
class Peterson implements Lock {
  private boolean[] flag = new boolean[2]; // thread-local index, 0 or 1
  private int victim;
    
  public void lock() {
    int i = ThreadID.get();
    int j = 1 - i;
    flag[i] = true; // I’m interested
    victim = i;     // you go first
    while (flag[j] && victim == i) {} // wait
  }

  public void unlock() {
    int i = ThreadID.get();
    flag[i] = false; // I’m not interested
  }
}
```

满足1，

> 假设不满足1，那么 $lock_{A} → lock_{B}$ 成立，可以得出 $write_{A}(victim = A) → write_{B}(victim = B)$，即 $write_{B}(victim = B) → read_{B}(flag[A] == false)$，所以 $write_{A} (flag[A] = true) → write_{A} (victim = A) → write_{B}(victim = B) → read_{B}(flag[A] == false)$，即 $write_{A} (flag[A] = true) → read_{B}(flag[A] == false)$，矛盾。

满足3，

> 如果不满足3，那么对于任意线程（假设是 A）来说一定是在 while 循环中，即  $flag[B] == true$ 或 $victim == A$，那么对于 B 来说当它进入 lock() 时，会将 victim 设为 B，并且不会改变，所以 A 一定会被唤醒，矛盾。

满足3的同时一定满足2。

&nbsp;

### The Filter Lock

```java
class Filter implements Lock {
  int[] level;
  int[] victim;
  
  public Filter(int n) {
    level = new int[n];
    victim = new int[n]; // use 1..n-1
    for (int i = 0; i < n; i++) {
      level[i] = 0;
    }
  }

  public void lock() {
    int me = ThreadID.get();
    for (int i = 1; i < n; i++) { // attempt level i
    level[me] = i;
    victim[i] = me;
    // spin while conflicts exist
    while ((∃k != me) (level[k] >= i && victim[i] == me)) {}
  }
      
  public void unlock() {
    int me = ThreadID.get();
    level[me] = 0;
  }
}
```



### The Bakery Lock

* 二阶段保证公平性（fairness）
  1. Doorway: 执行间隔是有限的
  2. Waiting: 执行间隔可能是无限的

> Frist Come Frist Served: 先结束 Doorway 阶段的线程先加锁

```java
class Bakery implements Lock {
  boolean[] flag;
  Label[] label;

  public Bakery (int n) {
    flag = new boolean[n];
    label = new Label[n];
    for (int i = 0; i < n; i++) {
      flag[i] = false;
      label[i] = 0;
    }

  public void lock() {
    int i = ThreadID.get();
    flag[i] = true;
    label[i] = max(label[0], ..., label[n-1]) + 1;
    while ((∃k != i)(flag[k] && (label[k],k) << (label[i],i))) {}
  }

  public void unlock() {
    flag[ThreadID.get()] = false;
  }
}
```

