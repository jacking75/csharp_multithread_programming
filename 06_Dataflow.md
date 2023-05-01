# Dataflow
- 닷넷프레임워크 4.5에서 추가
- NuGet으로 입수 가능 https://www.nuget.org/packages/System.Threading.Tasks.Dataflow  
- [MS Docs](https://docs.microsoft.com/ko-kr/dotnet/standard/parallel-programming/dataflow-task-parallel-library?redirectedfrom=MSDN)
- 비동기 프로그래밍은 쉽지는 않지만 의외로 기존의 동기식 코드를 손쉽게 비동기 코드로 바꿀 수 있는 부분도 있다.
- 이름 공간: System.Threading.Tasks.Dataflow
- 어셈블리: System.Threading.Tasks.Dataflow (System.Threading.Tasks.Dataflow.dll 내)
  


## BufferBlock
- 범용적인 비동기 메시징 구조체를 나타낸다.
- 복수의 소스에서 쓰는 것이 가능하고 도 복수의 타겟에서 읽을 수도 있다.
- FIFO 큐에 메시지를 저장한다.
- 타겟이 메시지를 큐에서 가져가면 그 메시지는 큐에서 삭제된다.
- 복수의 메시지를 다른 컴포넌트에 전달하고, 그 컴포넌트에서 메시지를 수신하고 싶을 때 편리하다.
- 아래 예는 BufferBlock 오브젝트에 Int32 타입의 복수의 값을 post하고 그 오브젝트에서 값을 읽어들인다.  
  
```
// Create a BufferBlock<int> object.
var bufferBlock = new BufferBlock<int>();

// Post several messages to the block.
for (int i = 0; i < 3; i++)
{
   bufferBlock.Post(i);
}

// Receive the messages back from the block.
for (int i = 0; i < 3; i++)
{
   Console.WriteLine(bufferBlock.Receive());
}
```  
<pre>
Output:
   0
   1
   2
</pre>     


### TryReceive
- 정기적으로 폴링할 때 사용하면 좋다.
- 이 메소드를 호출한 스레드를 블럭 시키지 않는다.
  
```
// Post more messages to the block.
for (int i = 0; i < 3; i++)
{
   bufferBlock.Post(i);
}

// Receive the messages back from the block.
int value;
while (bufferBlock.TryReceive(out value))
{
   Console.WriteLine(value);
}
```   
<pre>
Output:
   0
   1
   2
</pre>  
  
### Post 예제
- post2와 post1은 병렬로 실행하므로 순서 보장은 못함.
- 아래 예제의 task의 완료 순서는 post2, post1, receive
  
```
// Write to and read from the message block concurrently. 
var post01 = Task.Run(() =>
   {
      bufferBlock.Post(0);
      bufferBlock.Post(1);
   });
var receive = Task.Run(() => 
   {
     for (int i = 0; i < 3; i++)
     {
        Console.WriteLine(bufferBlock.Receive());
     }
   });
var post2 = Task.Run(() => 
   {
     bufferBlock.Post(2);
   });
Task.WaitAll(post01, receive, post2);
```  
<pre>
Sample output:
   2
   0
   1
</pre>    


### 비동기로 쓰고, 읽기 
  
```
// Post more messages to the block asynchronously. 
for (int i = 0; i < 3; i++)
{
   await bufferBlock.SendAsync(i);
}

// Asynchronously receive the messages back from the block. 
for (int i = 0; i < 3; i++)
{
   Console.WriteLine(await bufferBlock.ReceiveAsync());
}
```  
<pre>
Output:
   0
   1
   2
</pre>     
  

### 통합 예제 
  
```
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

// Demonstrates a how to write to and read from a dataflow block. 
class DataflowReadWrite
{
   // Demonstrates asynchronous dataflow operations. 
   static async Task AsyncSendReceive(BufferBlock<int> bufferBlock)
   {
      // Post more messages to the block asynchronously. 
      for (int i = 0; i < 3; i++)
      {
        await bufferBlock.SendAsync(i);
      }

      // Asynchronously receive the messages back from the block. 
      for (int i = 0; i < 3; i++)
      {
        Console.WriteLine(await bufferBlock.ReceiveAsync());
      }
          
          /*Output:
             0
             1
             2
           */
     }

       static void Main(string[] args)
       {
          // Create a BufferBlock<int> object. 
          var bufferBlock = new BufferBlock<int>();

          // Post several messages to the block. 
          for (int i = 0; i < 3; i++)
          {
             bufferBlock.Post(i);
          }

          // Receive the messages back from the block. 
          for (int i = 0; i < 3; i++)
          {
             Console.WriteLine(bufferBlock.Receive());
          }

          /* Output:
             0
             1
             2
           */ 

          // Post more messages to the block. 
          for (int i = 0; i < 3; i++)
          {
             bufferBlock.Post(i);
          }

          // Receive the messages back from the block. 
          int value;
          while (bufferBlock.TryReceive(out value))
          {
             Console.WriteLine(value);
          }

          /* Output:
             0
             1
             2
           */ 

          // Write to and read from the message block concurrently. 
          var post01 = Task.Run(() =>
             {
                bufferBlock.Post(0);
                bufferBlock.Post(1);
             });
          var receive = Task.Run(() => 
             {
                for (int i = 0; i < 3; i++)
                {
                   Console.WriteLine(bufferBlock.Receive());
                }
             });
          var post2 = Task.Run(() => 
             {
                bufferBlock.Post(2);
             });
          Task.WaitAll(post01, receive, post2);


          // Demonstrate asynchronous dataflow operations.
          AsyncSendReceive(bufferBlock).Wait();
       }
    }
}
```  
<pre>
Sample output:
         2
         0
         1
</pre>              
  



## BroadcastBlock
- 다른 컴포넌트에 복수의 메시지를 전달해도 가장 최신의 값만 사용하는 경우에 편리하다.
- 즉 최신 메시지가 이전 메시지를 덮어버린다.
- 메시지를 읽어도 삭제되지 않는다.
  
```
// Create a BroadcastBlock<double> object.
var broadcastBlock = new BroadcastBlock<double>(null);

// Post a message to the block.
broadcastBlock.Post(Math.PI);

// Receive the messages back from the block several times.
for (int i = 0; i < 3; i++)
{
   Console.WriteLine(broadcastBlock.Receive());
}
```  
<pre>
Output:
   3.14159265358979
   3.14159265358979
   3.14159265358979
</pre>  
  


## WriteOnceBlock
- BroadcastBlock와 비슷하지만 1회만 WriteOnceBlock를 기술할 수 있다.
- 읽기 전용에 비슷
- 메시지를 읽어도 메시지는 삭제 되지 않는다.
    - 복수의 타겟에서 메시지를 복사하여 수신한다.
- 복수의 메시지의 선두만 전달할 때 편리하다.
  
```
// Create a WriteOnceBlock<string> object.
var writeOnceBlock = new WriteOnceBlock<string>(null);

// Post several messages to the block in parallel. The first 
// message to be received is written to the block. 
// Subsequent messages are discarded.
Parallel.Invoke(
   () => writeOnceBlock.Post("Message 1"),
   () => writeOnceBlock.Post("Message 2"),
   () => writeOnceBlock.Post("Message 3"));

// Receive the message from the block.
Console.WriteLine(writeOnceBlock.Receive());
```  
<pre>
Sample output:
   Message 2
</pre>  
  


## ActionBlock
- 모든 수신 데이터 요소의 Action(혹은 Func)로 지정된 델리게이트를 호출하는 데이터 흐름 블럭을 제공
- 아래 예제는 ActionBlock 클래스를 사용하여 데이터 흐름 블럭을 사용해서 복수의 계산을 실행하고, 계산에 소요된 시간을 반환한다.
  
```
// Performs several computations by using dataflow and returns the elapsed
// time required to perform the computations.
static TimeSpan TimeDataflowComputations(int maxDegreeOfParallelism,
   int messageCount)
{
   var workerBlock = new ActionBlock<int>(
      millisecondsTimeout => Thread.Sleep(millisecondsTimeout),

      // 몇개를 병렬로 실행할지 정한다.
      new ExecutionDataflowBlockOptions
      {
         MaxDegreeOfParallelism = maxDegreeOfParallelism
      });


   Stopwatch stopwatch = new Stopwatch();
   stopwatch.Start();

   for (int i = 0; i < messageCount; i++)
   {
      // 처리할 작업을 요청한다. ActionBlock을 생성할 때 인자로 넣은 함수가 실행된다.
      workerBlock.Post(1000);
   }
   // 더 이상 처리할 작업을 받지 않음을 통보
   workerBlock.Complete();

   // Completion 멤버는 Task를 반환한다. task가 완료될 때까지 대기한다
   workerBlock.Completion.Wait();

   // Stop the timer and return the elapsed number of milliseconds.
   stopwatch.Stop();
   return stopwatch.Elapsed;
}

static void Main(string[] args)
{
  int processorCount = Environment.ProcessorCount;
  int messageCount = processorCount;

  // Print the number of processors on this computer.
  Console.WriteLine("Processor count = {0}.", processorCount);

  TimeSpan elapsed;

  // Perform two dataflow computations and print the elapsed
  // time required for each.

  // This call specifies a maximum degree of parallelism of 1.
  // This causes the dataflow block to process messages serially.
  elapsed = TimeDataflowComputations(1, messageCount);
  Console.WriteLine("Degree of parallelism = {0}; message count = {1}; " +
     "elapsed time = {2}ms.", 1, messageCount, (int)elapsed.TotalMilliseconds);

  // Perform the computations again. This time, specify the number of 
  // processors as the maximum degree of parallelism. This causes
  // multiple messages to be processed in parallel.
  elapsed = TimeDataflowComputations(processorCount, messageCount);
  Console.WriteLine("Degree of parallelism = {0}; message count = {1}; " +
     "elapsed time = {2}ms.", processorCount, messageCount, (int)elapsed.TotalMilliseconds);
}
```  
  

### ActionBlock 예외 처리  
  
```
// Create an ActionBlock<int> object that prints its input
// and throws ArgumentOutOfRangeException if the input
// is less than zero.
var throwIfNegative = new ActionBlock<int>(n =>
{
   Console.WriteLine("n = {0}", n);
   if (n < 0)
   {
      throw new ArgumentOutOfRangeException();
   }
});

// Post values to the block.
throwIfNegative.Post(0);
throwIfNegative.Post(-1);
throwIfNegative.Post(1);
throwIfNegative.Post(-2);
throwIfNegative.Complete();

// Wait for completion in a try/catch block.
try
{
   throwIfNegative.Completion.Wait();
}
catch (AggregateException ae)
{
   // If an unhandled exception occurs during dataflow processing, all
   // exceptions are propagated through an AggregateException object.
   ae.Handle(e =>
   {
      Console.WriteLine("Encountered {0}: {1}", 
         e.GetType().Name, e.Message);
      return true;
   });
}
```  
<pre>   
Output:
n = 0
n = -1
Encountered ArgumentOutOfRangeException: Specified argument was out of the range of valid values.
</pre>  
   
- 예외가 발생하면 발생 이후의 요청 작업은 모두 취소 된다.
  

### ActionBlock과 연결되는 task
- ContinueWith를 사용하여 ActionBlock이 끝난 후에 처리할 작업을 지정할 수 있다.
- 예외가 발생하여 중간에 ActionBlock이 끝나더라도 호출된다.
  
```
// Create an ActionBlock<int> object that prints its input
// and throws ArgumentOutOfRangeException if the input
// is less than zero.
var throwIfNegative = new ActionBlock<int>(n =>
{
   Console.WriteLine("n = {0}", n);
   if (n < 0)
   {
      throw new ArgumentOutOfRangeException();
   }
});

// Create a continuation task that prints the overall 
// task status to the console when the block finishes.
throwIfNegative.Completion.ContinueWith(task =>
{
   Console.WriteLine("The status of the completion task is '{0}'.", 
      task.Status);
});

// Post values to the block.
throwIfNegative.Post(0);
throwIfNegative.Post(-1);
throwIfNegative.Post(1);
throwIfNegative.Post(-2);
throwIfNegative.Complete();

// Wait for completion in a try/catch block.
try
{
   throwIfNegative.Completion.Wait();
}
catch (AggregateException ae)
{
   // If an unhandled exception occurs during dataflow processing, all
   // exceptions are propagated through an AggregateException object.
   ae.Handle(e =>
   {
      Console.WriteLine("Encountered {0}: {1}",
         e.GetType().Name, e.Message);
      return true;
   });
}
```  
<pre>   
Output:
n = 0
n = -1
The status of the completion task is 'Faulted'.
Encountered ArgumentOutOfRangeException: Specified argument was out of the range of valid values.
</pre>   
    



## TransformBlock<TInput, TOutput>
- ActionBlock와 비슷하지만 소스에서 타겟으로 메시지를 전달한다.
    - System.Func<TInput, TOutput> 또는 System.Func<TInput, Task>
- System.Func<TInput, TOutput> 을 사용하는 TransformBlock<TInput, TOutput> 오브젝트를 사용하면 각 입력 요소를 처리하는 델리게이트를 반환
- System.Func<TInput, Task> 을 사용하는 TransformBlock<TInput, TOutput> 오브젝트를 사용하면 각 입력 요소를 처리하는 Task 를 반환
- ActionBlock 과 비슷하게 각 입력 요소를 동기 및 비동기로 조작할 수 있다
  
```
// Create a TransformBlock<int, double> object that 
// computes the square root of its input.
var transformBlock = new TransformBlock<int, double>(n => Math.Sqrt(n));

// Post several messages to the block.
transformBlock.Post(10);
transformBlock.Post(20);
transformBlock.Post(30);

// Read the output messages from the block.
for (int i = 0; i < 3; i++)
{
   Console.WriteLine(transformBlock.Receive());
}
```  
<pre>
Output:
   3.16227766016838
   4.47213595499958
   5.47722557505166
</pre>     
  


## TransformManyBlock<TInput, TOutput>
- TransformBlock과 비슷하지만 1종류의 출력값만 아닌 복수의 출력 값을 저장
- System.Func<TInput, IEnumerable> 또는 System.Func< TInput, Task<IEnumerable> >
  
```
// Create a TransformManyBlock<string, char> object that splits
// a string into its individual characters.
var transformManyBlock = new TransformManyBlock<string, char>(
   s => s.ToCharArray());

// Post two messages to the first block.
transformManyBlock.Post("Hello");
transformManyBlock.Post("World");

// Receive all output values from the block.
for (int i = 0; i < ("Hello" + "World").Length; i++)
{
   Console.WriteLine(transformManyBlock.Receive());
}
```  
<pre>
Output:
   H
   e
   l
   l
   o
   W
   o
   r
   l
   d
</pre>   
  


## ActionBlock, TransformBlock<TInput, TOutput>, TransformManyBlock<TInput, TOutput> 에 사용하는 델리게이트 타입
  
| Type | Synchronous Delegate Type | Asynchronous Delegate Type | 
|ActionBlock|System.Action|System.Func<TInput, Task>| 
|TransformBlock<TInput, TOutput> | System.Func<TInput, TOutput>`2|System.Func<TInput, Task>| 
|TransformManyBlock<TInput, TOutput> |System.Func<TInput, IEnumerable> |System.Func< TInput, Task<IEnumerable> > |
  


## BatchBlock
- 출력 데이터 배열에 배치라고 부르는 입력 데이터 셋을 결합
  
```
// Create a BatchBlock<int> object that holds ten
// elements per batch.
var batchBlock = new BatchBlock<int>(10);

// Post several values to the block.
for (int i = 0; i < 13; i++)
{
   batchBlock.Post(i);
}
// Set the block to the completed state. This causes
// the block to propagate out any any remaining
// values as a final batch.
batchBlock.Complete();

// Print the sum of both batches.

Console.WriteLine("The sum of the elements in batch 1 is {0}.",
   batchBlock.Receive().Sum());

// 위에서 공간은 10개를 잡았는데 13개를 입력해서 여기서는 위에서 처리하지 못한 나머지 3개를 처리한다
Console.WriteLine("The sum of the elements in batch 2 is {0}.",
   batchBlock.Receive().Sum());
```  
<pre>
Output:
   The sum of the elements in batch 1 is 45.
   The sum of the elements in batch 2 is 33.
</pre>  
  

   
## JoinBlock<T1, T2>
  
```
// Create a JoinBlock<int, int, char> object that requires
// two numbers and an operator.
var joinBlock = new JoinBlock<int, int, char>();

// Post two values to each target of the join.

joinBlock.Target1.Post(3);
joinBlock.Target1.Post(6);

joinBlock.Target2.Post(5);
joinBlock.Target2.Post(4);

joinBlock.Target3.Post('+');
joinBlock.Target3.Post('-');

// Receive each group of values and apply the operator part
// to the number parts.

for (int i = 0; i < 2; i++)
{
   var data = joinBlock.Receive();
   switch (data.Item3)
   {
      case '+':
         Console.WriteLine("{0} + {1} = {2}",
            data.Item1, data.Item2, data.Item1 + data.Item2);
         break;
      case '-':
         Console.WriteLine("{0} - {1} = {2}",
            data.Item1, data.Item2, data.Item1 - data.Item2);
         break;
      default:
         Console.WriteLine("Unknown operator '{0}'.", data.Item3);
         break;
   }
}
```  
<pre>
Output:
   3 + 5 = 8
   6 - 4 = 2
</pre>  
  


## BatchedJoinBlock
  
```
// For demonstration, create a Func<int, int> that  
// returns its argument, or throws ArgumentOutOfRangeException 
// if the argument is less than zero.
Func<int, int> DoWork = n =>
{
   if (n < 0)
      throw new ArgumentOutOfRangeException();
   return n;
};

// Create a BatchedJoinBlock<int, Exception> object that holds  
// seven elements per batch. 
var batchedJoinBlock = new BatchedJoinBlock<int, Exception>(7);

// Post several items to the block. 
foreach (int i in new int[] { 5, 6, -7, -22, 13, 55, 0 })
{
   try
   {
      // Post the result of the worker to the  
      // first target of the block.
      batchedJoinBlock.Target1.Post(DoWork(i));
   }
   catch (ArgumentOutOfRangeException e) 
   {
      // If an error occurred, post the Exception to the  
      // second target of the block.
      batchedJoinBlock.Target2.Post(e); 
   }
}

// Read the results from the block. 
var results = batchedJoinBlock.Receive();

// Print the results to the console. 

// Print the results. 
foreach (int n in results.Item1)
{
   Console.WriteLine(n);
}
// Print failures. 
foreach (Exception e in results.Item2)
{
   Console.WriteLine(e.Message);
}
```  
<pre>
Output:
   5
   6
   13
   55
   0
   Specified argument was out of the range of valid values.
   Specified argument was out of the range of valid values.
</pre>  
  


## Producer-Consumer Dataflow Pattern
  
```
using System;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

// Demonstrates a basic producer and consumer pattern that uses dataflow. 
class DataflowProducerConsumer
{
   // Demonstrates the production end of the producer and consumer pattern. 
   static void Produce(ITargetBlock<byte[]> target)
   {
      // Create a Random object to generate random data.
      Random rand = new Random();

      // In a loop, fill a buffer with random data and 
      // post the buffer to the target block. 
      for (int i = 0; i < 100; i++)
      {
         // Create an array to hold random byte data. 
         byte[] buffer = new byte[1024];

         // Fill the buffer with random bytes.
         rand.NextBytes(buffer);

         // Post the result to the message block.
         target.Post(buffer);
      }

      // Set the target to the completed state to signal to the consumer 
      // that no more data will be available.
      target.Complete();
   }

   // Demonstrates the consumption end of the producer and consumer pattern. 
   static async Task<int> ConsumeAsync(ISourceBlock<byte[]> source)
   {
      // Initialize a counter to track the number of bytes that are processed. 
      int bytesProcessed = 0;

      // Read from the source buffer until the source buffer has no  
      // available output data. 
      while (await source.OutputAvailableAsync())
      {
         byte[] data = source.Receive();

         // Increment the count of bytes received.
         bytesProcessed += data.Length;
      }

      return bytesProcessed;
   }

   static void Main(string[] args)
   {
      // Create a BufferBlock<byte[]> object. This object serves as the  
      // target block for the producer and the source block for the consumer. 
      var buffer = new BufferBlock<byte[]>();

      // Start the consumer. The Consume method runs asynchronously.  
      var consumer = ConsumeAsync(buffer);

      // Post source data to the dataflow block.
      Produce(buffer);

      // Wait for the consumer to process all data.
      consumer.Wait();

      // Print the count of bytes processed to the console.
      Console.WriteLine("Processed {0} bytes.", consumer.Result);
   }
}
```  
<pre>
Output:
Processed 102400 bytes.
</pre>  

- 신뢰성 높이기
    - 복수의 consumer가 있는 경우 TryReceive를 사용하는 것이 좋다. 만약 Receive를 사용하면 다른 consumer가 읽으면 다른 consumer는 블럭이 된다.
```
// Demonstrates the consumption end of the producer and consumer pattern.
static async Task<int> ConsumeAsync(IReceivableSourceBlock<byte[]> source)
{
   // Initialize a counter to track the number of bytes that are processed.
   int bytesProcessed = 0;

   // Read from the source buffer until the source buffer has no 
   // available output data.
   while (await source.OutputAvailableAsync())
   {
      byte[] data;
      while (source.TryReceive(out data))
      {
         // Increment the count of bytes received.
         bytesProcessed += data.Length;
      }
   }

   return bytesProcessed;
}
```
  


## 데이터 블럭으로 데이터를 수신 했을 때 지정된 동작 실행
  
```
using System;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;

// Demonstrates how to provide delegates to exectution dataflow blocks. 
class DataflowExecutionBlocks
{
   // Computes the number of zero bytes that the provided file 
   // contains. 
   static int CountBytes(string path)
   {
      byte[] buffer = new byte[1024];
      int totalZeroBytesRead = 0;
      using (var fileStream = File.OpenRead(path))
      {
         int bytesRead = 0;
         do
         {
            bytesRead = fileStream.Read(buffer, 0, buffer.Length);
            totalZeroBytesRead += buffer.Count(b => b == 0);
         } while (bytesRead > 0);
      }

      return totalZeroBytesRead;
   }

   static void Main(string[] args)
   {
      // Create a temporary file on disk. 
      string tempFile = Path.GetTempFileName();

      // Write random data to the temporary file. 
      using (var fileStream = File.OpenWrite(tempFile))
      {
         Random rand = new Random();
         byte[] buffer = new byte[1024];
         for (int i = 0; i < 512; i++)
         {
            rand.NextBytes(buffer);
            fileStream.Write(buffer, 0, buffer.Length);
         }
      }

      // Create an ActionBlock<int> object that prints to the console  
      // the number of bytes read. 
      var printResult = new ActionBlock<int>(zeroBytesRead =>
      {
         Console.WriteLine("{0} contains {1} zero bytes.",
            Path.GetFileName(tempFile), zeroBytesRead);
      });

      // Create a TransformBlock<string, int> object that calls the  
      // CountBytes function and returns its result. 
      var countBytes = new TransformBlock<string, int>(
         new Func<string, int>(CountBytes));

      // Link the TransformBlock<string, int> object to the  
      // ActionBlock<int> object.
      countBytes.LinkTo(printResult);

      // Create a continuation task that completes the ActionBlock<int> 
      // object when the TransformBlock<string, int> finishes.
      countBytes.Completion.ContinueWith(delegate { printResult.Complete(); });

      // Post the path to the temporary file to the  
      // TransformBlock<string, int> object.
      countBytes.Post(tempFile);

      // Requests completion of the TransformBlock<string, int> object.
      countBytes.Complete();

      // Wait for the ActionBlock<int> object to print the message.
      printResult.Completion.Wait();

      // Delete the temporary file.
      File.Delete(tempFile);
   }
}
```  
<pre>
Sample output:
tmp4FBE.tmp contains 2081 zero bytes.
</pre>  
  
- 신뢰성 높이기  
```
// Asynchronously computes the number of zero bytes that the provided file  
// contains. 
static async Task<int> CountBytesAsync(string path)
{
   byte[] buffer = new byte[1024];
   int totalZeroBytesRead = 0;
   using (var fileStream = new FileStream(
      path, FileMode.Open, FileAccess.Read, FileShare.Read, 0x1000, true))
   {
      int bytesRead = 0;
      do
      {
         // Asynchronously read from the file stream.
         bytesRead = await fileStream.ReadAsync(buffer, 0, buffer.Length);
         totalZeroBytesRead += buffer.Count(b => b == 0);
      } while (bytesRead > 0);
   }

   return totalZeroBytesRead;
}

// Create a TransformBlock<string, int> object that calls the  
// CountBytes function and returns its result. 
var countBytesAsync = new TransformBlock<string, int>(async path =>
{
   byte[] buffer = new byte[1024];
   int totalZeroBytesRead = 0;
   using (var fileStream = new FileStream(
      path, FileMode.Open, FileAccess.Read, FileShare.Read, 0x1000, true))
   {
      int bytesRead = 0;
      do
      {
         // Asynchronously read from the file stream.
         bytesRead = await fileStream.ReadAsync(buffer, 0, buffer.Length);
         totalZeroBytesRead += buffer.Count(b => b == 0);
      } while (bytesRead > 0);
   }

   return totalZeroBytesRead;
});
```  
  


## Dataflow Block 병렬 실행 수 지정
  
```
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks.Dataflow;

// Demonstrates how to specify the maximum degree of parallelism  
// when using dataflow. 
class Program
{
   // Performs several computations by using dataflow and returns the elapsed 
   // time required to perform the computations. 
   static TimeSpan TimeDataflowComputations(int maxDegreeOfParallelism,
      int messageCount)
   {
      // Create an ActionBlock<int> that performs some work. 
      var workerBlock = new ActionBlock<int>(
         // Simulate work by suspending the current thread.
         millisecondsTimeout => Thread.Sleep(millisecondsTimeout),
         // Specify a maximum degree of parallelism. 
         new ExecutionDataflowBlockOptions
         {
            MaxDegreeOfParallelism = maxDegreeOfParallelism
         });

      // Compute the time that it takes for several messages to  
      // flow through the dataflow block.

      Stopwatch stopwatch = new Stopwatch();
      stopwatch.Start();

      for (int i = 0; i < messageCount; i++)
      {
         workerBlock.Post(1000);
      }
      workerBlock.Complete();

      // Wait for all messages to propagate through the network.
      workerBlock.Completion.Wait();

      // Stop the timer and return the elapsed number of milliseconds.
      stopwatch.Stop();
      return stopwatch.Elapsed;
   }
   static void Main(string[] args)
   {
      int processorCount = Environment.ProcessorCount;
      int messageCount = processorCount;

      // Print the number of processors on this computer.
      Console.WriteLine("Processor count = {0}.", processorCount);

      TimeSpan elapsed;

      // Perform two dataflow computations and print the elapsed 
      // time required for each. 

      // This call specifies a maximum degree of parallelism of 1. 
      // This causes the dataflow block to process messages serially.
      elapsed = TimeDataflowComputations(1, messageCount);
      Console.WriteLine("Degree of parallelism = {0}; message count = {1}; " +
         "elapsed time = {2}ms.", 1, messageCount, (int)elapsed.TotalMilliseconds);

      // Perform the computations again. This time, specify the number of  
      // processors as the maximum degree of parallelism. This causes 
      // multiple messages to be processed in parallel.
      elapsed = TimeDataflowComputations(processorCount, messageCount);
      Console.WriteLine("Degree of parallelism = {0}; message count = {1}; " +
         "elapsed time = {2}ms.", processorCount, messageCount, (int)elapsed.TotalMilliseconds);
   }
}
```  
<pre>
Sample output:
Processor count = 4.
Degree of parallelism = 1; message count = 4; elapsed time = 4032ms.
Degree of parallelism = 4; message count = 4; elapsed time = 1001ms.
</pre>  
  


## How to: Specify a Task Scheduler in a Dataflow Block
http://msdn.microsoft.com/en-us/library/hh228599.aspx   
```
using System;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using System.Threading.Tasks.Dataflow;
using System.Windows.Forms;

namespace WriterReadersWinForms
{
   public partial class Form1 : Form
   {
      // Broadcasts values to an ActionBlock<int> object that is associated 
      // with each check box.
      BroadcastBlock<int> broadcaster = new BroadcastBlock<int>(null);

      public Form1()
      {         
         InitializeComponent();


         // Create an ActionBlock<CheckBox> object that toggles the state 
         // of CheckBox objects. 
         // Specifying the current synchronization context enables the  
         // action to run on the user-interface thread. 
         var toggleCheckBox = new ActionBlock<CheckBox>(checkBox =>
         {
            checkBox.Checked = !checkBox.Checked;
         }, 
         new ExecutionDataflowBlockOptions
         {
            TaskScheduler = TaskScheduler.FromCurrentSynchronizationContext()  // UI 스레드에서 ActionBlock을 실행되도록 설정
         });

         // Create a ConcurrentExclusiveSchedulerPair object. 
         // Readers will run on the concurrent part of the scheduler pair. 
         // The writer will run on the exclusive part of the scheduler pair. 
         var taskSchedulerPair = new ConcurrentExclusiveSchedulerPair();

         // Create an ActionBlock<int> object for each reader CheckBox object. 
         // Each ActionBlock<int> object represents an action that can read  
         // from a resource in parallel to other readers. 
         // Specifying the concurrent part of the scheduler pair enables the  
         // reader to run in parallel to other actions that are managed by  
         // that scheduler. 
         var readerActions = 
            from checkBox in new CheckBox[] {checkBox1, checkBox2, checkBox3}
            select new ActionBlock<int>(milliseconds =>
            {
               // Toggle the check box to the checked state.
               toggleCheckBox.Post(checkBox);

               // Perform the read action. For demonstration, suspend the current 
               // thread to simulate a lengthy read operation.
               Thread.Sleep(milliseconds);

               // Toggle the check box to the unchecked state.
               toggleCheckBox.Post(checkBox);
            },
            new ExecutionDataflowBlockOptions
            {
               TaskScheduler = taskSchedulerPair.ConcurrentScheduler  // 현재의 스레드에서 실행되도록
            });

         // Create an ActionBlock<int> object for the writer CheckBox object. 
         // This ActionBlock<int> object represents an action that writes to  
         // a resource, but cannot run in parallel to readers. 
         // Specifying the exclusive part of the scheduler pair enables the  
         // writer to run in exclusively with respect to other actions that are  
         // managed by the scheduler pair. 
         var writerAction = new ActionBlock<int>(milliseconds =>
         {
            // Toggle the check box to the checked state.
            toggleCheckBox.Post(checkBox4);

            // Perform the write action. For demonstration, suspend the current 
            // thread to simulate a lengthy write operation.
            Thread.Sleep(milliseconds);

            // Toggle the check box to the unchecked state.
            toggleCheckBox.Post(checkBox4);
         },
         new ExecutionDataflowBlockOptions
         {
            TaskScheduler = taskSchedulerPair.ExclusiveScheduler  // 다른 스레드에서 실행되도록 
         });

         // Link the broadcaster to each reader and writer block. 
         // The BroadcastBlock<T> class propagates values that it  
         // receives to all connected targets. 
         foreach (var readerAction in readerActions)
         {
            broadcaster.LinkTo(readerAction);
         }
         broadcaster.LinkTo(writerAction);

         // Start the timer.
         timer1.Start();
      }

      // Event handler for the timer. 
      private void timer1_Tick(object sender, EventArgs e)
      {
         // Post a value to the broadcaster. The broadcaster 
         // sends this message to each target. 
         broadcaster.Post(1000);
      }
   }
}
```  
  


## 테스트: ActionBlock에서 Complete 메소드 호출 영향
- workerBlock.Post 호출과 동시에 스레드에서 작업을 한다.
    - 즉 Complete를 늦게 호출하거나 호출하지 않아도 ActionBlock에 정의된 작업은 처리된다.
- 만약 Complete을 호출하지 않으면 workerBlock.Completion.Wait()에 무한 대기 상태에 들어간다.
    - 즉 Complete 호출하지 않는다면 Completion.Wait()도 호출하면 안된다.
```
static TimeSpan TimeDataflowComputations(int maxDegreeOfParallelism, int messageCount)
{
    var workerBlock = new ActionBlock<int>(
       millisecondsTimeout => { Thread.Sleep(millisecondsTimeout); 
                                 Console.WriteLine("process workerBlock      : {0}.", DateTime.Now.Ticks); },

       // 몇개를 병렬로 실행할지 정한다.
       new ExecutionDataflowBlockOptions
       {
           MaxDegreeOfParallelism = maxDegreeOfParallelism
       });


    Stopwatch stopwatch = new Stopwatch();
    stopwatch.Start();

    for (int i = 0; i < messageCount; i++)
    {
        // 처리할 작업을 요청한다. ActionBlock을 생성할 때 인자로 넣은 함수가 실행된다.
        Console.WriteLine("Call workerBlock.Post       : {0}.", DateTime.Now.Ticks);
        workerBlock.Post(1000);
    }

    int DelayMillisecond = 2000;
    Console.WriteLine("딜레이 {0}밀리세컨드       : {1}.", DelayMillisecond, DateTime.Now.Ticks);
    Thread.Sleep(DelayMillisecond);

    // 더 이상 처리할 작업을 받지 않음을 통보
    Console.WriteLine("Call workerBlock.Complete   : {0}.", DateTime.Now.Ticks);
    workerBlock.Complete();

    // Completion 멤버는 Task를 반환한다. task가 완료될 때까지 대기한다
    Console.WriteLine("Call workerBlock.Completion : {0}.", DateTime.Now.Ticks);
    workerBlock.Completion.Wait();

    // Stop the timer and return the elapsed number of milliseconds.
    stopwatch.Stop();
    return stopwatch.Elapsed;
}

static void Main(string[] args)
{
    int processorCount = Environment.ProcessorCount;
    int messageCount = processorCount;

    Console.WriteLine("Processor count = {0}.", processorCount);

    TimeSpan elapsed;

    elapsed = TimeDataflowComputations(1, messageCount);
    Console.WriteLine("Degree of parallelism = {0}; message count = {1}; " +
       "elapsed time = {2}ms.", 1, messageCount, (int)elapsed.TotalMilliseconds);
}
```  
  
- 만약 Complete() 호출 후 또 Post 호출을 한다면?
    - 다시 Post()를 하면 Post 호출 결과가 실패가 되면서 정의된 작업이 처리 되지 않는다.  
```
static TimeSpan TimeDataflowComputations(int maxDegreeOfParallelism, int messageCount)
{
    var workerBlock = new ActionBlock<int>(
       millisecondsTimeout => { Thread.Sleep(millisecondsTimeout); 
                                 Console.WriteLine("process workerBlock      : {0}.", DateTime.Now.Ticks); },

       // 몇개를 병렬로 실행할지 정한다.
       new ExecutionDataflowBlockOptions
       {
           MaxDegreeOfParallelism = maxDegreeOfParallelism
       });


    Stopwatch stopwatch = new Stopwatch();
    stopwatch.Start();

    for (int i = 0; i < messageCount; i++)
    {
        var bResult = workerBlock.Post(1000);
        Console.WriteLine("Call workerBlock.Post       : {0}, Result:{1}", DateTime.Now.Ticks, bResult);
    }

    int DelayMillisecond = 2000;
    Console.WriteLine("딜레이 {0}밀리세컨드       : {1}.", DelayMillisecond, DateTime.Now.Ticks);
    Thread.Sleep(DelayMillisecond);

    // 더 이상 처리할 작업을 받지 않음을 통보
    Console.WriteLine("Call workerBlock.Complete   : {0}.", DateTime.Now.Ticks);
    workerBlock.Complete();

    // Completion 멤버는 Task를 반환한다. task가 완료될 때까지 대기한다
    Console.WriteLine("Call workerBlock.Completion : {0}.", DateTime.Now.Ticks);
    workerBlock.Completion.Wait();
    Console.WriteLine("End workerBlock.Completion : {0}.", DateTime.Now.Ticks);


    // 다시 한번 더
    Console.WriteLine("");
    Console.WriteLine("다시 한번 더 !!!       : {0}.", DateTime.Now.Ticks);

    for (int i = 0; i < messageCount; i++)
    {
        var bResult = workerBlock.Post(1000);
        Console.WriteLine("Call workerBlock.Post       : {0}, Result:{1}", DateTime.Now.Ticks, bResult);
    }

    DelayMillisecond = 2000;
    Console.WriteLine("딜레이 {0}밀리세컨드       : {1}.", DelayMillisecond, DateTime.Now.Ticks);
    Thread.Sleep(DelayMillisecond);

    // 더 이상 처리할 작업을 받지 않음을 통보
    Console.WriteLine("Call workerBlock.Complete   : {0}.", DateTime.Now.Ticks);
    workerBlock.Complete();

    // Completion 멤버는 Task를 반환한다. task가 완료될 때까지 대기한다
    Console.WriteLine("Call workerBlock.Completion : {0}.", DateTime.Now.Ticks);
    workerBlock.Completion.Wait();
    Console.WriteLine("End workerBlock.Completion : {0}.", DateTime.Now.Ticks);

    stopwatch.Stop();
    return stopwatch.Elapsed;
}

static void Main(string[] args)
{
    int processorCount = Environment.ProcessorCount;
    int messageCount = processorCount;

    Console.WriteLine("Processor count = {0}.", processorCount);

    TimeSpan elapsed;

    elapsed = TimeDataflowComputations(1, messageCount);
    Console.WriteLine("Degree of parallelism = {0}; message count = {1}; " +
       "elapsed time = {2}ms.", 1, messageCount, (int)elapsed.TotalMilliseconds);
}
```  



## TPL DataFlow를 사용하여 처리의 진행 상황을 표시하기  
    
```
/// </summary>
 public static class Status
 {
     public static BufferBlock<string> TextBlock { get; private set; } = new BufferBlock<string>();
     public static BufferBlock<int> ProgressBlock { get; private set; } = new BufferBlock<int>();
 }

 /// <summary>
 /// MainWindow.xaml の相互作用ロジック
 /// </summary>
 public partial class MainWindow : Window
 {
     public MainWindow()
     {
         InitializeComponent();
         //UI 스레드 얻기
         TaskScheduler uiTaskScheduler = TaskScheduler.FromCurrentSynchronizationContext();

         //progressBar용 ActionBlock 만들기
         ActionBlock<int> progressBlock = new ActionBlock<int>((n) =>
         {
             progressBar.Value = Math.Min( Math.Max(0, n),100);
         }, new ExecutionDataflowBlockOptions() { TaskScheduler = uiTaskScheduler });

         //정적 클래스의 BufferBlock에 진행 표시 처리를 연결한다
         Status.ProgressBlock.LinkTo(progressBlock, new DataflowLinkOptions() { PropagateCompletion = true });

         //한번에 쓴다.
         Status.TextBlock.LinkTo( 
             new ActionBlock<string>((n) => 
             { textBlock.Text = n; },
             new ExecutionDataflowBlockOptions() { TaskScheduler = uiTaskScheduler })
             , new DataflowLinkOptions() { PropagateCompletion = true });
     }


     private async void button_Click(object sender, RoutedEventArgs e)
     {
         await Task.Factory.StartNew(() => {
              Status.TextBlock.Post("보통의 For 문");
              for (int i = 0; i < 100; i++)
             {
                 System.Threading.Thread.Sleep(20);
                 Status.ProgressBlock.Post(i);
             }
         });

         await Task.Factory.StartNew(() => {
             Status.TextBlock.Post("Parallel.For 버전");
             int size = 1200;
             int c1 = 0;
　　　　　　　 //Progress용 수치를 만드는 함수  병렬처리이므로 Interlocked로 더한다
             Func<int> func = () => { c1 = System.Threading.Interlocked.Add(ref c1, 1); return c1 * 100 / size; };

             Parallel.For(0, size, (n) =>
             {
                 System.Threading.Thread.Sleep(80);
                 Status.ProgressBlock.SendAsync(func()).Wait();
             });
         });

     }
 }
```
  
   

## TPL Dataflow를 사용하여 복수의 스레드에서의 결과를 하나의 파일에 쓰기

```
[TestMethod]
public void TestMethod3()
{
    BufferBlock<int> inputBlock = new BufferBlock<int>();

    //병렬 수 5로 실행시킨다
    TransformManyBlock<int, string> transBlock = new TransformManyBlock<int, string>((n) =>
     {
         return Enumerable.Range(0, 500).Select(m => n + "_" + m);
     },new ExecutionDataflowBlockOptions() { MaxDegreeOfParallelism = 5 });

    //100개 모으면 다음 블럭에 보낸다
    BatchBlock<string> batchBlock = new BatchBlock<string>(100);
    //파일에 쓰기 위한 ActionBlock
    ActionBlock<string[]> fileWriteBlock2 = new ActionBlock<string[]>((n) => {
        using (var f = System.IO.File.AppendText("test3.txt"))
        {
            n.ToList().ForEach(m => f.WriteLine(m));
        }
    });

    //block을 연결한다
    inputBlock.LinkTo(transBlock, new DataflowLinkOptions() { PropagateCompletion = true });
    transBlock.LinkTo(batchBlock, new DataflowLinkOptions() { PropagateCompletion = true });
    batchBlock.LinkTo(fileWriteBlock2, new DataflowLinkOptions() { PropagateCompletion = true });

    //inputBlock에 입력한다
    for (int i = 0; i < 5; i++)
    {
        inputBlock.SendAsync(i);
    }
    //실행
    inputBlock.Complete();
    fileWriteBlock2.Completion.Wait();
}
```
  


## 참고
- [(일어)이제서야 사용하는 TPL Dataflow](https://azyobuzin.hatenablog.com/entry/2019/05/26/164155  )


