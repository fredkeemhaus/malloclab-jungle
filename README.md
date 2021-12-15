## 말록랩 과제 설명
- http://csapp.cs.cmu.edu/3e/malloclab.pdf
- 출처: CMU (카네기멜론)
## 참고자료
- http://csapp.cs.cmu.edu/3e/labs.html
- CMU 강의자료 전체



## Malloc이란? 

> C언어의 동적 메모리 할당을 의미한다. (Dynamic Memory Allocation)


-  배열의 크기를 미리 알 수 없다면, 어떻게 메모리를 할당해주어야할까?
	→ 이 때 쓰이는 것이 '동적 메모리 할당'


동적 메모리 할당기는 힙(heap)이라고 하는 프로세스의 가상 메모리 영역을 관리한다.

### 동적 메모리 할당기 만들기

**할당기는 아래와 같이 크게 2종류가 있다.**

- `명시적 할당기`: 애플리케이션이 명시적으로 할당된 블록을 반환해줄 것을 요구함.
	- C에서는 malloc 함수로 블록을 할당, free 함수로 블록을 반환
    <br />
- `묵시적 할당기`: 언제 할당된 블록이 더 이상 프로그램에 의해 사용되지 않고, 블록을 반환하는지를 할당기가 검출할 수 있을 것을 요구한다.
	- 묵시적 할당기는 가비지 컬렉션(gabage collection)이라고 알려져 있다. 자동으로 사용하지 않은 할당된 블록을 반환시켜주는 작업을 한다.

--- 
**명시적 할당기의 요구사항과 목표는 다음과 같다. **
> - `임의의 요청 순서 처리`: 가용할 수 있는 블럭이 할당 요청에 의해 할당되어야 한다. 임의의 순서로 할당과 반환요청을 할 수가 있다.
- `요청에 즉시 응답하기`: 요청기는 무조건 요청에 즉시 응답. 그러므로 요청기는 요청을 나중으로 미룰 수 없다.
- `힙만 사용하기`: 확장성을 갖기 위해서는 할당기가 사용하는 비확장성 자료 구조들은 힙 자체에 저장되어야 한다.
- `블럭 정렬하기`: 할당기는 블럭들을 이들이 어떤 종류의 데이터 객체라도 저장할 수 있도록 하는 방식으로 정렬해야 한다.
-  `할당된 블럭을 수정하지 않기`: 일단, 블럭이 할당되면 이들을 수정하거나 이동하지 않는다.

## Implicit Malloc

> implicit, explicit, seglist 등 여러 방법이 있지만, 일단 `implicit` 방법부터 구현해 보겠습니다.


`mm.c`파일에서 조작해야 할 함수는 아래와 같다.

>1. `mm_init`
2. `extend_heap`
3. `mm_free`
4. `coalesce`
5. `mm_malloc`
6. `find_fit`
7. `mm_realloc`
8. `place`


### 기본 설계

기본적으로 `mm_init`은 할당기를 초기화하고 성공이면 0, 실패면 -1을 리턴한다. 초기화 하는 과정에는 가용 리스트의 `prologue header`, `prologue footer`, `epilogue header`를 만드는 과정이 포함된다.

이때 최소 블록 크기는 16바이트로 정한다. 가용 리스트는 묵시적 가용 리스트로 구성된다.

---

### mm_init
할당기의 초기화 함수. 묵시적 가용 리스트는 아래와 같은 구조를 가지고 있다.

