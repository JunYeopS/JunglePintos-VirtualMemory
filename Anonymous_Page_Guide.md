# Anonymous Page 구현 가이드

## 개요

이 프로젝트에서는 디스크 기반이 아닌 익명 페이지(Anonymous Page)를 구현합니다.

**익명 매핑(Anonymous Mapping)이란?**
- 백업 파일이나 디바이스가 없는 메모리 매핑
- 파일 기반 페이지와 달리 이름이 있는 파일 소스가 없음
- 실행 파일의 스택과 힙에 사용됨

---

## 구조체

### `struct anon_page` (`include/vm/anon.h`)
- 익명 페이지를 설명하는 구조체
- 현재는 비어 있지만, 필요한 정보나 상태를 저장하기 위해 멤버를 추가할 수 있음

### `struct page` (`include/vm/page.h`)
- 페이지의 일반적인 정보를 포함
- 익명 페이지의 경우, `struct anon_page anon`이 페이지 구조체에 포함됨

---

## Lazy Loading (지연 로딩)

### 개념
- 메모리 로딩을 실제로 필요한 시점까지 연기하는 디자인
- 페이지는 할당되지만 (page struct가 존재), 전용 물리 프레임은 없음
- 실제 내용은 page fault가 발생할 때 로드됨

### 페이지 초기화 흐름

1. **`vm_alloc_page_with_initializer` 호출**
   - 커널이 새 페이지 요청을 받을 때 호출됨
   - 페이지 타입에 따라 적절한 initializer를 설정
   
2. **Page Fault 발생**
   - 프로그램이 아직 내용이 없는 페이지에 접근하려 할 때
   
3. **`uninit_initialize` 호출**
   - Fault 처리 과정에서 호출됨
   - 이전에 설정한 initializer를 호출:
     - Anonymous pages: `anon_initializer`
     - File-backed pages: `file_backed_initializer`

### 페이지 생명 주기
```
initialize -> (page_fault -> lazy-load -> swap-in -> swap-out -> ...) -> destroy
```

각 전환마다 페이지 타입(VM_TYPE)에 따라 필요한 절차가 다름

---

## 실행 파일의 Lazy Loading

### `VM_UNINIT` 페이지 타입
- **목적**: Lazy loading을 지원하기 위한 페이지 타입
- **위치**: `include/vm/vm.h`
- **특징**: 모든 페이지는 처음에 `VM_UNINIT` 페이지로 생성됨

### `struct uninit_page`
- **위치**: `include/vm/uninit.h`
- **관련 함수**: `include/vm/uninit.c`
- 초기화되지 않은 페이지를 생성, 초기화, 삭제하는 함수들

### Page Fault 처리

1. **`page_fault`** (`userprog/exception.c`)
   - Page fault handler

2. **`vm_try_handle_fault`** (`vm/vm.c`)
   - 유효한 page fault인지 확인
   - 세 가지 bogus page fault 케이스:
     - Lazy-loaded 페이지
     - Swapped-out 페이지
     - Write-protected 페이지

3. **Lazy-loaded 페이지 처리**
   - `lazy_load_segment`를 호출하여 세그먼트를 lazy load

---

## 구현해야 할 함수들

### 1. `vm_alloc_page_with_initializer`

```c
bool vm_alloc_page_with_initializer (enum vm_type type, void *va,
        bool writable, vm_initializer *init, void *aux);
```

**역할:**
- 주어진 타입의 초기화되지 않은 페이지 생성
- 적절한 initializer를 가져와서 `uninit_new`를 호출
- 페이지 구조체를 프로세스의 supplementary page table에 삽입

**힌트:** `VM_TYPE` 매크로 사용 (`vm.h`)

---

### 2. `uninit_initialize`

```c
static bool uninit_initialize (struct page *page, void *kva);
```

**역할:**
- 첫 번째 fault 시 페이지 초기화
- `vm_initializer`와 `aux`를 가져와서 함수 포인터를 통해 `page_initializer` 호출

**참고:** 템플릿 코드가 완전한 구현을 제공하지만, 디자인에 따라 수정 필요할 수 있음

---

### 3. `vm_anon_init` (`vm/anon.c`)

```c
void vm_anon_init (void);
```

**역할:**
- Anonymous page 서브시스템 초기화
- Anonymous page 관련 설정

---

### 4. `anon_initializer` (`vm/anon.c`)

```c
bool anon_initializer (struct page *page, enum vm_type type, void *kva);
```

**역할:**
- `page->operations`에 anonymous page의 핸들러 설정
- `anon_page`의 정보 업데이트 (현재는 빈 구조체)
- Anonymous pages (`VM_ANON`)의 initializer로 사용됨

---

### 5. `load_segment` (`userprog/process.c`)

```c
static bool load_segment (struct file *file, off_t ofs, uint8_t *upage,
        uint32_t read_bytes, uint32_t zero_bytes, bool writable);
```

**역할:**
- 실행 파일로부터 세그먼트 로딩 구현
- 모든 페이지는 lazily 로드되어야 함 (커널이 page fault를 가로챌 때만)
- 루프 내에서:
  - 파일에서 읽을 바이트 수 계산
  - 0으로 채울 바이트 수 계산
  - `vm_alloc_page_with_initializer`를 호출하여 pending 페이지 객체 생성
  - `aux` 인자에 바이너리 로딩에 필요한 정보를 담은 구조체 설정

