# Linux 6.19 커널 프로세스의 삶 — 생성, 전환, 소멸

> 소스 기준: `/home/inineapa/Lab/linux-6.19`

---

## 1. 핵심 자료구조와 관계

### 1.1 전체 구조 다이어그램

```mermaid
graph TB
    subgraph "task_struct (include/linux/sched.h:819)"
        TS_STATE["__state: unsigned int<br/>TASK_RUNNING | INTERRUPTIBLE | ..."]
        TS_PID["pid / tgid"]
        TS_STACK["void *stack (커널 스택)"]
        TS_THREAD["thread: thread_struct<br/>(arch별 CPU 레지스터)"]
        TS_SCHED["sched_class *<br/>se: sched_entity<br/>rt: sched_rt_entity<br/>dl: sched_dl_entity"]
        TS_EXIT["exit_state / exit_code / exit_signal"]
        TS_REL["real_parent / parent<br/>children / sibling / group_leader"]
    end

    TS_STATE --- MM["mm_struct *mm<br/>(유저 주소 공간)"]
    TS_STATE --- AMM["mm_struct *active_mm<br/>(커널 스레드가 빌려쓰는 mm)"]
    TS_STATE --- FS["fs_struct *fs<br/>(cwd, root, umask)"]
    TS_STATE --- FILES["files_struct *files<br/>(열린 파일 디스크립터 테이블)"]
    TS_STATE --- SIG["signal_struct *signal<br/>(스레드 그룹 공유 시그널 상태)"]
    TS_STATE --- SIGHAND["sighand_struct *sighand<br/>(시그널 핸들러 테이블)"]
    TS_STATE --- NSPROXY["nsproxy *nsproxy<br/>(pid/net/mnt/uts/ipc ns)"]
    TS_STATE --- CRED["cred *cred<br/>(uid, gid, capabilities)"]

    MM --> VMA["vm_area_struct<br/>(VMA 리스트 / maple tree)"]
    MM --> PGD["pgd_t *pgd<br/>(페이지 테이블 루트)"]

    TS_SCHED --> SE["sched_entity"]
    SE --> RB["rb_node run_node<br/>(CFS RB-tree 내 위치)"]
    SE --> VRT["u64 vruntime<br/>(가상 실행 시간)"]

    FILES --> FDT["fdtable<br/>(fd → file * 매핑)"]
```

### 1.2 스케줄러 관련 자료구조

```mermaid
graph LR
    subgraph "Per-CPU runqueue (kernel/sched/sched.h:1119)"
        RQ["struct rq"]
        RQ_LOCK["raw_spinlock_t __lock"]
        RQ_NR["nr_running"]
        RQ_CURR["curr: task_struct *"]
        RQ_IDLE["idle: task_struct *"]
        RQ_CFS["cfs: cfs_rq"]
        RQ_RT["rt: rt_rq"]
        RQ_DL["dl: dl_rq"]
    end

    subgraph "CFS runqueue"
        CFS_RQ["struct cfs_rq"]
        CFS_RB["RB-tree<br/>(vruntime 순 정렬)"]
        CFS_NR["h_nr_queued"]
    end

    subgraph "sched_entity (sched.h:570)"
        SE2["load_weight load"]
        SE2_RB["rb_node run_node"]
        SE2_VR["u64 vruntime"]
        SE2_SLICE["u64 slice"]
        SE2_EXEC["u64 sum_exec_runtime"]
    end

    RQ_CFS --> CFS_RQ
    CFS_RQ --> CFS_RB
    CFS_RB --> SE2_RB

    subgraph "스케줄링 클래스 우선순위"
        SC_STOP["stop_sched_class"]
        SC_DL["dl_sched_class"]
        SC_RT["rt_sched_class"]
        SC_FAIR["fair_sched_class"]
        SC_IDLE["idle_sched_class"]
    end
    SC_STOP -->|"높음 →"| SC_DL --> SC_RT --> SC_FAIR --> SC_IDLE
```

### 1.3 task_struct 상태 전이

