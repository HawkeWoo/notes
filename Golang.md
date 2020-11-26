## Channel

死锁一

```go
func main() {
   ch := make(chan int)
   ch <- 1		// ch为0缓存channel 这里会阻塞，然后就死锁了
   println("get val: ", <-ch)
}
```

```go
func main() {
	ch := make(chan int)
	go func() {
		println("go into goroutine")
		ch <- 1
	}()		// 阻塞之后挂起当前协程，所以不会死锁
	println("get val: ", <-ch)	
}
```

```go
func main() {
   ch := make(chan int, 1)	// 或者使用缓冲channel
   ch <- 1
   println(<-ch)
}
```

死锁二

```go
func main() {
   ch := make(chan int)
   go func() {
      for i:=0; i<3; i++{
         println("put value into channel: ", i)
         ch <- i
      }
   }()
   for v := range ch{
      println("get value from channel: ", v)		// 这里拿完所有数据后，因为发送端没有关闭channel，这个for循环一直监听channel，发生死锁
   }
}
```

```go
func main() {
   ch := make(chan int)
   go func() {
      for i:=0; i<3; i++{
         println("put value into channel: ", i)
         ch <- i
      }
      close(ch)		// 正确做法在发送端关闭channel
   }()
   for v := range ch{
      println("get value from channel: ", v)
   }
}
```



从关闭的或者空的channel只能读到零值, 若不关闭则一直阻塞

```go
func main() {
   ch := make(chan bool)
   close(ch)
   println(<-ch)	// output：false
}
```



汇总不同goroutine计算结果

```go
func main() {
   ch := make(chan int)
   res := 0
   go func() {
      t := 0
      for i:=0; i<10; i++{
         t += i
      }
      println("go func1 cal value: ", t)
      ch <- t
   }()

   go func() {
      t := 0
      for i:=0; i<5; i++{
         t += i
      }
      println("go func1 cal value: ", t)
      ch <- t
   }()


   go func() {
      t := 0
      for i:=5; i<10; i++{
         t += i
      }
      println("go func1 cal value: ", t)
      ch <- t
   }()

   for i:=0; i<3; i++{
      res += <- ch
   }
   
   println("result: ", res)
}
```



广播通知

```go
func main() {
   exit := make(chan int)
   done := make(chan int)
   for i:=0; i<10; i++{
      go func(i int) {
         for {
            select {
              //在 select 中使用发送操作并且有 default 可以确保发送不被阻塞！如果没有 default，select 就会一直阻塞, 等待直到其中一个可以处理。
            case <- exit:	// 从已关闭的 channel 读取数据时总是非阻塞的，当exit关闭的时候 其他所有的协程从exit channel收到一个零值
               fmt.Println("worker ", i, " exit")
               done <- 1
               return
            case <- time.After(time.Second):	// 表示time.Duration长的时候后返回一条time.Time类型的通道消息。只有在本次select 操作中会有效， 再次select 又会重新开始计时
               fmt.Println("worker ", i, " still working")
            }
         }
      }(i)
   }
   time.Sleep(3 * time.Second)
   close(exit)
   for i:=0; i<10; i++{
      <- done
   }
   fmt.Println("all worker done")
}
```

互斥量

```go
func main() {
   mutex := make(chan struct{}, 1)
   counter := 0
   add := func(){
      mutex <- struct{}{}    // lock, 如果mutex内有元素，则阻塞，等待元素被释放，信号量同理，信号量channel的大小可以不为1
      counter++
      <- mutex   //unlock
   }

   for i:=0; i<1000; i++{
      go func() {
         add()
      }()
   }
   time.Sleep(time.Second)
   println(counter)
}
```



关闭channel：

- 不要在读取端关闭 channel ，因为写入端无法知道 channel 是否已经关闭，往已关闭的 channel 写数据会 panic ；
- 有多个写入端时，不要再写入端关闭 channle ，因为其他写入端无法知道 channel 是否已经关闭，关闭已经关闭的 channel 会发生 panic ；
- 如果只有一个写入端，可以在这个写入端放心关闭 channel 。
- channel重复关闭会panic



一写多读

```go
func main() {
   ch := make(chan int)
   go func() {
      for i:=0; i<10; i++{
         ch <- i
      }
      close(ch)
   }()

   for i:=0; i<3; i++{
      go func(i int) {
         for v := range ch{
            println("receiver ", i, " get value: ", v)
         }
      }(i)
   }

   time.Sleep(time.Second)
   println("end")
}
//receiver  0  get value:  0
//receiver  2  get value:  1
//receiver  2  get value:  3
//receiver  2  get value:  4
//receiver  0  get value:  2
//receiver  1  get value:  5
//receiver  0  get value:  7
//receiver  1  get value:  8
//receiver  2  get value:  6
//receiver  0  get value:  9
//end
```