---

### 6. `lazy_load_segment` (`userprog/process.c`)

```c
static bool lazy_load_segment (struct page *page, void *aux);
```

**역할:**
- 실행 파일 페이지의 initializer
- Page fault 발생 시 호출됨
- `aux`는 `load_segment`에서 설정한 정보
- 이 정보를 사용하여:
  - 세그먼트를 읽을 파일을 찾음
  - 세그먼트를 메모리로 읽음

---

### 7. `setup_stack` 수정 (`userprog/process.c`)

**역할:**
- 새로운 메모리 관리 시스템에 맞게 스택 할당 조정
- 첫 번째 스택 페이지는 lazy하게 할당할 필요 없음
- 로드 타임에 command line arguments와 함께 할당 및 초기화
- 스택을 식별하는 방법 제공 필요
  - `vm_type`의 auxiliary markers 사용 가능 (예: `VM_MARKER_0`)

---

### 8. `vm_try_handle_fault` 수정

**역할:**
- Supplemental page table을 참조하여 faulted 주소에 해당하는 page struct 찾기
- `spt_find_page` 사용

---

## Supplemental Page Table - Revisit

프로세스를 생성(자식 프로세스 생성)하거나 파괴할 때 필요한 copy 및 clean up 작업 지원

### 9. `supplemental_page_table_copy` (`vm/vm.c`)

```c
bool supplemental_page_table_copy (struct supplemental_page_table *dst,
    struct supplemental_page_table *src);
```

**역할:**
- `src`에서 `dst`로 supplemental page table 복사
- 자식이 부모의 실행 컨텍스트를 상속받을 때 사용 (`fork()`)
- `src`의 각 페이지를 순회하여 `dst`에 정확한 복사본 생성
- Uninit 페이지를 할당하고 즉시 claim 필요

---

### 10. `supplemental_page_table_kill` (`vm/vm.c`)

```c
void supplemental_page_table_kill (struct supplemental_page_table *spt);
```

**역할:**
- Supplemental page table이 보유하던 모든 리소스 해제
- 프로세스 종료 시 호출 (`process_exit()` in `userprog/process.c`)
- 페이지 엔트리를 순회하며 `destroy(page)` 호출
- 이 함수에서는 실제 page table(pml4)과 물리 메모리(palloc-ed memory)는 신경쓰지 않아도 됨 (호출자가 처리)

---

## Page Cleanup

### 11. `uninit_destroy` (`vm/uninit.c`)

```c
static void uninit_destroy (struct page *page);
```

**역할:**
- 초기화되지 않은 페이지의 destroy 연산 핸들러
- 프로세스 종료 시에도 uninit 페이지가 존재할 수 있음
- 페이지의 vm type을 확인하고 적절히 처리
- Page struct가 보유하던 리소스 해제

**참고:** 우선은 anonymous 페이지만 처리, 나중에 file-backed 페이지도 처리

---

### 12. `anon_destroy` (`vm/anon.c`)

```c
static void anon_destroy (struct page *page);
```

**역할:**
- Anonymous 페이지가 보유하던 리소스 해제
- Page struct 자체는 명시적으로 해제하지 않음 (호출자가 처리)

---

## 테스트 통과 기준

### 중간 단계
Anonymous page 관련 모든 요구사항 구현 후:
- Project 2의 `fork`를 제외한 모든 테스트 통과

### 최종 단계
Supplemental Page Table 및 Page Cleanup 구현 후:
- Project 2의 **모든 테스트** 통과

---

## 주요 개념 정리

### VM_TYPE
- `VM_UNINIT`: 초기화되지 않은 페이지
- `VM_ANON`: Anonymous 페이지
- `VM_FILE`: File-backed 페이지

### Auxiliary Markers
- `VM_MARKER_0` 등을 사용하여 페이지 식별 (예: 스택 페이지)

### Page Fault Types
1. **Lazy-loaded**: 아직 로드되지 않은 페이지
2. **Swapped-out**: 스왑 아웃된 페이지
3. **Write-protected**: 쓰기 보호된 페이지

---

## 구현 순서 권장사항

1. ✅ `vm_alloc_page_with_initializer` 구현
2. ✅ `vm_anon_init` 및 `anon_initializer` 구현
3. ✅ `load_segment` 및 `lazy_load_segment` 구현
4. ✅ `setup_stack` 수정
5. ✅ `vm_try_handle_fault` 수정
6. ✅ 테스트 (fork 제외)
7. ✅ `supplemental_page_table_copy` 구현
8. ✅ `supplemental_page_table_kill` 구현
9. ✅ `uninit_destroy` 및 `anon_destroy` 구현
10. ✅ 최종 테스트 (모든 테스트)

---

## 주의사항

- 각 페이지 타입마다 초기화 루틴이 다름
- Life cycle의 각 전환마다 VM_TYPE에 따라 다른 절차 필요
- Uninit 페이지는 다른 페이지 객체로 변환되지만, 프로세스 종료 시에도 남아있을 수 있음