```mermaid
stateDiagram-v2
    [*] --> TASK_NEW: copy_process()에서 생성
    TASK_NEW --> TASK_RUNNING: wake_up_new_task()

    TASK_RUNNING --> TASK_INTERRUPTIBLE: schedule() 호출<br/>(시그널로 깨울 수 있음)
    TASK_RUNNING --> TASK_UNINTERRUPTIBLE: schedule() 호출<br/>(I/O 대기 등)
    TASK_RUNNING --> __TASK_STOPPED: SIGSTOP / ptrace
    TASK_RUNNING --> __TASK_TRACED: ptrace attach

    TASK_INTERRUPTIBLE --> TASK_RUNNING: wake_up() / 시그널
    TASK_UNINTERRUPTIBLE --> TASK_RUNNING: wake_up()
    __TASK_STOPPED --> TASK_RUNNING: SIGCONT
    __TASK_TRACED --> TASK_RUNNING: ptrace detach

    TASK_RUNNING --> TASK_DEAD: do_task_dead()
    TASK_DEAD --> EXIT_ZOMBIE: exit_notify()
    EXIT_ZOMBIE --> EXIT_DEAD: release_task()<br/>(부모가 wait() 호출 후)
    EXIT_DEAD --> [*]: free_task()
```

---

## 2. 프로세스 생성

### 2.1 유저 공간 프로세스 생성 (fork / clone)

```mermaid
sequenceDiagram
    participant U as 유저 프로세스
    participant SYS as syscall entry
    participant KC as kernel_clone()<br/>fork.c:2610
    participant CP as copy_process()<br/>fork.c:1966
    participant CT as copy_thread()<br/>arch/x86/process.c:170
    participant SCHED as 스케줄러
    participant CHILD as 자식 프로세스

    U->>SYS: fork() / clone() / clone3()
    SYS->>KC: kernel_clone(args)

    Note over KC: clone_flags 유효성 검증

    KC->>CP: copy_process(NULL, trace, NUMA_NO_NODE, args)

    Note over CP: dup_task_struct(current)<br/>→ task_struct + 커널 스택 복사

    rect rgb(255, 235, 205)
        Note over CP: === 리소스 복사 (clone_flags에 따라 공유 or 복사) ===
        CP->>CP: copy_files() — 파일 디스크립터 테이블
        CP->>CP: copy_fs() — 파일시스템 컨텍스트 (cwd, root)
        CP->>CP: copy_sighand() — 시그널 핸들러
        CP->>CP: copy_signal() — 시그널 상태
        CP->>CP: copy_mm() — 메모리 주소 공간 (COW 페이지 테이블)
        CP->>CP: copy_namespaces() — 네임스페이스
        CP->>CP: copy_io() — I/O 컨텍스트
    end

    rect rgb(220, 240, 255)
        Note over CP,SCHED: === 스케줄러 초기화 ===
        CP->>SCHED: sched_fork(clone_flags, p)
        Note over SCHED: p->__state = TASK_NEW<br/>p->prio 계산<br/>p->sched_class 결정<br/>p->se.vruntime 초기화
    end

    CP->>CT: copy_thread(p, args)
    Note over CT: childregs = task_pt_regs(p)<br/>*childregs = *current_pt_regs()<br/>childregs->ax = 0 (자식 반환값)<br/>p->thread.sp = childregs<br/>p->thread.io_bitmap 복사<br/>ret_from_fork_asm 설정

    CP->>CP: alloc_pid() — PID 할당
    CP-->>KC: return p (새 task_struct)

    rect rgb(220, 255, 220)
        Note over KC,SCHED: === 스케줄러가 새 태스크를 런큐에 삽입 ===
        KC->>SCHED: wake_up_new_task(p)
        Note over SCHED: p->__state = TASK_RUNNING<br/>activate_task(rq, p)<br/>→ enqueue_task(rq, p)<br/>→ CFS: rbtree에 삽입<br/>check_preempt_curr()로<br/>현재 태스크 선점 여부 확인
    end

    KC-->>SYS: return child pid
    SYS-->>U: fork() 반환 (부모: child pid)

    Note over CHILD: 스케줄러에 의해 pick되면 실행 시작
    SCHED->>CHILD: context_switch → ret_from_fork_asm
    Note over CHILD: schedule_tail(prev) 호출<br/>→ finish_task_switch(prev)<br/>→ syscall_exit_to_user_mode()<br/>→ 유저공간 복귀 (fork 반환값 = 0)
```

