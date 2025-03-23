CPU Core는 CPU 내부에서 실제 연산을 수행하는 독립적인 처리 장치를 말하며, 명령어를 실행하는 주체다. 초기 CPU의 발전은 core에 집적되는 트랜지스터의 수를 증가하는 방향으로 진행되었다. 트랜지스터의 크기를 작게 만들어서 core에 더 많은 트랜지스터를 집적함으로써 core의 성능 향상을 이루었다.

그러나 다량의 트랜지스터를 집적하면서 발열과 전력소모 문제가 발생했으며, 트랜지스터를 작게 만들다보니 양자 터널링 현상을 겪게 되었다. 이처럼 CPU Core는 이전만큼 폭발적인 퍼포먼스 향상을 이루기 어려워졌다. 더이상 하나의 CPU Core만으로는 성능 향상을 이루기 어려워, 다량의 CPU Core를 활용하는 방향으로 발전하게 됐다.

# Process & Thread

## Process
프로세스는 하나의 실행 단위로, OS에서 Process Control Block (PCB)라는 자료구조로 나타내어진다. 

## Thread
스레드는 하나의 실행 단위로, OS에서 Thread Control Block (TCB)라는 자료구조로 나타내어진다. 

### Kernel level thread & User level thread

#### Kernel level thread



#### User level thread

```C++
#include <iostream>
#include <cstdlib>
#include <cstring>
#include <vector>
#include <csignal>
#include <setjmp.h>
#include <unistd.h>

#define STACK_SIZE 1024 * 64  // 각 스레드의 스택 크기
#define MAX_THREADS 3         // 최대 스레드 개수

struct Thread {
    jmp_buf context;   // 컨텍스트 저장 (레지스터 상태 포함)
    char* stack;       // 스택 메모리
    bool finished;     // 스레드 종료 여부
};

// 유저 레벨 스레드 목록
std::vector<Thread> threads;
int currentThread = 0;  // 현재 실행 중인 스레드 인덱스

// 다음 스레드를 실행하는 스케줄러
void schedule(int) {
    int prevThread = currentThread;
    
    // 다음 실행할 스레드 찾기 (라운드 로빈 방식)
    do {
        currentThread = (currentThread + 1) % threads.size();
    } while (threads[currentThread].finished); // 종료된 스레드는 건너뜀

    // 이전 스레드 저장 후, 다음 스레드로 스위칭
    if (setjmp(threads[prevThread].context) == 0) {
        longjmp(threads[currentThread].context, 1);
    }
}

// 유저 레벨 스레드가 실행할 함수 (스레드 내부에서 실행됨)
void threadFunction(int id) {
    for (int i = 0; i < 5; i++) {
        std::cout << "Thread " << id << " is running.\n";
        sleep(1);
        raise(SIGALRM);  // 타이머 신호를 발생시켜 컨텍스트 스위칭을 유도
    }
    
    // 스레드 종료 처리
    threads[currentThread].finished = true;
    raise(SIGALRM);  // 다음 스레드로 전환
}

// 유저 레벨 스레드 생성
void createThread(void (*func)(int), int id) {
    Thread thread;
    thread.stack = new char[STACK_SIZE];  // 스택 할당
    thread.finished = false;

    // 새로운 스레드의 컨텍스트 설정
    if (setjmp(thread.context) == 0) {
        return;
    }

    // 새 스레드의 스택을 설정
    asm volatile(
        "movq %0, %%rsp\n"
        :
        : "r"(thread.stack + STACK_SIZE)
    );

    // 스레드 실행
    func(id);

    // 스레드 종료 시 raise() 호출
    thread.finished = true;
    raise(SIGALRM);
}

// 메인 함수
int main() {
    signal(SIGALRM, schedule);  // SIGALRM 신호를 받을 때 스케줄링 실행

    // 3개의 유저 레벨 스레드 생성
    for (int i = 0; i < MAX_THREADS; i++) {
        createThread(threadFunction, i);
        threads.push_back({});
    }

    // 첫 번째 스레드 실행
    if (setjmp(threads[0].context) == 0) {
        longjmp(threads[0].context, 1);
    }

    std::cout << "All threads finished.\n";
    return 0;
}

```


