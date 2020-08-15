# iOS 질문 모음 - 6

### Q.

> DispatchQueue.global() 과 DispatchQueue.init() 의 차이가 무엇인가요?

백그라운드 스레드를 관리하는 큐를 생성하고 싶은데요. [DispatchQueue.global()](https://developer.apple.com/documentation/dispatch/dispatchqueue/2300077-global) 으로 생성하는 방법과 [DispatchQueue.init()](https://developer.apple.com/documentation/dispatch/dispatchqueue/2300059-init) 으로 label 을 설정하여 생성하는 방법에는 어떤 차이가 있나요?

[질문 바로가기](https://stackoverflow.com/questions/53489764/understanding-dispatch-queue-threading/53489813#53489813)

### A.

* DispatchQueue.global() 은 인자로 받은 우선순위에 해당하는 전역 시스템 큐를 반환합니다. 이 전역 시스템 큐는 여러 작업을 동시에 수행하는 concurrent queue 입니다.

* DispatchQueue 의 생성자로 만든 큐는 인자로 받은 여러 설정을 적용한 큐를 생성합니다.

  ```swift
  let myQueue = DispatchQueue(label: "myQueue")
  ```

  위와 같이 label 만으로 만들게 되면, unspecified 의 우선순위를 가진 serial queue 를 생성합니다.

  ```swift
  let myQueue = DispatchQueue(label: "myQueue", qos: .userInitiated, attributes: .concurrent)
  ```

  우선순위와 serial, concurrent 구분을 직접 설정해줄 수도 있습니다.

* 지금까지의 예시들을 봤을 때 두 방식에 별다른 차이가 없는데요. 그럼 왜 global() 을 사용하지 않고 생성자를 사용할까요?

  첫 번째는 디버깅에서의 이점이 있습니다.

  ```swift
  func debugQueue(_ queue: DispatchQueue) {
      for _ in 1...5 {
          queue.async {
              print("break point")
          }
      }
      
      for _ in 1...5 {
          queue.async {
              print("break point")
          }
      }
  }
  ```

  디버깅 상황을 만들기 위해 큐를 인자로 받아 그 큐에서 간단한 동작을 하는 함수를 만들었습니다. 위에서 만든 myQueue 를 인자로 넘겨주고 print 가 실행되는 부분에 브레이크포인트를 설정해보겠습니다.

  <img width="456" alt="스크린샷 2020-08-16 오전 12 44 22" src="https://user-images.githubusercontent.com/50410213/90315863-b598e100-df59-11ea-9c0a-433f4f912daf.png">

  Xcode 의 Debug navigator 를 보면 브레이크포인트가 설정된 코드가 실행된 큐와 스레드, 그리고 해당 큐가 관리 중인 스레드들도 확인할 수 있습니다. 지금은 큐가 하나뿐이지만 사용하는 큐가 더 많아지고, 해야 하는 작업량도 많아진다면 label 을 통해 큐를 구분할 수 있으니 디버깅하기 편해지겠죠?

  <img width="458" alt="스크린샷 2020-08-16 오전 1 16 24" src="https://user-images.githubusercontent.com/50410213/90316593-22ae7580-df5e-11ea-925c-889c446a2d58.png">

  반면에 DispatchQueue.global() 을 넘겨주면 시스템에 구현되어있는 전역 큐를 사용하게 되어 디버깅이 어려워지게 됩니다.

  두 번째는 [barrier](https://developer.apple.com/documentation/dispatch/dispatchworkitemflags/1780674-barrier) 와 같은 DispatchQueue 의 몇 가지 세부적인 설정을 사용하기 위함입니다. barrier 란 단어 뜻 그대로 동시성을 지원하는 비동기 큐에서 장벽 같은 역할을 해주는 기능인데요.

  ![barrier](https://koenig-media.raywenderlich.com/uploads/2014/09/Dispatch-Barrier-Swift.png)

  위 그림처럼 비동기 작업들이 실행되다가 flag 가 barrier 로 설정된 작업이 실행되면 그 작업이 끝날 때 까지 serial queue 처럼 동작하게 됩니다. 간단한 예시코드를 보겠습니다.

  ```swift
  let myQueue = DispatchQueue(label: "myQueue", attributes: .concurrent)
  
  for i in 1...5 {
      myQueue.async {
          print("\(i)")
      }
  }
  
  myQueue.async(flags: .barrier) {
      print("barrier!!")
      sleep(5)
  }
  
  for i in 6...10 {
      myQueue.async {
          print("\(i)")
      }
  }
  ```

  숫자들을 비동기로 출력하는 코드 사이에 flags 를 barrier 로 설정한 코드를 추가해주었습니다.

  ![barrier](https://user-images.githubusercontent.com/50410213/90316847-f98ee480-df5f-11ea-9de9-2bc991e8bf83.gif)

  숫자들이 출력되다가 barrier 블록이 실행되자 다음 작업들이 barrier 블록이 끝날 때까지 기다리는 것을 확인할 수 있습니다. 만약 barrier 가 없다면 concurrent queue 에서 비동기로 숫자들을 출력하기 때문에 한번에 모든 숫자가 출력될 것입니다. [barrier](https://developer.apple.com/documentation/dispatch/1452917-dispatch_barrier_sync?language=occ) 공식문서를 보면 barrier 를 지정하는 큐는 직접 만든 concurrent queue 여야만 한다고 합니다. barrier 를 포함한 몇몇 세부적인 설정들은 글로벌 큐에서는 사용할 수 없습니다. 

### 참고할 만한 비슷한 질문, 자료

* [Grand Central Dispatch Tutorial for Swift 4](https://www.raywenderlich.com/5370-grand-central-dispatch-tutorial-for-swift-4-part-1-2)
* [DispatchQueue sync vs sync barrier in concurrent queue](https://stackoverflow.com/questions/58236153/dispatchqueue-sync-vs-sync-barrier-in-concurrent-queue)

----

### Q.

> DispatchQueue 에서 main.sync 는 언제 사용하나요?

DispatchQueue 에 대해 공부하면서 생긴 의문인데, 많은 자료에서 DispatchQueue.main 에서 sync 를 호출하지 말라고 합니다. 왜 호출하지 말라고 하는 것이며, 그럼 왜 GCD 에서 DispatchQueue.main.sync 가 구현되어있는 건가요?

[질문 바로가기](https://stackoverflow.com/questions/42772907/what-does-main-sync-in-global-async-mean)

### A.

* 먼저 DispatchQueue.main 에서 sync 를 호출하면 안 되는 이유를 이해하려면 [Deadlock(교착 상태)](https://ko.wikipedia.org/wiki/교착_상태) 이란 개념을 알아야 하는데요. Deadlock 이란 `두 개 이상의 작업이 서로 상대방의 작업이 끝나기만을 기다리고 있기 때문에 결과적으로 아무것도 완료되지 못하는 상태` 라고 합니다. Deadlock 은 DispatchQueue.main 뿐만 아니라 백그라운드 스레드를 관리하는 큐에서도 발생할 수 있습니다.

* 간단한 코드로 예를 들어보겠습니다. DispatchQueue 의 생성자로 myQueue 를 생성해주었습니다. 생성자의 attributes 를 concurrent 로 설정해주지 않으면 기본값으로 serial queue 를 생성하게 됩니다.

  ```swift
  let myQueue = DispatchQueue(label: "label")
  
  myQueue.async {
      myQueue.sync {
          // 외부 블록이 완료되기 전에 내부 블록은 시작되지 않습니다.
          // 외부 블록은 내부 블록이 완료되기를 기다립니다.
          // => deadlock
      }
      // 이 부분은 영원히 실행되지 않습니다.
  }
  ```

  myQueue 는 **serial queue** 이므로 한 번에 하나의 작업을 합니다. serial queue 에 추가되는 작업은 DispatchQueue 에서 관리하는 고유한 스레드에서 실행됩니다. 따라서 위 예시에서 sync 내부 블록은 외부 블록이 완료될 때까지 기다리고, sync 블록 이후의 동작은 sync 내부 블록이 완료될 때까지 기다리는 deadlock 이 발생하게 됩니다.

  이 문제는 myQueue 를 생성할 때 

  ```swift
  let myQueue = DispatchQueue(label: "com.ttozzi", attributes: .concurrent)
  ```

  와 같이 concurrent 속성을 설정하여 한번에 여러 작업을 실행할 수 있도록 해주어 해결할 수 있습니다.

* 다시 본론으로 돌아오면, DispatchQueue.main 은 프로그램의 메인 스레드에서 작업을 실행하는 전역적으로 사용이 가능한 **serial queue** 입니다. 앱의 run loop 와 함께 작동하며, 대기 중인 동작을 run loop 의 다른 이벤트 처리와 조율해줍니다. 이러한 DispatchQueue.main 에서 sync 를 호출하게 된다면 끊임없이 앱의 이벤트 처리를 하고 있던 메인 스레드가 sync 호출에 의해 멈추게 되고 위에서 예시로 들었던 코드와 같이 deadlock 이 발생하게 됩니다.

* 그럼 DispatchQueue.main 은 어떤 경우에 사용할까요? 백그라운드 스레드에서 이루어지는 작업들 사이에 순서에 맞게 메인 스레드에서 어떠한 작업이 이루어져야 할 때 사용합니다. 다음 코드를 보겠습니다.

  ```swift
  DispatchQueue.global().async {
      // UI 업데이트 전 실행되어야만 하는 코드
      DispatchQueue.main.sync {
          // UI 업데이트
      }
      // UI 업데이트 후에 실행되어야만 하는 코드
  }
  ```

  백그라운드 스레드에서 작업이 진행되는 중 UI 업데이트가 이루어져야 하는 상황입니다. 좀 더 구체적으로 예를 들자면 테이블 뷰나 콜렉션 뷰에 셀을 추가하거나 제거할 때, 셀의 추가, 제거 작업이 이루어지던 중 네트워크 통신과 같은 다른 비동기 작업으로 인해 DataSource 의 데이터가 변경된다면 DataSource 의 데이터와 뷰의 데이터가 일치하지 않아 에러가 발생하게 됩니다. 이를 방지하기 위해 사용합니다. 애플의 [PHPhotoLibraryChangeObserver](https://developer.apple.com/documentation/photokit/phphotolibrarychangeobserver) 문서 `Handling photo library changes in a collection view` 의 예시코드에서도 사진앨범의 데이터와 업데이트한 콜렉션 뷰의 데이터가 달라지는 상황을 방지하기위해 DispatchQueue.main.sync 를 호출하는 것을 확인할 수 있습니다.

### 참고할 만한 비슷한 질문, 자료

* [How do I create a deadlock in Grand Central Dispatch?](https://stackoverflow.com/questions/15381209/how-do-i-create-a-deadlock-in-grand-central-dispatch)
* [why concurrentQueue.sync DON'T cause deadlock](https://stackoverflow.com/questions/53293965/why-concurrentqueue-sync-dont-cause-deadlock)
* [Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW2)
* [Difference between DispatchQueue.main.async and DispatchQueue.main.sync](https://stackoverflow.com/questions/44324595/difference-between-dispatchqueue-main-async-and-dispatchqueue-main-sync)
* [How to dispatch on main queue synchronously without a deadlock?](https://stackoverflow.com/questions/10330679/how-to-dispatch-on-main-queue-synchronously-without-a-deadlock)

-----