#### 핵심 코드 포인트

| 단계 | 함수 | 위치 | 핵심 동작 |
|------|------|------|-----------|
| 진입 | `kernel_clone()` | `fork.c:2610` | clone_flags 검증, copy_process 호출 |
| 복사 | `dup_task_struct()` | `fork.c:1052` | task_struct 메모리 할당 + 복사 |
| 스케줄러 초기화 | `sched_fork()` | `core.c` | 상태=TASK_NEW, vruntime 초기화 |
| arch 설정 | `copy_thread()` | `arch/x86/process.c:170` | 레지스터 프레임 설정, ax=0 |
| 런큐 삽입 | `wake_up_new_task()` | `core.c` | TASK_RUNNING 전환, enqueue |
| 자식 첫 실행 | `ret_from_fork_asm` | `entry_64.S` | schedule_tail() → 유저 복귀 |

**`CLONE_*` 플래그에 따른 공유 vs 복사:**

| 플래그 | 설정 시 | 미설정 시 |
|--------|---------|-----------|
| `CLONE_VM` | mm 공유 (스레드) | mm 복사 (COW) |
| `CLONE_FILES` | files_struct 공유 | 복사 |
| `CLONE_FS` | fs_struct 공유 | 복사 |
| `CLONE_SIGHAND` | sighand 공유 | 복사 |
| `CLONE_THREAD` | 같은 스레드 그룹 | 새 스레드 그룹 |

### 2.2 커널 스레드 생성

```mermaid
sequenceDiagram
    participant CALLER as 커널 코드
    participant KCR as kthread_create()<br/>kthread.c:578
    participant LIST as kthread_create_list<br/>(전역 링크드 리스트)
    participant KTD as kthreadd (PID 2)<br/>kthread.c:815
    participant CK as create_kthread()<br/>kthread.c:478
    participant KT as kernel_thread()<br/>fork.c:2700
    participant SCHED as 스케줄러
    participant NEW as 새 커널 스레드

    CALLER->>KCR: kthread_create_on_node(threadfn, data, node, name)
    KCR->>LIST: kthread_create_info를 리스트에 추가
    KCR->>KTD: wake_up_process(kthreadd_task)
    KCR->>KCR: wait_for_completion_killable(&done)

    Note over KTD: kthreadd 무한 루프에서 깨어남

    KTD->>CK: create_kthread(create_info)
    CK->>KT: kernel_thread(kthread, create, name,<br/>CLONE_FS | CLONE_FILES | SIGCHLD)

    Note over KT: kernel_clone_args.kthread = 1<br/>flags |= CLONE_VM | CLONE_UNTRACED

    KT->>KT: kernel_clone(args)<br/>→ copy_process()
    Note over KT: copy_thread()에서:<br/>childregs 전부 0으로 초기화<br/>kthread_frame_init(frame, fn, arg)<br/>→ ret_from_fork_asm 진입점 설정

    rect rgb(220, 255, 220)
        Note over KT,SCHED: 스케줄러 삽입
        KT->>SCHED: wake_up_new_task(p)
    end

    Note over NEW: kthread() 함수가 진입점 (kthread.c:411)

    SCHED->>NEW: 스케줄에 의해 실행 시작
    NEW->>NEW: complete(done) — 생성 요청자 깨움
    NEW->>NEW: schedule_preempt_disabled() — park 상태로 대기
    KCR-->>CALLER: return task_struct * (새 커널 스레드)

    Note over CALLER: kthread_run() 또는 wake_up_process()로<br/>실제 실행 시작

    CALLER->>NEW: wake_up_process()
    NEW->>NEW: threadfn(data) — 사용자 지정 함수 실행
    NEW->>NEW: kthread_exit(ret)
```

