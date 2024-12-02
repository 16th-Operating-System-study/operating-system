# java.util.concurrent.locks
이 패키지는 자바에서 제공하는 고급 동기화 도구로, `synchronized`와 비교하여 더 유연한 제어를 제공한다.
동시성 제어를 위한 다양한 락 메커니즘을 제공한다. Lock 에 대해 공부할 때 멀티스레드 환경에서 성능과 스레드 안전성을 보장하는 것이 중요하다는 것을 염두에 두자.


## 주요 Interface
- `Lock` : 공유 자원에 한번에 한 쓰레드만 read, write를 수행 가능하도록 한다.
- `ReadWriteLock` : Lock에서 한단계 발전된 메커니즘을 제공하는 인터페이스다. 공유 자원에 여러개의 쓰레드가 read를 수행할 수 있지만, write는 한번에 한 쓰레드만 수행 가능하다.
- `Condition` : Object 클래스의 monitor method인 wait, nofity, notifyAll 메서드를 대체한다. wait → await, notify → signal, notifyAll → signalAll로 생각하면 된다.


## Interface의 구현체
### ReentrantLock
 - `Lock` 인터페이스의 구현체로, **재진입 가능**(Reentrant) 락을 제공한다. 같은 스레드가 여러 번 락을 획득할 수 있으며, 락 획득 횟수만큼 해제해야 완전히 해제된다.
 - 특징으로, 공정성(Fairness) 설정이 가능하다. 공정한 락을 설정하면 먼저 대기한 스레드가 먼저 락을 획득할 수 있다.
    - `synchronized`와 달리 **락 획득/해제**를 명시적으로 호출해야 한다. (`lock()`과 `unlock()`을 반드시 호출)
    - **타임아웃**이나 **인터럽트 처리 등 세밀한 제어를 제공한다.**
    - `ReentrantLock`은 락킹 상태를 알아보기 위한 유용한 쿼리 메서드를 제공한다.
        1. **`isLocked()`** : 락이 걸려있는지 아닌지 알려준다.
        2. **`getOwner()`** : 현재 락을 갖고 있는 스레드를 리턴한다.
        3. **`getQueuedThreads()`** : 락을 얻기 위해 대기하고 있는 스레드큐를 `Collection`으로 반환한다.
     
      ![ReentrantLock](https://github.com/16th-Operating-System-study/operating-system/blob/master/week4_concurrency/img/KangJinju/ReentrantLock.png?raw=true)


### ReentrantReadWriteLock
- ReadWriteLock의 구현체
- 읽기/쓰기 작업이 많은 경우 성능을 최적화하기 위한 락이다.
- 읽기 작업은 여러 스레드가 동시에 접근할 수 있지만, 쓰기 작업은 오직 하나의 스레드만 접근할 수 있다.
    - 주요 메서드:
        - `readLock()`: 읽기 작업을 위한 락 반환
        - `writeLock()`: 쓰기 작업을 위한 락 반환


# ReentrantLock 에 대해서 자세히 살펴보기
## 1. Reentrant = 재진입이 가능하다는 의미가 무엇인지?
- **재진입성(Reentrant)** 또는 **재진입 가능성**은 멀티스레드 환경에서 중요한 개념 중 하나로, 특정 스레드가 동일한 락을 여러 번 획득할 수 있는 능력을 뜻한다.
- 예를 들어, 스레드가 이미 락을 소유하고 있을 때, 해당 스레드가 다시 그 락을 획득해야 하는 상황이 발생할 수 있다.
- 일반적인 락이라면 이때 교착 상태(deadlock)가 발생할 수 있지만, 재진입 가능한 락(Reentrant Lock)은 같은 스레드가 락을 여러 번 획득하는 것을 허용하기 때문에 데드락을 막을 수 있다.


## 2. 재진입 가능성(락의 재획득)이 필요한 이유
**재진입 가능성**(Reentrancy)이 중요한 이유는 주로 멀티스레드 환경에서의 **동기화 문제**를 더 유연하고 안전하게 처리하기 위함이다.

특정 상황에서 동일한 스레드가 여러 번 동일한 락을 얻을 필요가 생길 수 있으며, 이를 통해 **교착 상태**(deadlock)를 피하고 **논리적 일관성**을 유지할 수 있다.


### 사용 예시
### (1) 재귀 호출 상황
- 재귀적인 메서드 호출이 발생할 때, 한 스레드가 이미 락을 소유한 상태에서 다시 그 메서드를 호출하는 상황이 발생한다.
- 이 때, 락을 재진입할 수 없다면 해당 스레드는 계속 대기해야 하고 결국 **교착 상태**에 빠지기 때문에 재진입 가능한 락이 필요하다. 동일한 스레드가 반복적으로 락을 얻어 정상적으로 재귀 호출을 처리할 수 있게 된다. (락을 획득하는 만큼 카운팅이 올라간다)


``` java
public class ReentrantExample {
    private final ReentrantLock lock = new ReentrantLock();

    public void outer() {
        lock.lock(); // 락 획득
        try {
            inner(); // 내부 메서드 호출, 이 메서드도 동일한 락을 요청
        } finally {
            lock.unlock(); // 락 해제
        }
    }

    public void inner() {
        lock.lock(); // 동일한 스레드가 다시 락을 요청
        try {
            System.out.println("Inner method");
        } finally {
            lock.unlock(); // 락 해제
        }
    }
}
```

### (2) **상속 구조에서 메서드 호출 상황**
- 상속 구조에서 부모 클래스의 동기화된 메서드가 자식 클래스의 동기화된 메서드를 호출할 때도 재진입이 필요할 수 있다.


``` java
public class Parent {
    protected ReentrantLock lock = new ReentrantLock();

    public void parentMethod() {
        lock.lock();
        try {
            System.out.println("Parent method");
        } finally {
            lock.unlock();
        }
    }
}

public class Child extends Parent {
    public void childMethod() {
        lock.lock();
        try {
            System.out.println("Child method");
            parentMethod(); // 부모 메서드 호출
        } finally {
            lock.unlock();
        }
    }
}
```

childMethod가 parentMethod를 호출할 때, 둘 다 동일한 락을 사용하는데, 만약 락이 재진입 가능하지 않다면 parentMethod는 락을 다시 획득하지 못하고 교착 상태가 발생하게 될 것이다.


### (3) **스레드 안전성과 논리적 일관성 보장**
- 락을 재진입할 수 있는 경우, 하나의 스레드가 여러 번 락을 얻더라도 프로그램의 **논리적 일관성**을 보장할 수 있다. 
- **특히, 복잡한 비즈니스 로직**을 처리하는 동기화된 메서드 간 호출에서, 재진입 가능한 락을 사용해 일관성 있고 안전한 스레드 처리 구조를 만들 수 있다.


### 결론
**멀티 스레드 환경에서 재진입 가능성의 의의**
→ deadlock 상태 방지 (락의 재획득을 허용하여 방지)
→ 재귀적 호출 & 상속 관계의 메서드 호출 & 복잡한 비즈니스 로직에서 스레드 안전성과 논리적 정합성을 보장하기 위해 사용


## 3. 내부 구조 (AQS_AbstractQueuedSynchronizer) & volatile 로 관리되는 매커니즘
<img src="https://github.com/16th-Operating-System-study/operating-system/blob/master/week4_concurrency/img/KangJinju/reentrantlock_arch.png?raw=true" width="400"/>


AQS는 **`synchronized`** 키워드를 직접 사용하지 않고, 내부적으로 **대기 큐**를 관리하여 스레드의 동기화를 처리한다. 

AQS의 동작은 다음과 같다.

### (1) **상태 관리**:

- AQS는 `volatile int state`라는 변수를 통해 현재 동기화 상태를 관리한다. 이 상태는 락의 소유 여부, 락의 소유자 스레드 등 다양한 정보를 나타낼 수 있다.
- volatile은 변수에 대한 접근과 관련된 메모리 가시성을 보장하는 데 사용된다. 주로 멀티 스레드 환경에서 스레드 간 데이터 일관성을 유지하기 위해 사용된다.
        
   ### **`volatile`의 의미**
- **가시성 보장**: `volatile`로 선언된 변수는 모든 스레드가 메모리에 직접 접근하므로, 하나의 스레드에서 변경된 값이 다른 스레드에게 즉시 반영된다. 즉, 스레드가 캐시된 값을 읽지 않고 항상 최신 값을 읽도록 보장한다.
- **순서 보장**: `volatile` 변수를 읽고 쓰는 작업은 특정 순서로 실행된다. 
- 하지만, `volatile`는 원자성을 보장하지 않기 때문에 여러 스레드가 동시에 접근하여 값을 변경하는 경우에 문제가 발생할 수 있고 해당 상황에서 원자성을 보장해주는 다른 처리가 필요하다.

### (2) **대기 큐**:
- AQS는 스레드가 락을 얻지 못할 경우 대기 큐에 대기하는 기능을 제공한다. 이 큐는 FIFO(선입선출) 방식으로 동작하여, 먼저 도착한 스레드가 먼저 락을 요청할 수 있도록 한다. 이를 통해 공정성을 보장할 수 있다.


### (3) **하위 클래스 구현**:
- AQS는 직접 사용되기보다는 **ReentrantLock**, **CountDownLatch**, **Semaphore**와 같은 동기화 클래스의 상위 클래스로 사용된다. 이들은 AQS를 상속받아 필요한 동기화 기능을 구현한다.
- 동기화 매커니즘을 위한 다양한 메서드를 제공하므로 상속받은 클래스들이 메서드 재정의하여 락을 획득하거나 해제하는 로직을 구체적으로 구현할 수 있게 된다.


## 4. Synchronized와 ReentrantLock의 차이점
- Synchronized 비용 발생 문제
    - [YT_대용량 트래픽 처리 시스템 - 자바 동시성 문제 해결?](https://youtu.be/XBXmHCy1EBA?t=1168)
 - 공정성 설정 가능
    - 이 둘의 차이는 **fairness(공정성)** 설정 가능 여부이다. 여기서 공정성은 모든 스레드가 자신의 작업을 수행할 기회를 공평하게 갖는 것을 의미한다.
    - 공정한 방법에선 큐 안에서 스레드들이 무조건 순서를 지켜가며 lock을 확보한다.
    - 불공정한 방법에서는, 대기 큐에 있던 스레드가 우선적으로 락을 얻는 것이 아니라 **막 락을 요청한 스레드**가 락을 바로 얻을 가능성이 있다.
    - 불공정 상태에서는 다른 스레드들에 우선 순위가 밀려 자원을 계속해서 할당 받지 못하는 스레드가 존재하는 starvation(기아 상태)가 발생할 수 있는데, 이를 공정성 설정으로 해결할 수 있다.



## 5. ReentrantLock의 공정성 설정 방법과 사용하는 상황?
- synchronized는 공정성을 지원하지 않는 반면, ReentrantLock은 생성자의 인자를 통해 공정/불공정을 설정할 수 있다.

``` java
    /* ReentrantLock의 공정성 설정을 위한 생성자 정의 */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
``` 

하지만 fairness가 무조건 좋은 것은 아니다. 여러 스레드의 작업이 막힘없이 공정하게 진행되어야 하거나(like UI 그리는 작업), 정합성을 보장해야 하는 작업(like 모든 거래 요청이 정확한 순서대로 처리되어야 하는 경우) 같이 반드시 필요한 상황이 아니라면 주의해서 설정해야 한다. 

왜냐하면 공정성 보장을 위한 추가적인 관리와 순서 제어가 필요하므로 오버 헤드가 발생하여 성능 저하의 위험이 있기 때문이다.


***

ref.
- [java_concurrent_locks](https://www.baeldung.com/java-concurrent-locks)
- [lock을 사용한 동기화](https://zion830.tistory.com/57)
- [ReentrantLock의 DeadLock 해결 방법](https://sslblog.tistory.com/207#google_vignette)