![](https://images.velog.io/images/fredkeemhaus/post/e86b9c8b-ee03-40bc-96dc-5b9a2a0d8835/malloc_block.png)

>-  `header`: 블록의 할당 여부와 크기를 저장한다.
- `payload`: 할당된 블록에만 값이 들어있다.
- `padding`: Double Word 정렬을 위해 optional하게 존재
- `footer`: 경계 태그로 사용되며, `header`의 값이 복사되어있다. 블록을 반환할 때 이전 블록을 상수 시간 내에 탐색할 수 있도록 하는 장치.
<br />
(Footer가 없으면 모든 블록의 size 정보가 다르기 때문에 이전 Header를 찾아내기 위해서는 힙의 시작점부터 순차적으로 탐색해야 하는 불편이 생긴다)

`mm_init` 함수에 필요한 것

- 힙을 초기 사이즈만큼 세팅한다.
- 가용 리스트에 패딩을 넣는다.
- 가용 리스트에 프롤로그 헤더를 넣는다.
- 가용 리스트에 프롤로그 푸터를 넣는다.
- 가용 리스트에 에필로그 헤더를 넣는다.
- 힙에 블록을 할당하기 위해 사이즈를 적당히 한번 늘린다.


![](https://images.velog.io/images/fredkeemhaus/post/5b0baca8-a19b-4949-862e-9ce1c0556061/malloc_init.png)

다시 한 번 정리하면,

1. `프롤로그 블록` + `에필로그 블록`을 초기화 시 할당. 이 블록은 `header`와 `footer`로 구성된 8byte를 할당 블럭이며, 할당기 프로그램이 종료될 때까지 반환되지 않는다. (이유는 경계 조건을 무시하기 위해)

2. 또한 Double Word 정렬을 위해 사용하지 않는 패딩 블록을 맨 앞에 붙여놓는다.
3. 힙이 확장될 때 에필로그 블록은 확장된 힙의 마지막에 위치하도록 한다.

---
- `mm_init`

```c
int mm_init(void) {
    if ((heap_listp = mem_sbrk(4*WSIZE)) == (void*)-1) {
        return -1;
    }
    PUT(heap_listp, 0);
    PUT(heap_listp + (1*WSIZE), PACK(DSIZE,1)); // prologue header 생성.
    PUT(heap_listp + (2*WSIZE), PACK(DSIZE,1)); // prologue footer 생성.
    PUT(heap_listp + (3*WSIZE), PACK(0,1)); // epilogue block header
    heap_listp+= (2*WSIZE);

    if (extend_heap(CHUNKSIZE/WSIZE)==NULL)
        return -1;
    return 0;
}
```
![](https://images.velog.io/images/fredkeemhaus/post/5903ccdb-3214-4bf8-a5c5-121b5ae64c20/Malloc%20Lab-2.jpg)

![](https://images.velog.io/images/fredkeemhaus/post/8050955e-cb25-4473-a3db-7e3607735857/Malloc%20Lab-3.jpg)

---

### extend_heap
이 함수는 2가지 경우에 호출될 수 있다.

1. 힙이 초기화될 때
	- 초기화 후에 초기 가용 블럭을 생성하기 위해 호출된다.
    
 2. 요청한 크기의 메모리 할당을 위해 **충분한 공간을 찾지 못했을 때**
 	 - Double word 정렬을 위해 8의 배수로 반올림하고, 추가 힙 공간을 요청한다.
     
---

- `extend_heap`

```c
static void *extend_heap(size_t words) { 
    char *bp;
    size_t size;
    size = (words % 2) ? (words + 1) * WSIZE : words * WSIZE; 
    if ( (long)(bp = mem_sbrk(size)) == -1) {
        return NULL;
    }
    
    PUT(HDRP(bp), PACK(size,0)); // free block header 생성.
    PUT(FTRP(bp), PACK(size,0)); // free block footer 생성.
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0,1));

    return coalesce(bp);
}
```

![](https://images.velog.io/images/fredkeemhaus/post/d758acce-0427-4247-9175-a7fb70304543/Malloc%20Lab-4.jpg)

---

### mm_free / coalesce

블록을 반환하고(= `mm_free`) 경계 태그(`Header`, `Footer`)를 사용해서 상수 시간 내에 인접한 가용 블록(free)들과 통합한다. (= `coalesce`)

여기서는 4가지 Case가 존재한다.

> - Case1: 이전과 다음 블럭이 모두 할당되어 있다.
 	**- 현재 블럭만 가용 상태로 변경한다.**
    
> - Case2: 이전 블럭은 할당 상태, 다음 블럭은 가용 상태이다.
 	**- 현재 블록과 다음 블럭을 통합.**
    
> - Case3: 이전 블럭은 가용 상태, 다음 블럭은 할당 상태이다.
 	**- 이전 블록과 현재 블럭을 통합.**
    
> - Case4: 이전 블럭과 다음 블럭 모두 가용 상태이다.
 	**- 이전 블럭, 현재블럭 다음 블럭 모두 통합.**
    
![](https://images.velog.io/images/fredkeemhaus/post/6e1b776e-b700-4293-8407-4f5b9c9caa5a/Malloc%20Lab-5.jpg)


프롤로그와 에필로그를 할당 상태로 초기화했기 때문에, `free`를 요청한 블록의 주소 `bp`가 힙의 시작 부분 또는 끝 부분에 위치하는 경계조건(Edge Condition)을 무시할 수 있게 된다.

- `mm_free` / `coalesce`

```c
static void place(void *bp, size_t asize) {
    size_t currSize = GET_SIZE(HDRP(bp));
    if ( (currSize - asize) >= (2 * DSIZE)) {
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(currSize - asize, 0));
        PUT(FTRP(bp), PACK(currSize - asize, 0));
    }
    else{
        PUT(HDRP(bp), PACK(currSize, 1));
        PUT(FTRP(bp), PACK(currSize, 1));
    }
}

static void *coalesce(void *bp) {
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size =  GET_SIZE(HDRP(bp));

    if (prev_alloc && next_alloc) {
        return bp;
    }
    else if (prev_alloc && !next_alloc) {
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size, 0));
        PUT(FTRP(bp), PACK(size, 0));
    }
    else if(!prev_alloc && next_alloc) {
        size+= GET_SIZE(HDRP(PREV_BLKP(bp))); 
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    else {
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) + GET_SIZE(FTRP(NEXT_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    return bp;
}
```

![](https://images.velog.io/images/fredkeemhaus/post/394c0e48-b739-46c1-acce-8b82e2c195bf/Malloc%20Lab-6.jpg)


---

### mm_malloc / find_fit

요청한 size 만큼의 공간이 있는 메모리 블록을 할당하는 함수. 아래의 기능을 구현한다.

- `Header`와 `Footer` 공간을 위해 최소 16bytes(4워드)의 크기를 유지하도록 한다.

- Double Word 정렬을 유지하기 위해 8의 배수로 반올림한 메모리 크기를 할당한다.

- 메모리 할당 크기를 조절한 후, 가용 블록 리스트를 검색하여 적합한 블록을 찾았을 때 배치한다.
	- `find_fit` 함수
	- 여기에서는 First Fit 방식을 사용한다.

> 가용 상태이고, 요청한 사이즈만큼 충분한 공간이 있을 때 해당 블록 포인터(`bp`)를 반환

- OPTION : 찾은 블록에 배치한 후 여유공간이 충분하다면 분할한다. (= `place`)


- `mm_malloc`

```c
void *mm_malloc(size_t size)
{
    size_t asize;      /* Adjusted block size */
    size_t extendsize; /* Amount to extend heap if no fit */
    char *bp;

    /* Ignore spurious requests */
    if (size == 0)
        return NULL;

    /* Adjust block size to include overhead and alignment reqs. */
    if (size <= DSIZE)
        asize = 2 * DSIZE;
    else
        asize = DSIZE * ((size + (DSIZE) + (DSIZE - 1)) / DSIZE);

    /* Search the free list for a fit */
    if ((bp = find_fit(asize)) != NULL)
    {
        place(bp, asize);
        return bp;
    }

    /* No fit found. Get more memory and place the block */
    extendsize = MAX(asize, CHUNKSIZE);
    if ((bp = extend_heap(extendsize / WSIZE)) == NULL)
        return NULL;
    place(bp, asize);
    return bp;
}
```



- `find_fit`

```c
static void *find_fit(size_t asize)
{
    /* first-fit search */
    char *bp;

    // start the search from the beginning of the heap
    for (bp = heap_listp; GET_SIZE(HDRP(bp)) > 0; bp = NEXT_BLKP(bp))
    {
        if (!GET_ALLOC(HDRP(bp)) && (asize <= GET_SIZE(HDRP(bp))))
        {
            return bp;
        }
    }
    return NULL; /* No fit */
}
```


---
### mm_realloc

`realloc`은 기존에 할당되어 있던 블록을 새로운 사이즈로 재할당하는 함수이다.
여기에는 2가지 경우가 고려되어야 한다.

> 1. 새로 할당하려는 size가 기존 size보다 작을 때
	**- 기존 정보를 일부만 복사한 뒤 새로운 포인터를 반환
    
> 2. 새로 할당하려는 size가 기존 size보다 클 때
	**- 기존 정보를 모두 복사한 뒤 새로운 포인터를 반환**
    
    
- `mm_realloc`

```c
void *mm_realloc(void *bp, size_t size) {
    if(size <= 0) { 
        mm_free(bp);
        return 0;
    }
    if(bp == NULL) {
        return mm_malloc(size);
    }
    void *newp = mm_malloc(size); 
    if(newp == NULL) {
        return 0;
    }
    size_t oldsize = GET_SIZE(HDRP(bp));
    if(size < oldsize) {
    	oldsize = size; 
	}
    memcpy(newp, bp, oldsize);
    mm_free(bp);

    return newp;
}
```

---

### place


`place` 함수는 블록이 할당되거나 재할당될 때 여유 공간을 판단하여 분할해 주는 함수이다.

여기에도 2가지 경우가 있을 수 있다.

> 1. 현재 블록에 size를 할당한 후에 남은 size가 최소 블록 size(header와 footer를 포함한 4워드)보다 크거나 같을 때
	**- 현재 블록의 size 정보를 변경하고 남은 size를 새로운 가용 블록으로 분할한다**
    
> 2. 현재 블록에 size를 할당한 후에 남은 size가 최소 블록 size보다 작을 때
**	- 현재 블록의 size 정보만 바꿔준다**


```c

static void place(void *bp, size_t asize) {
    size_t currSize = GET_SIZE(HDRP(bp));
    if ( (currSize - asize) >= (2 * DSIZE)) {
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(currSize - asize, 0));
        PUT(FTRP(bp), PACK(currSize - asize, 0));
    }
    else{
        PUT(HDRP(bp), PACK(currSize, 1));
        PUT(FTRP(bp), PACK(currSize, 1));
    }
}
```

--- 
### 👉요약

#### Implicit, Explicit, Segreagated

다시 한번 정리하자면,

- `Implicit`은 모든 블럭을 연결시킨다. 그래서 탐색하는데 모든 블럭 갯수만큼의 연산이 필요하다.

- `Explicit`은 가용 블럭(free block)을 연결시킨다.
	그래서 탐색하는데 프리 블럭의 갯수만큼의 연산이 필요하다. 이 때, free block들은 이중리스트로 연결되어 있어야 하고 메모리는 최소 4word(16byte)의 크기를 갖는다. 이는 헤더와 푸터 그리고 전 블럭 주소, 다음 블럭 주소를 이중리스트로 담기 위함이다. 할당된 블럭은 연결을 끊음으로 전 블럭주소, 다음 블럭 주소를 담을 필요도 공간도 필요가 없다.
    
    새로운 블럭을 stack처럼 맨 앞으로 LIFO 구조로 연결시킬 수 있고, 블럭의 주소값 순서로 정렬시켜 연결시키는 방법이 있다.
    
    
- `Segreagated` size 별로 따른 free list를 만들어서 각 메모리 사이즈 별로 관리한다.
그래서 가장 빠른 탐색속도를 가진다. buddy list 방법이 가장 대표적으로 2의 승수별로 블록 사이즈를 나누어 각각 연결해준다. 탐색시에는 들어가 블록이 속해 있는 클라스부터 탐색하기 시작해서 더 큰 사이즈 클라스쪽으로 전부 탐색하면 된다.

---

#### First fit, Next fit, Best fit, Good fit

- `First fit`은 처음부터 출발해, 적절한 블록을 만나면 그 블록을 선택한다.
탐색은 가장 빠르지만, 최적의 메모리 사용은 아닐 확률이 있다.

- `Next fit`은 저번 탐색 지점을 기억해, 그곳에서 출발해 충분한 블록을 만나면 그 블록을 선택한다.
최근 사용한 메모리를 위주로 비할당시킬 확률이 높다면 유리할 수 있다. 최적의 메모리 사용은 아닐 확률이 있다.

- `Best fit`은 처음부터 출발해 모든 블록을 탐색한 뒤, 가장 적절한 블록을 선택한다.
완전 탐색을 해야하지만, external fragementation은 줄일 수 있다.


--- 

### 👏TIL


해당 코드들은 책에 나와있는 내용대로 구현한 것이다. 여기까지 완성하면 `Implicit malloc` 프로그램을 정상적으로 구동할 수 있게 되며, `54/100` 을 받을 수 있다. (`first_fit`)