**커널 스레드 vs 유저 프로세스 차이:**

| 속성 | 커널 스레드 | 유저 프로세스 |
|------|-------------|---------------|
| `mm` | NULL (유저 주소 공간 없음) | 고유 mm_struct |
| `active_mm` | 이전 태스크에서 빌려씀 (lazy TLB) | == mm |
| `PF_KTHREAD` 플래그 | 설정됨 | 미설정 |
| 페이지 테이블 | 커널 페이지 테이블만 사용 | 유저+커널 |
| `copy_thread()` 동작 | childregs=0, kthread_frame_init | 부모 regs 복사, ax=0 |

---

## 3. 컨텍스트 스위치 (Context Switch)

### 3.1 전체 흐름

```mermaid
sequenceDiagram
    participant PREV as 현재 태스크 (prev)
    participant SCHED as __schedule()<br/>core.c:6722
    participant PICK as pick_next_task()
    participant CFS as CFS 스케줄러<br/>fair.c
    participant CTX as context_switch()<br/>core.c:5201
    participant ASM as __switch_to_asm()<br/>entry_64.S:178
    participant HW as CPU 하드웨어
    participant NEXT as 다음 태스크 (next)
    participant FIN as finish_task_switch()<br/>core.c:5075

    Note over PREV: schedule() 호출 지점들:<br/>1) 자발적: schedule(), cond_resched()<br/>2) 선점: preempt_schedule_irq()<br/>3) syscall 리턴: exit_to_user_mode()

    PREV->>SCHED: __schedule(sched_mode)
    Note over SCHED: local_irq_disable()<br/>rq_lock(rq, &rf)<br/>update_rq_clock(rq)

    SCHED->>SCHED: prev_state = READ_ONCE(prev->__state)
    Note over SCHED: prev가 TASK_RUNNING이 아니면<br/>try_to_block_task()로 런큐에서 제거

    rect rgb(220, 240, 255)
        Note over SCHED,CFS: === 스케줄러: 다음 태스크 선택 ===
        SCHED->>PICK: pick_next_task(rq, prev, rf)
        PICK->>CFS: pick_next_task_fair(rq, prev, rf)
        Note over CFS: RB-tree에서 가장 작은<br/>vruntime을 가진 sched_entity 선택
        CFS-->>PICK: next task_struct
        PICK-->>SCHED: next
    end

    Note over SCHED: clear_tsk_need_resched(prev)

    alt prev != next (태스크 전환 필요)
        SCHED->>CTX: context_switch(rq, prev, next, rf)

        Note over CTX: === 1단계: 전환 준비 ===
        CTX->>CTX: prepare_task_switch(rq, prev, next)
        Note over CTX: perf_event_task_sched_out()<br/>WRITE_ONCE(next->on_cpu, 1)

        Note over CTX: === 2단계: 메모리 공간 전환 ===
        alt next->mm == NULL (커널 스레드)
            CTX->>CTX: enter_lazy_tlb(prev->active_mm, next)
            Note over CTX: next->active_mm = prev->active_mm<br/>(TLB flush 없이 빌려씀)
        else next->mm != NULL (유저 프로세스)
            CTX->>HW: switch_mm_irqs_off(prev->active_mm, next->mm, next)
            Note over HW: CR3 레지스터에 next->mm->pgd 로드<br/>→ 페이지 테이블 전환<br/>→ TLB flush (PCID 지원시 최소화)
        end

        Note over CTX: === 3단계: CPU 레지스터 전환 ===
        CTX->>ASM: switch_to(prev, next, prev)

        rect rgb(255, 220, 220)
            Note over ASM,HW: === 하드웨어 레벨 레지스터 전환 ===
            ASM->>HW: pushq %rbp, %rbx, %r12-r15 (prev 스택에 저장)
            ASM->>HW: movq %rsp → prev->thread.sp (prev 스택 포인터 저장)
            Note over HW: ★ 여기서 스택이 바뀜 ★
            ASM->>HW: movq next->thread.sp → %rsp (next 스택 포인터 로드)
            ASM->>HW: popq %r15-r12, %rbx, %rbp (next 스택에서 복원)
            ASM->>HW: FILL_RETURN_BUFFER (Spectre RSB 완화)
        end

        ASM->>ASM: jmp __switch_to (process_64.c:610)
        Note over ASM: switch_fpu(prev) — FPU/SIMD 상태 저장/복원<br/>save_fsgs(prev) / x86_fsgsbase_load()<br/>load_TLS(next) — TLS 디스크립터<br/>DS/ES 세그먼트 레지스터 전환<br/>x86_pkru_load() — 메모리 보호 키<br/>raw_cpu_write(current_task, next)

        Note over NEXT: ← 이 시점부터 next 태스크의 스택에서 실행

        NEXT->>FIN: finish_task_switch(prev)
        Note over FIN: vtime_task_switch(prev)<br/>perf_event_task_sched_in()<br/>smp_store_release(&prev->on_cpu, 0)<br/>finish_lock_switch(rq) — rq 락 해제<br/>prev가 TASK_DEAD이면 put_task_struct()
    end
```