多写一读

```go
func main() {
   ch := make(chan int)
   exit := make(chan struct{})	//设置一个额外的channel ，由读取端通过关闭来通知写入端任务完成不要再继续再写入数据了, 多读多写同理，通过额外的channel来通知读写端

   //发送端
   for i:=0; i<3; i++{
      go func(id int) {
         for i:=0;;i++{
            select {
            case<-exit:
               println("sender ", id, " exit")
               return
            default:
               ch<-id*1000+i
            }
         }
      }(i)
   }

   // 读取端
   go func() {
      counter := 0
      for v := range ch{
         println("receive val: ", v)
         counter++
         if counter > 20{
            close(exit)
            return //不要忘记退出 不然panic
         }
      }
   }()
   
   time.Sleep(3*time.Second)
}
```



## Sync

#### Once

Once 是一个可以被多次调用但是只执行一次，若每次调用Do时传入参数f不同，但是只有第一个才会被执行。

```go
func main() {
   var once sync.Once
   onceBody := func(){
      println("Execute Once")
   }
   for i:=0; i<10; i++{
      go func(i int) {
         println("goroutine ", i)
         once.Do(onceBody)
      }(i)
   }
   time.Sleep(time.Second)
   println("done")
}
```

#### WaitGroup

wait group 用来等待一组goroutines的结束，在主Goroutine里声明，并且设置要等待的goroutine的个数，每个goroutine执行完成之后调用 Done，最后在主Goroutines 里Wait即可。

```go
func main() {
   wg := &sync.WaitGroup{}
   for i:=0; i<10; i++{
      wg.Add(1)
      go func(i int) {
         defer wg.Done()
         println("goroutine ", i)
      }(i)
   }
   wg.Wait()
}
```

### Mutex

互斥锁

```go
type Bank struct {
   sync.Mutex
   balance map[string] float64
}
// 关于锁要不要声明为指针
// 如果不声明为指针，则在对象拷贝的过程中锁也会被拷贝
// 如果声明为指针，那对象拷贝之后出现copy2，那么copy2被锁的时候原对象也会被锁住，这种情况下锁不应该声明在结构体内部，如果不想copy2和原对象同时被锁，则锁不能声明为指针
// 用指针是一种懒惰的行为，自己需要对代码有明确的了解，结构体是否在业务逻辑中会不会被拷贝，

func (bank *Bank) In(account string, value float64){
   bank.Lock()
   defer bank.Unlock()
   if _, ok := bank.balance[account]; !ok{
      bank.balance[account] = 0.0
   }
   bank.balance[account]+=value
}


func (bank *Bank) Out(account string, value float64) error{
   bank.Lock()
   defer bank.Unlock()
   if _, ok := bank.balance[account]; !ok{
      bank.balance[account] = 0.0
   }
   v := bank.balance[account]
   if v < value{
      return errors.New("not enough balance")
   }
   bank.balance[account] -= value
   return nil
}
```

### RWMutex

读写锁，是`sync.Mutex`的一种变种，RWMutex来自于计算机操作系统非常有名的读者写者问题。
RWMutex目的是为了能够支持多个并发协程同时读取某一个资源，但只有一个并发协程能够更新资源。也就是说读和写是互斥的，写和写也是互斥的，读和读是不互斥的。

总结起来如下

1. 当有一个协程在读的时候，所有写的协程必须等到所有读的协程结束才可以获得锁进行写操作。
2. 当有一个协程在读的时候，所有读的协程不受影响都可以进行读操作。
3. 当有一个协程在写的时候，所有读、写的协程必须等到写的协程结束才可以获得锁进行读、写操作。

### Cond

信号量，指的是同步条件变量，一般需要与互斥锁组合使用，本质上是一些正在等待某个条件的协程的同步机制。



### Pool

临时对象池



### Map

带锁的map



## 方法

1. 结构指针接收者，顾名思义，会在方法内部改变该结构内部变量的值
2. 结构值接收者，在方法内部对变量的改变不会影响该结构。
3. 对于指针接收者，如果你调用的是值方法，即使你是指针调用者，也不会改变你的结构内的变量值
4. 对于值接收者，如果你调用的是指针方法，即使你是值调用者，也会改变你的结构内的变量值

3，4点概括就是说只要方法的接受者是指针，则可以在方法中改变结构体内的变量值



## 接口

1. 对于指针接受者，如果传入值对象，会panic
2. 对于值接受者，如果传入指针对象，会自动解引用

