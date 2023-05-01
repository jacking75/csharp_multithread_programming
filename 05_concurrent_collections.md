# Concurrent Collections
.NET 4.0에서 추가된 스레드 세이프한 컬렉션이다.  
  
아래의 기능을 가진다.  
- Producer-Consumer 패턴을 구현하고 있다.
- IProducerConsumerPattern 인터페이스를 구현하고 있다.
- 복수의 스레드에서 스레드 세이프하게 요소를 추가 및 삭제할 수 있다.
- 요소가 존재하지 않는 경우에 다른 스레드가 요소를 삭제하려고 한다면 블록 시킬 수 있다.
- 요소가 꽉 찬 경우에 다른 스레드가 요소를 삽입하려고 한다면 블록 시킬 수 있다.
- 요소의 삽입 (Insert) 및 삭제(Delete) 조작에 더하여 TryXXX계의 메소드가 준비 되어 있다.
- 타임 아웃을 설정할 수 있다.
- 취소 토큰을 설정 할 수 있다.      
  
<br>     
    
foreach 할 때에 2 종류의 Enumeration를 이용할 수 있다.  
- 읽기 전용 상태에서 루프.
    - 루프 중에 요소를 변경하면 예외가 발생하는 모드.
- 통상의 컬렉션 루프와 같다.루프 중에 컬렉션 변경을 허용하는 모드.
    - 루프 중에 요소를 변경하여도 예외가 발생하지 않는다.
  
  
## .NET 4.0에서 추가된 스레드 세이프한 컬렉션들
- BlockingCollection
- ConcurrentBag
    - 순서를 가지지 않고 중복을 허용하는 컬렉션
    - Set은 순서를 다가지지 않고 중복을 허용하지 않는 컬렉션.
- ConcurrentDictionary
    - Dictionary의 스레드 세이프 판
- ConcurrentQueue
    - Queue의 스레드 세이프 판
- ConcurrentStack
    - Stack의 스레드 세이프 판
  