### 3.2 스케줄러 개입 지점 종합

```mermaid
graph TB
    subgraph "스케줄러가 개입하는 지점들"
        direction TB

        subgraph "프로세스 생성 시"
            C1["sched_fork()<br/>→ __state=TASK_NEW, prio 계산, vruntime 초기화"]
            C2["wake_up_new_task()<br/>→ activate_task() → enqueue_task()<br/>→ check_preempt_curr() 선점 검사"]
        end

        subgraph "자발적 양보"
            V1["schedule()<br/>→ sched_submit_work() → __schedule()"]
            V2["cond_resched()<br/>→ TIF_NEED_RESCHED 체크 후 schedule()"]
            V3["mutex_lock() / wait_event() 등<br/>→ set_current_state(INTERRUPTIBLE)<br/>→ schedule()"]
        end

        subgraph "비자발적 선점 (Preemption)"
            P1["타이머 인터럽트 (tick)<br/>→ scheduler_tick()<br/>→ curr->sched_class->task_tick()<br/>→ resched_curr() (TIF_NEED_RESCHED 설정)"]
            P2["인터럽트 리턴 시<br/>→ preempt_schedule_irq()"]
            P3["syscall 리턴 시<br/>→ exit_to_user_mode_loop()<br/>→ TIF_NEED_RESCHED 체크 → schedule()"]
            P4["wake_up_process() 시<br/>→ try_to_wake_up()<br/>→ ttwu_do_activate() → enqueue<br/>→ check_preempt_curr() → 선점 가능시 resched"]
        end

        subgraph "프로세스 소멸 시"
            D1["do_task_dead()<br/>→ __state=TASK_DEAD<br/>→ __schedule() (마지막 스케줄링)<br/>→ finish_task_switch()에서<br/>   put_task_struct()으로 정리"]
        end
    end

    style C1 fill:#e8f5e9
    style C2 fill:#e8f5e9
    style V1 fill:#fff3e0
    style V2 fill:#fff3e0
    style V3 fill:#fff3e0
    style P1 fill:#fce4ec
    style P2 fill:#fce4ec
    style P3 fill:#fce4ec
    style P4 fill:#fce4ec
    style D1 fill:#e3f2fd
```

### 3.3 x86_64에서의 CPU 상태 전환 상세

전환 시 저장/복원되는 CPU 상태:

| 분류 | 항목 | 저장 위치 | 전환 함수 |
|------|------|-----------|-----------|
| 범용 레지스터 | rbp, rbx, r12-r15 | 커널 스택 (push/pop) | `__switch_to_asm` |
| 스택 포인터 | rsp | `thread.sp` | `__switch_to_asm` |
| 페이지 테이블 | CR3 | `mm->pgd` | `switch_mm_irqs_off` |
| FPU/SSE/AVX | xmm, ymm, zmm 등 | `fpu->__fpstate` | `switch_fpu` |
| 세그먼트 레지스터 | FS, GS, DS, ES | `thread.fsbase/gsbase/ds/es` | `__switch_to` |
| TLS | GDT 엔트리 | `thread.tls_array` | `load_TLS` |
| 메모리 보호 키 | PKRU | `thread.pkru` | `x86_pkru_load` |
| per-CPU 변수 | `current_task` | GS base 기반 | `raw_cpu_write` |
| 스택 canary | `__stack_chk_guard` | per-CPU | `__switch_to_asm` |

