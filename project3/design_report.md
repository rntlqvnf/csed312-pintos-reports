# [CSED312] OS project 3 Design Report

- 20180673 하재현
- 20180501 최진수

# Analysis on current Pintos system

현재 pintos system 상에서 virtual memory 자체는 하나도 구현되어 있지 않다.

따라서 전체적인 메모리 시스템에 대해서 설명하겠다.

## Memory Structure

핀토스는 virutal memory를 user memory와 kernel memory, 이 두개의 pool로 나눈다.

User memory의 경우 0x0에서 PHYS_BASE까지이다.

Kernel memory의 경우 PHYS_BASE에서 메모리의 끝 까지이다.

이때, Kernel address와 physical address의 mapping은 physical address = kernel virtual address - PHYS_BASE 으로 이루어진다.

그리고 제공되는 스택은 fixed size stack (1 page)이다. 

## Address Translation

핀토스의 주소 번역은 두 단계로 이루어진다.

이때, 상위 20bit는 번역에 쓰이고, 하위 12bit는 offset에 쓰인다. (Page size가 4KB이므로)

첫 번째 단계는 상위 10bit를 page directory index로 사용하여 page directory를 참조 해, page table을 얻어낸다.

두 번째 단계는 그 다음 10bit를 page table index로 사용하여 첫 번째 단계에서 구한 page table을 참조해, page를 얻어낸다.

만약 이 lookup이 실패하면 page fault가 발생한다.

```c
static void
page_fault (struct intr_frame *f) 
{
  ...

  /* To implement virtual memory, delete the rest of the function
     body, and replace it with code that brings in the page to
     which fault_addr refers. */
  printf ("Page fault at %p: %s error %s page in %s context.\n",
          fault_addr,
          not_present ? "not present" : "rights violation",
          write ? "writing" : "reading",
          user ? "user" : "kernel");
  kill (f);
}
```

현재로써는 page fault가 발생할 시, 곧바로 process가 종료된다.

## Loading Program

현재 핀토스는 실행에 필요한 데이터를 모두 메모리에 적재한 후 실행하고 있다.

이로인해 메모리가 효율적으로 사용되지 못하고 있다.

```c
static bool
load_segment (struct file *file, off_t ofs, uint8_t *upage,
              uint32_t read_bytes, uint32_t zero_bytes, bool writable) 
{
  ASSERT ((read_bytes + zero_bytes) % PGSIZE == 0);
  ASSERT (pg_ofs (upage) == 0);
  ASSERT (ofs % PGSIZE == 0);

  file_seek (file, ofs);
  while (read_bytes > 0 || zero_bytes > 0) 
    {
      /* Calculate how to fill this page.
         We will read PAGE_READ_BYTES bytes from FILE
         and zero the final PAGE_ZERO_BYTES bytes. */
      size_t page_read_bytes = read_bytes < PGSIZE ? read_bytes : PGSIZE;
      size_t page_zero_bytes = PGSIZE - page_read_bytes;

      /* Get a page of memory. */
      uint8_t *kpage = palloc_get_page (PAL_USER);
      if (kpage == NULL)
        return false;

      /* Load this page. */
      if (file_read (file, kpage, page_read_bytes) != (int) page_read_bytes)
        {
          palloc_free_page (kpage);
          return false; 
        }
      memset (kpage + page_read_bytes, 0, page_zero_bytes);

      /* Add the page to the process's address space. */
      if (!install_page (upage, kpage, writable)) 
        {
          palloc_free_page (kpage);
          return false; 
        }

      /* Advance. */
      read_bytes -= page_read_bytes;
      zero_bytes -= page_zero_bytes;
      upage += PGSIZE;
    }
  return true;
}
```

`load_segment`는 메모리 적재를 담당하는 함수이다.

## Access File

메모리 매핑이 구현되어있지 않다.

# Solutions for each requirement

## 1. Frame Table

## 2. Lazy Loading

## 3. Supplemental Page Table

## 4. Stack Growth

## 5. File Memory Mapping

## 6. Swap Table

## 7. On Process Termination