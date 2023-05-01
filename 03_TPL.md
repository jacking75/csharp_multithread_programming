# Task
- 직접 스레드를 만들지 않고, 스레드풀을 이용한다.
- Thread 보다 Task를 사용하는 것이 더 좋다.  
- 일시 대기를 할 때는 `Task.Delay`를 사용한다
- [소개](https://jacking75.github.io/csharp_TPL/)  
- [(일어)TPL 입문](http://xin9le.net/tpl-intro  )  


## Task 병렬 라이브러리
- 태스크 라이브러리와 병렬 라이브리의 모음.
- 태스크: System.Threading.Tasks.Task를 중심으로 한 라이브러리.
- 병렬: System.Threading.Tasks.Parallel를 중심으로 한 라이브러리.
  
  
## Task 만드는 법
- TaskFactory를 이용하여 만드는 법
    - Task.Factory.StartNew 메소드를 이용하여 만든다
    - 이 경우 오브젝트를 취득한 시점에서 태스크 처리는 시작하고 있다.
    - 인수에는 Action을 지정하면 반환 값은 없다.Func을 지정하면 반환 값이 있는 태스크가 된다.
    ```
    Task t = Task.Factory.StartNew(() => ...);                     // Action
    Task t = Task.Factory.StartNew((obj) => ..., T);               // Action<T>
    Task<T> t = Task<T>.Factory.StartNew(() => { return T });      // Func<T>
    Task<T> t = Task<T>.Factory.StartNew(obj) => { return T }, A); // Func<A, T>
    ```  
    ```
    Task<List<string>> t = Task<List<string>>.Factory.StartNew(
                                  i => 
                                  { 
                                     return new List<string>(); 
                                   }, 2000 );

    ```
  
  
## Task 대기
태스크 오브젝트의  Wait 메소드   
스레드와 비슷하게 Wait를 호출한 태스크가 종료할 때까지 대기한다.  
```
Task<List<string>> t = Task<List<string>>.Factory.StartNew(
                                  i => 
                                  { 
                                     return new List<string>(); 
                                   }, 2000 );
t.Wait();
```
  
WaitAll 메소드를 이용하여 지정한 태스크 모두가 끝날 때까지 기다린다.  
```
Task t1 = Task.Factory.StartNew(() => Console.WriteLine("Task 1"));
Task t2 = Task.Factory.StartNew(() => Console.WriteLine("Task 2"));
Task t3 = Task.Factory.StartNew(() => Console.WriteLine("Task 3"));

Task.WaitAll(new Task[]{ t1, t2, t3 });
```
  
```
using System;
using System.Threading.Tasks;

class Program
{
  static void Main(string[] args)
  {
    // 서브 태스크
    var task = Task.Factory.StartNew(() =>
      {
        for (int i = 0; i < 100; i++) Console.Write('B');
      });

    // 메인 태스크
    for (int i = 0; i < 100; i++) Console.Write('A');

    task.Wait();
  }
}
```
  
  
  
## Task 바로 실행 - Task.Run
.NET 4.5에서 태스크 시작 방법에 새로운 메소드가 생겼다.  
`Task.Run(...)`  
   
```
class TaskSamples05 : IExecutable
{
    async Task RunAsync()
    {
      int result = await Task.Run(() => 200);
      Output.WriteLine(result);
    }

    public void Execute()
    {
      // 인수에 Action을 지정(가장 간단한 패턴)
      Task.Run(() => Output.WriteLine("Task.Run(Action)")).Wait();

      // 인수에 Func을 지정
      var task1 = Task.Run(() => 100);
      Output.WriteLine(task1.Result);

      RunAsync();
      
      var task2   = Task.Run(() => 300);
      var awaiter = task2.GetAwaiter();
      awaiter.OnCompleted(() =>
        {
          Output.WriteLine(awaiter.GetResult());
        }
      );

      task2.Wait();     

      // Task.Run 메소드에는 Action, Func을 지정하고 또CancellationToken을 지정할 수도 있다.
      var tokenSource = new CancellationTokenSource();
      var token       = tokenSource.Token;

      var printDot = new Action(() =>
        {
          while (true)
          {
            if (token.IsCancellationRequested)
            {
               Output.WriteLine(string.Empty);
               break;
            }

            Output.Write(".");
            Thread.Sleep(TimeSpan.FromMilliseconds(500));
          }
        }
      );

      var task3 = Task.Run(printDot, token);
      
      Thread.Sleep(TimeSpan.FromSeconds(3));
      tokenSource.Cancel();

      task3.Wait();
      tokenSource.Dispose();
    }
 }
}
```
  


## 이어서 다음 작업 하기
  
```
using System;
using System.Threading.Tasks;

class Program
{
  private static void notify(Task t)
  {
    Console.WriteLine();
    Console.WriteLine("sub task {0} done", t.Id);
  }

  static void Main(string[] args)
  {
    var task = Task.Factory.StartNew(() =>
      {
        for (int i = 0; i < 100; i++) Console.Write('B');
      });
    task.ContinueWith(notify);

    for (int i = 0; i < 100; i++) Console.Write('A');

    task.Wait();
  }
}
```
    
```
// 람다식
task.ContinueWith((t) =>
{
  Console.WriteLine();
  Console.WriteLine("sub task {0} done", t.Id);
});
```  
  
```
Task<int> t1 = new Task<int>(() => { return 1; });
Task<int> t2 = new Task<int>(() => { return 2; });
Task<int> t3 = new Task<int>(() => { return 3; });

Task<int>[] tasks = new Task<int>[]{ t1, t2, t3 };

// 복수의 연속 태스크가 모두 완료한 후에 연속하는 태스크를 만든다
Task<int> continuationTask = Task<int>.Factory.ContinueWhenAll(
                               tasks, 
                               (antecedents) => { return antecedents.Sum(i => i.Result); }
                             );

// 연속 태스크를 시작
tasks.ToList().ForEach(t => t.Start());

// 연속 태스크 결과를 표시
Console.WriteLine(continuationTask.Result);
```  
   
```
Task A = Task.Factory.StartNew(() => { ParallelLogicA(); });
Task B = Task.Factory.StartNew(() => { ParallelLogicB(); });

Task.Factory.ContinueWhenAll( new Task[] { A, B }, _ =>
                                          {
                                             AdditionalLogic();
                                          });
```  
  
  
   
## 복수의 Task 완료 대기
  
```
class Program
{
  static void Main(string[] args)
  {
      var task1 = Task.Factory.StartNew(() =>
      {
        for (int i = 0; i < 100; i++) Console.Write('A');
      });
    
      var task2 = Task.Factory.StartNew(() =>
      {
        for (int i = 0; i < 200; i++) Console.Write('B');
      });
      
      var task3 = Task.Factory.StartNew(() =>
      {
        for (int i = 0; i < 300; i++) Console.Write('C');
      });

    Task.WaitAny(task1, task2, task3);
    Console.WriteLine();
    Console.WriteLine("stopped one task");

    Task.WaitAll(task1, task2, task3);
    Console.WriteLine();
    Console.WriteLine("stopped all tasks");
  }
}
```
  

## 반환 값을 가지는 Task
  
```
Task<DayOfWeek> rootTask = new Task<DayOfWeek>(() =>
                                         DateTime.Now.DayOfWeek);
Task<string> continuationTask = rootTask.ContinueWith((antecedent) =>
                  {
                      return antecedent.Result.ToString();
                  });

rootTask.Start();

Console.WriteLine(continuationTask.Result);
```
  
  
  
## 복수의 Task에서 예외 발생
  
```
Task<int> t = Task<int>.Factory.StartNew(() => { return 1; });
Task<int> c = t.ContinueWith((antecedent) => { throw new InvalidOperationException(""); });

try
{
    t.Wait();
    c.Wait();
}
catch (AggregatedException aggEx)
{
    foreach (Exception ex in aggEx.InnerExceptions)
    {
        Console.WriteLine(e.Message);
    }
}
```
   
  
  
## 부모-자식 Task 예
  
```
public class TaskSamples03 : IExecutable
{
	public void Execute()
	{
	  // 単純な入れ子のタスクを作成.
	  //
	  Console.WriteLine("外側のタスク開始");
	  Task t = new Task(ParentTaskProc);
	  t.Start();
	  t.Wait();
	  Console.WriteLine("外側のタスク終了");
	  
	}

	void ParentTaskProc()
	{
	  PrintTaskId();

	  //
	  // 明示的に、TaskCreationOptionsを指定していないので
	  // 以下の入れ子タスクは、「デタッチされた入れ子タスク」
	  // として生成される。
	  //
	  Task detachedTask = new Task(ChildTaskProc, TaskCreationOptions.None);
	  detachedTask.Start();
	  
	  //
	  // 以下のWaitをコメントアウトすると
	  // 出力が
	  //     外側のタスク開始
	  //      Task Id: 1
	  //     内側のタスク開始
	  //      Task Id: 2
	  //     外側のタスク終了
	  //
	  // と出力され、「内側のタスク終了」の出力がされないまま
	  // メイン処理が終了したりする。
	  //
	  // これは、2つのタスクが親子関係を持っていないため
	  // 別々で処理が行われているからである。
	  //
	  detachedTask.Wait();
	}

	void ChildTaskProc()
	{
	  Console.WriteLine("内側のタスク開始");
	  PrintTaskId();
	  Thread.Sleep(TimeSpan.FromSeconds(2.0));
	  Console.WriteLine("内側のタスク終了");
	}

	void PrintTaskId()
	{
	  //
	  // 現在実行中のタスクのIDを表示.
	  //
	  Console.WriteLine("\tTask Id: {0}", Task.CurrentId);
	}
}
```
   
```
public class TaskSamples04 : IExecutable
{
	public void Execute()
	{	  
	  // 親子関係を持つ子タスクを作成.
	  //
	  Console.WriteLine("親のタスク開始");
	  Task t = new Task(ParentTaskProc);
	  t.Start();
	  t.Wait();
	  Console.WriteLine("親のタスク終了");
	}

	void ParentTaskProc()
	{
	  PrintTaskId();
	  
	  //
	  // 明示的にTaskCreationOptionsを指定して
	  // アタッチされた入れ子タスクを指定する。
	  //
	  Task childTask = new Task(ChildTaskProc, TaskCreationOptions.AttachedToParent);
	  childTask.Start();
	  
	  //
	  // 「デタッチされた入れ子タスク」と違い、親タスクにアタッチされた入れ子タスクは
	  // 明示的にWaitをしなくても、親のタスクが子のタスクの終了を待ってくれる。
	  //
	}

	void ChildTaskProc()
	{
	  Console.WriteLine("子のタスク開始");
	  PrintTaskId();
	  Thread.Sleep(TimeSpan.FromSeconds(2.0));
	  Console.WriteLine("子のタスク終了");
	}

	void PrintTaskId()
	{
	  //
	  // 現在実行中のタスクのIDを表示.
	  //
	  Console.WriteLine("\tTask Id: {0}", Task.CurrentId);
	}
}
```
       
  
  
## 예외처리와 트리거 메소드
- 예외 처리와 트리거 메소드
- 태스크 내부에서 발생한 예외는 내부에서 AggregateException로 모아서 처리 된다.
- 태스크에서는 아래의 메소드/프로퍼티를 호출한 타이밍에서 예외는 throw 된다. (트리거 메소드/프로퍼티）
    - Wait
    - Result
- 예외를 잡을지 말지는 유저에는 맡긴다.
- 태스크를 시작 시킨 후 Wait도 Result도 호출하지 않은 경우는 최종적으로 예외는 throw 되지 않는다.
- 태스크 내부에서 발생하여 AggregateException 상태로 관리된 채로 된다.
- 기본적으로 반환 값을 가진 태스크의 경우는 Result를 호출하지 않으면 반화 값을 얻지 않으므로 문제 없다.
- AggregateException의 InnerExceptions 프로퍼티에서 태스크 내부에서 발생한 예외를 얻을 수 있다.
    
- 태스크 캔슬 처리
- 태스크에 캔슬 기능을 부여할 때는 CancellationToken을 이용한다.
- CancellationToken은 CancellationTokenSource에서 얻는다.
  
```
CancellationTokenSource tokenSource = new CancellationTokenSource();
CancellationToken       token       = tokenSource.Token;

// 태스크를 작성하고 토큰을 넘기는 오버로드를 사용한다
Task t = Task.Factory.StartNew(DoSomeWork, token);
```
   
- 캔슬을 할 때는 CancellationTokenSource.Cancel을 호출한다. 호출되면 토큰이 캔슬을 인식한다.
- CancellationTokenSource.Cancel을 호출하는 것만으로는  캔슬 처리는 완료하지 않는다.
- 뒤에는 token.IsCancellationRequested 나 ThrowIfCancellationRequested를 이용하여 태스크 내부에서 처리를 캔슬 시킨다.
- 태스크 내부에서 캔슬 처리가 발생하면 OperationCanceledException이 발생한다.
- OperationCanceledException을 얻으려면 트리거 메소드(Wait Or Result）을 호출한다.
- 태스크 내부에서 캔슬 처리가 발생하면 태스크의 IsCanceled가 True로 된다. IsCanceled가 True가 되는 타이밍은 트리거 메소드가 호출될 때가 된다.
  
  
  
## CancellationTokenSource.CancelAfter
.NET 4.5에서 CancellationTokenSource 클래스에 아래의 메소드가 추가 되었다.이름 대로 지정한 시간 대기 후에 캔슬을 발생시킨다.  
지정할 수 있는 것은 Int32 나 TimeSpan 이다  
```
class CancellationTokenSamples02 : IExecutable
{
	public void Execute()
	{
	  //
	  // .NET 4.5からCancellationTokenSourceにCancelAfterメソッド
	  // が追加された。CancelAfterメソッドは、指定時間後にキャンセル
	  // 処理を行うメソッドとなっている。
	  //
	  // CancellationTokenSource.CancelAfter メソッド 
	  //   http://msdn.microsoft.com/ja-jp/library/hh194678(v=vs.110).aspx
	  //
	  // 引数には、Int32またはTimeSpanが指定できる。
	  //
	  var tokenSource = new CancellationTokenSource();
	  var token       = tokenSource.Token;

	  Action action = () =>
	  {
		while (true)
		{
		  if (token.IsCancellationRequested)
		  {
			Output.WriteLine("Canceled!");
			break;
		  }

		  Output.Write(".");
		  Thread.Sleep(TimeSpan.FromSeconds(1));
		}
	  };

	  var task = Task.Run(action, token);

	  //
	  // 3秒後にキャンセル
	  //
	  tokenSource.CancelAfter(TimeSpan.FromSeconds(3));
	  task.Wait();

	  tokenSource.Dispose();
	}
}
```
   
   
     
## UI 스레드와 동기
태스크에는 UI 스레드와 동기 기능이 준비 되어 있다.  
UI 스레드와 동기할 때는 SynchronizationContext을 이용한다. 
어떤 비동기 처리가 완료한 후에 UI 스레드에서 컨트롤을 조작할 때는 아래처럼 한다.  
  
```
CancellationTokenSource tokenSource = new CancellationTokenSource();
CancellationToken       token       = tokenSource.Token;

Task rootTask         = Task.Factory.StartNew(() => { 비동기 처리 });
Task continuationTask = rootTask.ContinueWith(
                            (antecedent) => { UI 갱신 처리 }, 
                            token, 
                            // 현재의 UI에 연결된 SynchronizationContext을 얻는다.
                            TaskScheduler.FromCurrentSynchronizationContext()
                        );
```
  
   
   
## 긴 시간 실행되는 태스크 
- Task를 만들 때 TaskCreationOptions을 지정할 수 있다.그 중에 TaskCreationOptions.LongRunning 라는 항목이 있다.
- 이름 대로 장 시간 처리되는 태스크라고 지정할 수 있는 것이다. 이것을 사용하면 태스크 스케줄러가 스레드풀 스레드를 이용하지 않고 태스크를 실행할 수 있다.
- LongRunning 플래그는 보통 사용하지 않는 것이 좋다. 기존 동작과 비교해서 성능이 크기 떨어지는 경우가 생겨도 괜찮은 경우만 사용하는 것이 좋다.
  
```
var task2 = Task.Factory.StartNew(() =>
		{
		  task2StartSignal.Set();
		  Output.WriteLine("LongRunning Task....");
		  SpinWait.SpinUntil(() => false, 5000);
		},
		TaskCreationOptions.LongRunning
```
  
  
  
## 태스크 스케쥴링
프리엠티브（preemptive: 전매권, 우선권） 멀티 태스크: 스레드가 어떤 처리를 하고 있어도 일정 간격으로 OS가 꼭 처리를 빼아서 컨텍스트 스위치를 한다. 통상 타이머를 사용한 하드웨어로 끼어들고 만약 스레드가 프리즈 하고 있어도 강제적으로 컨텍스트 스위치 할 수 있다.  
   
강조적（coooperative）멀티 태스크: 각 스레드에 자기 신고로 일정 간격으로 OS에 처리권을 돌려 받는다. 어디까지가 자기 신고이므로 공평성이 떨어진다(신고하지 않으면 쭉 같은 스레드가 처리를 계속한다). 만약 스레드가 프리즈한 경우면 OS 전체가 프리즈 한다. 불편한 반면 컨텍스트 스위치에 의한 부하가 작다는 이점이 있다.  
   
지금까지 설명한 스레드라는 것은 프리엠티브 멀티 태스크이다.
스레드풀은 프리엠티브와 강조적을 병향한다.프리엠티브한 스레드 위에 강조적 스레드를 사용하도록 되어 있어 공평성을 남기면서 컨텍스트 스위치 부담을 최소화 시킨다.   
  

## 태스크 추가와 가져오기
### 추가
태스크 실행 용의 스레드(워커스레드(worker thread)라고 불리는)에서 새로운 태스크를 추가하는 경우 태스크는 로컬 큐에 투입된다. 
보통 워커스레드 이외에서 태스크를 추가하면 글로벌 큐에 투입된다.  
  
### 가져오기
각 스레드는 현재의 태스크가 종료하면 먼저 로컬 큐를 보러간다. 로컬 큐에 태스크가 없다면 다음으로 다른 스레드 큐를 버로 간다. 만약 그곳에 태스크가 있다면 태스크를 강제로 빼어온다.   
이와 같은 행동을 워크스틸링（work stealing）이라고 부른다.
또 모든 스레드의 로컬 큐가 비어 있다면 글로벌 큐에서 태스크를 가져와서 실행한다.  
  

로컬 큐에서 빼오면 다른 스레드에서의 스틸링과는 태스크 가져오는 방향이 달라진다. 로컬의 경우에는 FIFO（first in first out）, 스틸링 경우에는 FILO（first in last out）로 된다. 이것에 의해 다음과 같은 효과를 얻을 수 있다.  
  
가져오는 위치가 바뀌므로 락을 최소한으로 누를 수 있다.  
로컬을 FIFO 하므로 가까운 처리가 같은 스레드 내에서 실행될 가능성이 높아져거 메모리 캐시 사용률이 좋아진다.  
   
   

## 글로벌 큐와 로컬큐의 태스크
  
```
void QueueTasks()
{
    // TaskA is a top level task.
    Task taskA = Task.Factory.StartNew( () =>
    {                
        Console.WriteLine("I was enqueued on the thread pool's global queue."); 

        // TaskB is a nested task and TaskC is a child task. Both go to local queue.
        Task taskB = new Task( ()=> Console.WriteLine("I was enqueued on the local queue."));
        Task taskC = new Task(() => Console.WriteLine("I was enqueued on the local queue, too."),
                                TaskCreationOptions.AttachedToParent);

        taskB.Start();
        taskC.Start();

    });
}
```    
  

## 태스크의 인라인 전개
경우에 따라서 태스크가 대기하고 있을 때 대기 조작을 실행하고 있는 스레드에서 태스크가 동기적으로 실행되는 경우가 있다.  
이것은 성능상으로 향상이 있다.  
  
이 처리가 끝나지 않으면 블럭되고 있는 기존의 스레드를 활용하는 것으로 추가 스레드가 필요 없기 때문이다.  
태스크 인라인 전개는 관련된 스레드의 로컬 큐에서 대기 대상이 발견된 경우에만 행해진다. 이것은 재입에 의한 에러를 막기 위함이다.   



## 유연한 스케쥴링
Task 클래스에서는 태스크 시작 시에 TaskScheduler를 넘기는 것으로 태스크의 실행 방법을 어느 정도 유연하게 제어할 수 있다.  
  
특별히 지정하지 않는 경우 태스크는 스레드풀 상에서 실행된다. 특정 스레드 상에서 실행할 필요가 있는 처리가 있으면 그 특정 스레드에 태스크를 던질 필요가 있다.   이와 같은 경우에 TaskScheduler를 사용한다.  
   
```
var t = new Task(() => { /* 생략 */ });

// 태스크 실행 장소를 제어하기 위해 명시적으로 TaskScheduler를 지정
t.Start(TaskScheduler.FromCurrentSynchronizationContext());
```
  
  
  
## 동기 컨택스트 지정
TaskScheduler.FromCurrentSynchronizationContext 메소드를 사용하면 태스크가 특정 스레드에서 실행되도록 스케쥴 할 수 있다.  
  
이것은 Windows Form 이나 Windows Presentation Foundation 등의 프레임워크에 도움이 된다. 이들 프레임워크에서는 많은 경우 유저 인터페이스 오브젝트로의 접근이 그 UI 오브젝트가 만들어진 스레드에서만 실행 되도록 코드를 제어하기 때문이다.  
  
자세한 사용 방법은 MSDN의 「지정된 동기 컨텍스트에서 작업을 스케쥴 하기」를 참고해라.  




<br>     
    