---

## 4. 프로세스 소멸

### 4.1 전체 소멸 과정

```mermaid
sequenceDiagram
    participant U as 유저 프로세스
    participant SYS as syscall
    participant DGE as do_group_exit()<br/>exit.c:1087
    participant DE as do_exit()<br/>exit.c:896
    participant EN as exit_notify()<br/>exit.c:736
    participant PAR as 부모 프로세스
    participant RT as release_task()<br/>exit.c:244
    participant SCHED as 스케줄러
    participant RCU as RCU callback

    U->>SYS: exit(code) 또는 exit_group(code)
    SYS->>DGE: do_group_exit((code & 0xff) << 8)
    Note over DGE: signal->flags = SIGNAL_GROUP_EXIT<br/>zap_other_threads(current)<br/>→ 같은 스레드 그룹의 다른 스레드에 SIGKILL

    DGE->>DE: do_exit(exit_code)

    rect rgb(255, 245, 230)
        Note over DE: === Phase 1: 조기 정리 ===
        DE->>DE: exit_signals(tsk) — PF_EXITING 설정
        DE->>DE: ptrace_event(PTRACE_EVENT_EXIT)
        DE->>DE: io_uring_files_cancel()
    end

    rect rgb(255, 235, 205)
        Note over DE: === Phase 2: 리소스 해제 (순서 중요!) ===
        DE->>DE: exit_mm() — 메모리 주소 공간 해제
        DE->>DE: exit_sem() — System V 세마포어
        DE->>DE: exit_shm() — 공유 메모리
        DE->>DE: exit_files() — 파일 디스크립터 닫기
        DE->>DE: exit_fs() — 파일시스템 컨텍스트
        DE->>DE: disassociate_ctty(1) — 제어 터미널 분리
        DE->>DE: exit_nsproxy_namespaces() — 네임스페이스
        DE->>DE: exit_thread() — 아키텍처별 정리
        DE->>DE: exit_io_context()
    end

    rect rgb(220, 240, 255)
        Note over DE,EN: === Phase 3: 부모 통지 ===
        DE->>EN: exit_notify(tsk, group_dead)
        EN->>EN: forget_original_parent(tsk, &dead)
        Note over EN: 자식 프로세스들을 reaper(init)에게 재배정<br/>(reparenting)
        EN->>EN: tsk->exit_state = EXIT_ZOMBIE

        alt 부모가 SA_NOCLDWAIT 또는 SIG_IGN
            EN->>EN: tsk->exit_state = EXIT_DEAD (autoreap)
            EN->>RT: release_task(tsk) — 즉시 해제
        else 일반 종료
            EN->>PAR: do_notify_parent(tsk, SIGCHLD)
            Note over PAR: siginfo 구성:<br/>si_code = CLD_EXITED / CLD_KILLED<br/>si_status = exit_code or signal<br/>__wake_up_parent()로 wait() 깨움
        end
    end

    rect rgb(255, 220, 220)
        Note over DE,SCHED: === Phase 4: 최종 스케줄링 ===
        DE->>DE: exit_rcu() / exit_tasks_rcu_finish()
        DE->>SCHED: do_task_dead()
        Note over SCHED: __state = TASK_DEAD<br/>__schedule(SM_NONE) — 마지막 스케줄 호출<br/>→ 다른 태스크로 전환<br/>→ finish_task_switch()에서:<br/>   put_task_struct_rcu_user(prev)
    end

    Note over PAR: (나중에) wait4() / waitpid() 호출
    PAR->>RT: release_task(zombie)

    rect rgb(230, 255, 230)
        Note over RT: === Phase 5: task_struct 완전 제거 ===
        RT->>RT: __exit_signal() — 시그널 구조 정리, unhash
        RT->>RT: __unhash_process() — PID/태스크 리스트에서 제거
        RT->>RT: proc_flush_pid() — /proc 엔트리 제거
        RT->>RCU: put_task_struct_rcu_user(p)

        Note over RCU: RCU grace period 후:
        RCU->>RCU: delayed_put_task_struct()
        RCU->>RCU: __put_task_struct()
        Note over RCU: io_uring_free()<br/>cgroup_task_free()<br/>security_task_free()<br/>exit_creds()<br/>put_signal_struct()
        RCU->>RCU: free_task()
        Note over RCU: 커널 스택 해제<br/>arch_release_task_struct()<br/>free_task_struct()<br/>→ kmem_cache_free()
    end
```