```
void ProduceData(BlockingCollection<int> output)
{
    try
    {
 	for (int i = 0; i < 100; i++)
	{
	    output.Add(i);
	}
    }
    finally
    {
	// 데이터 생산이 끝난 것을 통지
	output.CompleteAdding();
    }
}

void ConsumeData(BlockingCollection<int> input, BlockingCollection<int> output)
{
    try
    {
        // GetConsumingEnumerable 메소드에서 소비자용의 Enumeration을 얻을 수 있다.
        // 생산자측이 생산이 끝난 것을 통지 해줄 때까지 블록 시킨다.
        foreach (int value in input.GetConsumingEnumerable())
        {
	output.Add(value * value);
        }
    }
    finally
    {
	// 데이터 생산이 끝난 것을 통지.
	output.CompleteAdding();
    }
}

void Main()
{
	BlockingCollection<int> buffer1 = new BlockingCollection<int>(100);
	BlockingCollection<int> buffer2 = new BlockingCollection<int>(100);
	
	// 생산자와 소비자를 병렬로 처리한다.
	Parallel.Invoke(
		() => ProduceData(buffer1),
		() => ConsumeData(buffer1, buffer2)
	);
	
	// 소비자가 소비한 결과를 메인으로 마지막에 출력.
	foreach (int value in buffer2.GetConsumingEnumerable())
	{
		Console.WriteLine(value);
	}
}
```
  
  
## .NET의 모든 병렬 컨테이너는 lock-free 인가?
[FAQ :: Are all of the new concurrent collections lock-free?](https://blogs.msdn.microsoft.com/pfxteam/2010/01/26/faq-are-all-of-the-new-concurrent-collections-lock-free/ ) 의 글을 일부 번역/정리.  
(이 대답은 .NET Framework 4를 기반으로 한다. 아래의 세부 정보는 문서화 되지 않은 구현 세부 사항이므로 향후 릴리스에서 변경 될 수 있다.)  
  
새로운 System.Collections.Concurrent 네임 스페이스의 모든 컬렉션은 보통 성능 이점을 얻기 위해 어느 정도 lock-free 기술을 사용하지만 일부는 기존 잠금이 사용된다.  
  
lock-free 기술에 의존하는 것이 때로는 가장 효율적인 해결책이 아닐 수 있다는 것에 주의해야 한다.  
우리가 “lock-free”라고 말하면 잠금(.NET에서의 전통적인 상호 배제 잠금은 일반적으로 C#의 “lock” 키워드 또는 Visual Basic “SyncLock” 키워드를 통해 System.Threading.Monitor 클래스를 사용할 수 있다)을 memory barriers와 비교 및 ​​스왑 CPU 명령을 사용하여 피할 수 있다(.NET에서는 “CAS” 작업을 System.Threading.Interlocked 클래스를 통해 사용할 수 있다).  
  
ConcurrentQueue 및 ConcurrentStack은 이러한 방식으로 완전히 lock-free 이다. 이들은 결코 lock을 사용하지 않는다. 그러나 이들은 spin과 CAS 작업이 실패하여 경쟁에 직면했을 때 작업을 다시 시도 할 수 있다.  
  
ConcurrentBag은 동기화 필요성을 최소화 하기 위해 다양한 메커니즘을 사용한다. 예를 들어, 접근하는 각 스레드에 대해 로컬 큐를 유지 관리하며 일부 조건에서는 스레드가 거의 또는 전혀 경합하지 않아서 lock-free 방식으로 로컬 큐에 접근 할 수 있다. 따라서 ConcurrentBag는 때때로 lock을 요구하지만 특정 동시 시나리오(예 : 많은 스레드가 동일한 속도로 생성 및 소비하는 경우)에 매우 효율적인 콜렉션이다.  
  
ConcurrentDictionary<TKey, TValue>는 Dictionary에 데이터를 추가하거나 업데이트 할 때 세밀한 lock을 사용하지만 읽기 작업에는 lock-free를 사용한다.  
그래서 Dictionary에서 읽는 것이 가장 빈번한 작업 인 시나리오에 최적화되어 있다.    
  
  
  
## ConcurrentQueue
[MS Docks](https://docs.microsoft.com/ko-kr/dotnet/api/system.collections.concurrent.concurrentqueue-1?view=net-5.0 )  
  
- 생성
    ```
    var queue = new System.Collections.Concurrent.ConcurrentQueue<string>();
    ```
- 추가
    ```
    queue.Enqueue("First");
    ```
- 참조
    ```
    string item;
    if (queue.TryPeek(out item))
    {
        Console.WriteLine("First item was " + item);
    }
    else
    {
        Console.WriteLine("Queue was empty.");
    }
    ```
- 데이터 가져오기(삭제)
    ```
    if (queue.TryDequeue(out item))
    {
        Console.WriteLine("Dequeued first item " + item);
    }
    ```
- Enumerator 
    ```
    var queue = new ConcurrentQueue<string>();
                    
    // adding to queue is much the same as before
    queue.Enqueue("First");
        
    var iterator = queue.GetEnumerator();
        
    queue.Enqueue("Second");
    queue.Enqueue("Third");
    
    // only shows First
    while (iterator.MoveNext() 
    ``` 
       



## ConcurrentDictionary     
- 생성
    ```
    using System.Collections.Concurrent;
    static ConcurrentDictionary<string, MemoryDBUserAuth> Cache = new ConcurrentDictionary<string, MemoryDBUserAuth>();
    ```
- 값 가져오기
    ```
    Cache.TryGetValue(userID, out oldAuthInfo) == false)
    ```
- 추가
    ```
    Cache.TryAdd(userID, newAuthInfo);
    ```
- 갱신
    ```
    var oldToken = oldAuthInfo.GameAuthToken;
    Cache.TryUpdate(userID, newAuthInfo, oldAuthInfo);
    ```
- 삭제
    ```
    MemoryDBUserAuth userInfo;
    Cache.TryRemove(userID, out userInfo);
    ```
  



<br>     
    
