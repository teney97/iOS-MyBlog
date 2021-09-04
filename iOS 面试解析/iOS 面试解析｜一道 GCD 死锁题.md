## iOS 面试解析｜一道 GCD 死锁题

分别在 mainThread 执行 test1 和 test2 函数，问执行情况如何？

```swift
  func test1() {
      DispatchQueue.main.sync { // task1
          print("1") 
      }
  }

  func test2() {
      print("1")
      let queue = DispatchQueue.init(label: "thread")
      queue.async { // task1
          print("2")
          DispatchQueue.main.sync { // task3
              print("3")
              queue.sync { // task4
                  print("4")
              }
          }
          print("5")
      }
      print("6")
      queue.async { // task2
          print("7")
      }
      print("8")
  }
```

1. 死锁。
   1. mainThread 当前正在执行 test1 函数。
   2. 这时候使用 sync 函数往 mainQueue 中提交 task1 以同步执行，需要 task1 执行完毕后才会返回。
   3. 由于队列 FIFO，要想从 mainQueue 取出 task1 放到 mainThread 执行，需要先等待上一个 task 也就是 test1 函数先执行完，而 test1 此时又被 sync 阻塞，需要 sync 函数先返回。因此 test1 与 task1 循环等待，产生死锁。
2. 打印 1、6、8、2、3，然后死锁。
   1. 创建一个 serialQueue。使用 async 函数往指定队列提交 task 以异步执行会直接返回，不会阻塞，因此打印 1、6、8，并且 mainThread 执行完 test2。
   2. 从 serialQueue 中取出 task1 放到一条 childThread 执行，因为是 serialQueue 所以 task2 需要等待 task1 执行完毕才会执行。 执行 task1，打印 2；
   3. 使用 sync 函数往 mainQueue 提交 task3。**此时 task1 被阻塞，需要等待 task3 执行完毕**，才会接下去打印 5；
   4. mainThread 当前没有在执行 task，因此执行 task3，打印 3；
   5. 接着，使用 sync 往 serialQueue 中提交 task4，**此时 task3 被阻塞，需要等待 task4 执行完毕**；
   6. 此时该 childThread 正在执行 task1，**因此 task4 需要等待 task1 先执行完毕**。
   7. 此时，task1 在等待 task3，task3 在等待 task4，task4 在等待 task1。循环等待，产生死锁。

使用 GCD 的时候，我们一定要注意死锁问题，不要使用 `sync 函数` 往 `当前 serialQueue` 中添加 task，否则会卡住当前 serialQueue，产生死锁。