### 4.2 리소스 해제 순서와 이유

```
do_exit() 내 리소스 해제 순서:

1. exit_mm()          ← 가장 먼저: 유저 메모리는 더 이상 접근 안 함
2. exit_sem()         ← IPC 세마포어 undo 리스트 정리
3. exit_shm()         ← 공유 메모리 세그먼트 분리
4. exit_files()       ← 열린 파일 닫기 (소켓, 파이프 포함)
5. exit_fs()          ← cwd, root 참조 해제
6. disassociate_ctty() ← 제어 터미널 분리 (세션 리더인 경우)
7. exit_nsproxy_namespaces() ← 네임스페이스 참조 해제
8. exit_thread()      ← 아키텍처별 정리 (I/O 비트맵 등)
9. exit_io_context()  ← 블록 I/O 스케줄러 컨텍스트

mm이 가장 먼저 해제되는 이유:
- 파일 close 등에서 유저 공간 접근이 필요 없음
- mm 해제 후 active_mm은 커널이 빌려쓸 수 있음
- COW 페이지 등 대량 메모리를 조기 반환
```

### 4.3 Zombie와 Wait의 관계

```mermaid
graph TB
    subgraph "프로세스 종료 → Zombie → 완전 제거"
        A["do_exit() 완료"]
        B["EXIT_ZOMBIE<br/>(task_struct만 남음)<br/>mm, files, fs 등은 이미 해제됨"]
        C["부모의 wait() / waitpid()"]
        D["release_task()"]
        E["EXIT_DEAD"]
        F["free_task()<br/>(메모리 완전 반환)"]
    end

    A -->|"exit_notify()"| B
    B -->|"부모가 SIGCHLD 받고"| C
    C -->|"do_wait() → wait_task_zombie()"| D
    D --> E --> F

    subgraph "autoreap 경로 (zombie 건너뜀)"
        AR["부모가 SA_NOCLDWAIT<br/>또는 SIGCHLD=SIG_IGN"]
    end
    A -->|"exit_notify()에서 autoreap"| AR
    AR -->|"바로 release_task()"| E

    subgraph "부모가 먼저 죽는 경우"
        RP["forget_original_parent()"]
        INIT["init (PID 1)이<br/>새 부모가 됨"]
    end
    RP --> INIT
    INIT -->|"init의 wait loop"| C
```

---

## 5. 프로세스의 전체 생애주기 통합 다이어그램

```mermaid
graph TB
    subgraph "탄생"
        FORK["fork() / clone()"]
        KCREATE["kthread_create()"]
        COPY["copy_process()<br/>dup_task_struct()"]
        SCHED_FORK["sched_fork()<br/>🔷 스케줄러: 초기화"]
        WAKE["wake_up_new_task()<br/>🔷 스케줄러: 런큐 삽입"]
    end

    subgraph "생존"
        RUN["TASK_RUNNING<br/>(실행 중 또는 실행 대기)"]
        SLEEP_I["TASK_INTERRUPTIBLE<br/>(시그널로 깨울 수 있는 수면)"]
        SLEEP_U["TASK_UNINTERRUPTIBLE<br/>(깨울 수 없는 수면)"]
        STOPPED["__TASK_STOPPED"]

        TICK["scheduler_tick()<br/>🔷 스케줄러: 타이머 틱"]
        PREEMPT["선점 검사<br/>🔷 TIF_NEED_RESCHED"]
        CS["context_switch()<br/>🔷 스케줄러: CPU 전환"]
        WAKEUP["try_to_wake_up()<br/>🔷 스케줄러: 깨우기"]
    end

    subgraph "죽음"
        EXIT["do_exit()"]
        ZOMBIE["EXIT_ZOMBIE"]
        DEAD["EXIT_DEAD"]
        FREE["free_task()<br/>메모리 반환"]
        TASK_DEAD_STATE["do_task_dead()<br/>🔷 스케줄러: 최종 전환"]
    end

    FORK --> COPY
    KCREATE --> COPY
    COPY --> SCHED_FORK --> WAKE --> RUN

    RUN <-->|"schedule()"| CS
    RUN -->|"wait_event() 등"| SLEEP_I
    RUN -->|"I/O, mutex 등"| SLEEP_U
    RUN -->|"SIGSTOP"| STOPPED
    SLEEP_I -->|"wake_up() / 시그널"| WAKEUP
    SLEEP_U -->|"wake_up()"| WAKEUP
    STOPPED -->|"SIGCONT"| WAKEUP
    WAKEUP --> RUN

    TICK -->|"time slice 소진"| PREEMPT
    PREEMPT -->|"resched_curr()"| CS

    RUN --> EXIT
    EXIT --> TASK_DEAD_STATE --> ZOMBIE
    ZOMBIE -->|"부모 wait()"| DEAD --> FREE

    style SCHED_FORK fill:#bbdefb
    style WAKE fill:#bbdefb
    style TICK fill:#bbdefb
    style PREEMPT fill:#bbdefb
    style CS fill:#bbdefb
    style WAKEUP fill:#bbdefb
    style TASK_DEAD_STATE fill:#bbdefb
```

> 🔷 표시는 스케줄러가 직접 관여하는 지점

---

## 6. 주요 함수 빠른 참조

| 함수 | 파일:라인 | 역할 |
|------|-----------|------|
| `kernel_clone()` | `kernel/fork.c:2610` | fork/clone 진입점 |
| `copy_process()` | `kernel/fork.c:1966` | 프로세스 깊은 복사 |
| `dup_task_struct()` | `kernel/fork.c:1052` | task_struct 메모리 복사 |
| `copy_thread()` | `arch/x86/kernel/process.c:170` | x86 레지스터 프레임 설정 |
| `sched_fork()` | `kernel/sched/core.c` | 스케줄러 초기화 |
| `wake_up_new_task()` | `kernel/sched/core.c` | 새 태스크 런큐 삽입 |
| `kernel_thread()` | `kernel/fork.c:2700` | 커널 스레드 생성 |
| `kthreadd()` | `kernel/kthread.c:815` | 커널 스레드 데몬 (PID 2) |
| `schedule()` | `kernel/sched/core.c:6954` | 자발적 스케줄링 진입 |
| `__schedule()` | `kernel/sched/core.c:6722` | 스케줄링 핵심 로직 |
| `pick_next_task()` | `kernel/sched/core.c:5971` | 다음 실행 태스크 선택 |
| `context_switch()` | `kernel/sched/core.c:5201` | mm + 레지스터 전환 총괄 |
| `__switch_to_asm()` | `arch/x86/entry/entry_64.S:178` | 스택/레지스터 물리 전환 |
| `__switch_to()` | `arch/x86/kernel/process_64.c:610` | x86 CPU 상태 전환 |
| `finish_task_switch()` | `kernel/sched/core.c:5075` | 전환 후 정리 (new 스택) |
| `do_exit()` | `kernel/exit.c:896` | 프로세스 종료 메인 |
| `exit_notify()` | `kernel/exit.c:736` | 부모 통지 + 자식 재배정 |
| `release_task()` | `kernel/exit.c:244` | zombie 완전 제거 |
| `do_task_dead()` | `kernel/sched/core.c:6880` | 마지막 스케줄 호출 |
| `free_task()` | `kernel/fork.c:528` | task_struct 메모리 해제 |
