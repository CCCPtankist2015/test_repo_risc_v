# Лабораторная работа 1. RISC-V в QEMU: уровни привилегий, ECALL и trap-handling

---

## Назначение работы

Эта лабораторная работа нужна для того, чтобы **руками пройти путь** от старта процессора в `M-mode` до выполнения кода в `U-mode`, затем инициировать переход вверх через `ECALL`, обработать trap и вернуться назад.

Работа выполняется на виртуальной платформе `virt` в `qemu-system-riscv64`, которая предназначена для full-system эмуляции RISC-V и используется как стандартная учебная и прототипная платформа.

Практический смысл работы:

- увидеть, что уровни привилегий — это не абстрактная теория, а реальные состояния процессора с отдельными регистрами управления;
- понять, как инструкция `ECALL` превращается в исключение `environment call` и почему это основа системного вызова;
- подготовить базу для следующих лабораторных, где поверх этих механизмов будет рассматриваться seL4 на RISC-V.

---

## Результат работы

В конце лабораторной должен получиться минимальный bare-metal проект, который:

1. стартует в `M-mode` после запуска в QEMU;
2. настраивает `mtvec` и переходит в `S-mode`;
3. в `S-mode` настраивает `stvec` и переходит в `U-mode`;
4. в `U-mode` выполняет `ECALL`;
5. в `S-mode` принимает trap, читает `scause`, `sepc`, `sstatus`, изменяет возвращаемое значение и делает `SRET` обратно в пользовательский код.

Итогом должны быть:

- текстовый вывод в консоль QEMU через UART;
- исходные файлы проекта;
- понимание, какие шаги относятся к ISA, какие — к ABI, а какие уже могли бы быть обёрнуты в API.

---

## Что именно изучается

В этой работе используются следующие элементы привилегированной архитектуры RISC-V:

| Элемент | Назначение |
|---------|-----------|
| `M-mode`, `S-mode`, `U-mode` | три базовых уровня привилегий |
| `mstatus`, `sstatus` | регистры состояния привилегий |
| `mepc`, `sepc` | адреса возврата после исключения |
| `mtvec`, `stvec` | адреса обработчиков trap |
| `medeleg` | механизм делегирования исключений из M-mode в S-mode |
| `MRET`, `SRET` | инструкции возврата из привилегированного обработчика |
| `ECALL` | инструкция генерации исключения "environment call" |

---

## Общая логика лабораторной

```
1. QEMU стартует образ → ядро начинает в M-mode
2. M-mode готовит среду → передаёт управление в S-mode через MRET
3. S-mode настраивает обработчик → передаёт управление в U-mode через SRET
4. U-mode вызывает псевдо-системный вызов через ECALL
5. Исключение делегируется в S-mode → обрабатывается → возврат в U-mode
```

Это минимальная модель: **пользовательский код → trap → привилегированный обработчик → возврат**.

---

## Требования к окружению

### Установка (Ubuntu / Debian)

```bash
sudo apt update
sudo apt install -y qemu-system-misc gcc-riscv64-unknown-elf binutils-riscv64-unknown-elf make
```

> **Скриншот:** результат установки пакетов
> `<!-- вставить assets/install.png -->`

### Проверка

```bash
qemu-system-riscv64 --version
riscv64-unknown-elf-gcc --version
```

> **Скриншот:** вывод версий
> `<!-- вставить assets/versions.png -->`

Если toolchain из пакетов недоступен — можно использовать prebuilt RISC-V GNU toolchain. Важно, чтобы были доступны `gcc`, `ld`, `objdump` для bare-metal `elf`.

---

## Структура проекта

```text
lab1-riscv-qemu/
├── Makefile
├── linker.ld
├── start.S
├── trap.S
├── uart.c
├── uart.h
└── main.c
```

| Файл | Назначение |
|------|-----------|
| `Makefile` | сборка и запуск проекта |
| `linker.ld` | размещение секций в памяти платформы `virt` |
| `start.S` | старт в M-mode, настройка `mtvec`, `medeleg`, переход в S-mode |
| `trap.S` | низкоуровневый trap entry/exit код |
| `uart.c/h` | минимальный вывод в последовательный порт QEMU `virt` |
| `main.c` | код в S-mode, код в U-mode, обработчик trap в C |

---

## Архитектурная договорённость

Чтобы снизить порог входа, в лабораторной используется следующая упрощённая модель:

- `M-mode` выполняет только начальную инициализацию и делегирует пользовательские исключения в `S-mode` через `medeleg`.
- Вся содержательная обработка `ECALL` происходит в `S-mode`.
- Пользовательский код в `U-mode` не имеет доступа к privileged-инструкциям и обращается вверх только через `ECALL`.

Это приближает модель к реальной логике современной ОС: машинный уровень минимален, ядро живёт в supervisor-уровне.

---

# Часть 1. Создание проекта

## Шаг 1. Создайте каталог и пустые файлы

```bash
mkdir -p lab1-riscv-qemu
cd lab1-riscv-qemu
touch Makefile linker.ld start.S trap.S uart.c uart.h main.c
```

> **Скриншот:** структура созданных файлов
> `<!-- вставить assets/step1_files.png -->`

---

## Шаг 2. Создайте `linker.ld`

Минимальный linker script для загрузки образа начиная с `0x80000000` — стандартный адрес для `-kernel` на QEMU `virt`.

> **Скриншот:** содержимое linker.ld в редакторе
> `<!-- вставить assets/step2_linker.png -->`

```ld
OUTPUT_ARCH(riscv)
ENTRY(_start)

SECTIONS
{
    . = 0x80000000;

    .text : {
        *(.text.init)
        *(.text*)
    }

    .rodata : {
        *(.rodata*)
    }

    .data : {
        *(.data*)
    }

    .bss : {
        *(.bss*)
        *(COMMON)
    }

    . = ALIGN(16);
    . += 16K;
    _stack_top = .;
}
```

**Что здесь важно:**

- `ENTRY(_start)` — точка входа — символ `_start` из `start.S`;
- `. = 0x80000000` — типичный адрес старта для `virt`;
- `_stack_top` — используется как вершина стека.

---

# Часть 2. Минимальный вывод через UART

На платформе `virt` QEMU эмулирует UART16550 по адресу `0x10000000`.

## Шаг 3. Создайте `uart.h`

> **Скриншот:** содержимое uart.h
> `<!-- вставить assets/step3_uart_h.png -->`

```c
#ifndef UART_H
#define UART_H

#include <stdint.h>

void uart_init(void);
void uart_putc(char c);
void uart_puts(const char *s);
void uart_puthex(uint64_t value);
void uart_putdec(uint64_t value);

#endif
```

## Шаг 4. Создайте `uart.c`

> **Скриншот:** содержимое uart.c
> `<!-- вставить assets/step4_uart_c.png -->`

```c
#include "uart.h"

#define UART0 0x10000000L

static volatile unsigned char *const uart = (volatile unsigned char *)UART0;

void uart_init(void) {
    /* QEMU virt позволяет выводить байты без дополнительной настройки */
}

void uart_putc(char c) {
    while ((uart[5] & 0x20) == 0);
    uart[0] = c;
}

void uart_puts(const char *s) {
    while (*s) {
        if (*s == '\n')
            uart_putc('\r');
        uart_putc(*s++);
    }
}

void uart_puthex(uint64_t value) {
    static const char hex[] = "0123456789ABCDEF";
    uart_puts("0x");
    for (int i = 15; i >= 0; --i) {
        uint64_t nibble = (value >> (i * 4)) & 0xF;
        uart_putc(hex[nibble]);
    }
}

void uart_putdec(uint64_t value) {
    char buf[32];
    int i = 0;
    if (value == 0) { uart_putc('0'); return; }
    while (value > 0) { buf[i++] = '0' + (value % 10); value /= 10; }
    while (i--) uart_putc(buf[i]);
}
```

---

# Часть 3. Старт в M-mode

## Шаг 5. Создайте `start.S`

> **Скриншот:** содержимое start.S
> `<!-- вставить assets/step5_start_s.png -->`

Этот файл:
- ставит стек;
- настраивает `mtvec`;
- делегирует `ECALL from U-mode` в `S-mode` через `medeleg`;
- настраивает PMP (доступ к памяти для S-mode);
- выполняет `MRET` для перехода в S-mode.

```asm
.section .text.init
.global _start

_start:
    la sp, _stack_top

    la t0, m_trap_vector
    csrw mtvec, t0

    /* делегируем ECALL из U-mode (бит 8) в S-mode */
    li t0, (1 << 8)
    csrw medeleg, t0

    /* настройка PMP: разрешаем S-mode доступ ко всей памяти */
    li t0, -1
    csrw pmpaddr0, t0
    li t0, 0x0F
    csrw pmpcfg0, t0

    /* mstatus.MPP = S-mode (01) */
    csrr t0, mstatus
    li t1, ~(3 << 11)
    and t0, t0, t1
    li t1, (1 << 11)
    or t0, t0, t1
    csrw mstatus, t0

    la t0, s_mode_main
    csrw mepc, t0

    mret
```

**Что здесь происходит:**

`MRET` возвращает в тот privilege level, который записан в поле `MPP` регистра `mstatus`. Поэтому перед `MRET` мы выставляем `MPP = S`, а адрес будущего исполнения — в `mepc`.

**Что изучить после запуска:**
- зачем нужен `mtvec`;
- зачем нужен `medeleg`;
- почему без корректного `MPP` переход в `S-mode` не произойдёт.

---

# Часть 4. Trap-вектор и сохранение контекста

## Шаг 6. Создайте `trap.S`

> **Скриншот:** содержимое trap.S
> `<!-- вставить assets/step6_trap_s.png -->`

Два trap entry:
- `m_trap_vector` — на случай ошибок на машинном уровне;
- `s_trap_vector` — основной обработчик для `ECALL` из `U-mode`.

```asm
.section .text

.global m_trap_vector
.global s_trap_vector

m_trap_vector:
    addi sp, sp, -128
    sd ra,  0(sp)
    sd a0,  8(sp)
    sd a1, 16(sp)
    sd a2, 24(sp)
    sd a3, 32(sp)
    sd a4, 40(sp)
    sd a5, 48(sp)
    sd a6, 56(sp)
    sd a7, 64(sp)

    call handle_m_trap

    ld ra,  0(sp)
    ld a0,  8(sp)
    ld a1, 16(sp)
    ld a2, 24(sp)
    ld a3, 32(sp)
    ld a4, 40(sp)
    ld a5, 48(sp)
    ld a6, 56(sp)
    ld a7, 64(sp)
    addi sp, sp, 128
    mret

s_trap_vector:
    addi sp, sp, -128
    sd ra,  0(sp)
    sd a0,  8(sp)
    sd a1, 16(sp)
    sd a2, 24(sp)
    sd a3, 32(sp)
    sd a4, 40(sp)
    sd a5, 48(sp)
    sd a6, 56(sp)
    sd a7, 64(sp)

    mv a0, sp
    call handle_s_trap

    ld ra,  0(sp)
    ld a1, 16(sp)
    ld a2, 24(sp)
    ld a3, 32(sp)
    ld a4, 40(sp)
    ld a5, 48(sp)
    ld a6, 56(sp)
    ld a7, 64(sp)
    ld a0,  8(sp)   /* a0 последним — содержит return value из обработчика */
    addi sp, sp, 128
    sret
```

> **Почему это достаточно для первой работы:** цель — увидеть механизм privilege transition, а не построить полноценный ABI-совместимый kernel entry path.

---

# Часть 5. Код на C: ядро S-mode и пользовательский U-mode код

## Шаг 7. Создайте `main.c`

> **Скриншот:** содержимое main.c
> `<!-- вставить assets/step7_main_c.png -->`

```c
#include <stdint.h>
#include "uart.h"

extern void s_trap_vector(void);
extern void user_entry(void);

/* CSR helpers */
static inline uint64_t csr_read_scause(void)  { uint64_t x; asm volatile("csrr %0, scause"  : "=r"(x)); return x; }
static inline uint64_t csr_read_sepc(void)    { uint64_t x; asm volatile("csrr %0, sepc"    : "=r"(x)); return x; }
static inline uint64_t csr_read_sstatus(void) { uint64_t x; asm volatile("csrr %0, sstatus" : "=r"(x)); return x; }
static inline uint64_t csr_read_mcause(void)  { uint64_t x; asm volatile("csrr %0, mcause"  : "=r"(x)); return x; }
static inline uint64_t csr_read_mepc(void)    { uint64_t x; asm volatile("csrr %0, mepc"    : "=r"(x)); return x; }

static inline void csr_write_stvec(uint64_t x)   { asm volatile("csrw stvec,   %0" :: "r"(x)); }
static inline void csr_write_sepc(uint64_t x)    { asm volatile("csrw sepc,    %0" :: "r"(x)); }
static inline void csr_write_sstatus(uint64_t x) { asm volatile("csrw sstatus, %0" :: "r"(x)); }

/* Trap frame — отражает сохранённые регистры в trap.S */
struct trap_frame {
    uint64_t ra;
    uint64_t a0, a1, a2, a3, a4, a5, a6, a7;
};

/* M-mode trap: только для непредвиденных ошибок */
void handle_m_trap(void) {
    uart_puts("\n[M-TRAP] unexpected machine trap\n");
    uart_puts("  mcause = "); uart_puthex(csr_read_mcause()); uart_puts("\n");
    uart_puts("  mepc   = "); uart_puthex(csr_read_mepc());   uart_puts("\n");
    while (1);
}

/* S-mode trap: обработка ECALL из U-mode */
void handle_s_trap(struct trap_frame *tf) {
    uint64_t scause  = csr_read_scause();
    uint64_t sepc    = csr_read_sepc();
    uint64_t sstatus = csr_read_sstatus();

    uart_puts("\n[S-TRAP] supervisor trap received\n");
    uart_puts("  scause  = "); uart_puthex(scause);  uart_puts("\n");
    uart_puts("  sepc    = "); uart_puthex(sepc);    uart_puts("\n");
    uart_puts("  sstatus = "); uart_puthex(sstatus); uart_puts("\n");

    /* пропускаем инструкцию ecall */
    csr_write_sepc(sepc + 4);

    /* диспетчеризация по номеру системного вызова (a7) */
    switch (tf->a7) {
        case 1:  tf->a0 = tf->a0 + tf->a1;               break; /* сложение */
        case 2:  tf->a0 = tf->a0 ^ tf->a1 ^ tf->a2;      break; /* XOR */
        case 42: tf->a0 = 0x1234;                         break; /* константа */
        default: tf->a0 = 0;                              break;
    }
}

/* Пользовательский код в U-mode */
void user_entry(void) {
    uart_puts("[U] user_entry started\n");

    register uint64_t a0 asm("a0") = 0x1111;
    register uint64_t a1 asm("a1") = 0x2222;
    register uint64_t a2 asm("a2") = 0x3333;
    register uint64_t a7 asm("a7") = 42;

    asm volatile(
        "ecall"
        : "+r"(a0)
        : "r"(a1), "r"(a2), "r"(a7)
        : "memory"
    );

    uint64_t result = a0;
    uart_puts("[U] returned from ecall, a0 = ");
    uart_puthex(result);
    uart_puts("\n");

    while (1);
}

/* Точка входа в S-mode */
void s_mode_main(void) {
    uart_init();
    uart_puts("\n=== Lab1: RISC-V privilege demo ===\n");
    uart_puts("[S] entered supervisor mode\n");

    csr_write_stvec((uint64_t)s_trap_vector);

    /* sstatus.SPP = 0 → возврат через sret будет в U-mode */
    uint64_t sstatus = csr_read_sstatus();
    sstatus &= ~(1UL << 8);
    csr_write_sstatus(sstatus);

    csr_write_sepc((uint64_t)user_entry);

    uart_puts("[S] switching to user mode...\n");
    asm volatile("sret");

    while (1);
}
```

---

## Что важно понять в этом коде

**1. Почему trap от `ECALL` попадает в `S-mode`**

Потому что в `M-mode` был настроен `medeleg`, и исключение "environment call from U-mode" делегировано в supervisor-обработчик.

**2. Почему нужно увеличивать `sepc` на 4**

`sepc` содержит адрес инструкции `ECALL`. Без `sepc += 4` после `SRET` та же инструкция будет выполнена снова → бесконечный trap-loop.

**3. Почему в первом варианте пользовательский код обращается к UART напрямую**

Для низкого порога входа: цель — показать trap-path, а не изоляцию устройства. В доп. задании 1 реализуется вывод только через `ECALL`.

---

# Часть 6. Сборка проекта

## Шаг 8. Создайте `Makefile`

> **Скриншот:** содержимое Makefile
> `<!-- вставить assets/step8_makefile.png -->`

```makefile
CROSS   = riscv64-unknown-elf-
CC      = $(CROSS)gcc
OBJDUMP = $(CROSS)objdump

CFLAGS  = -march=rv64imac_zicsr -mabi=lp64 -Wall -Wextra \
          -O0 -g -mcmodel=medany -ffreestanding -nostdlib -nostartfiles

OBJS    = start.o trap.o uart.o main.o

all: lab1.elf lab1.dis

lab1.elf: $(OBJS)
	$(CC) $(CFLAGS) $(OBJS) -T linker.ld -nostdlib -o $@

start.o: start.S
	$(CC) $(CFLAGS) -c $< -o $@

trap.o: trap.S
	$(CC) $(CFLAGS) -c $< -o $@

uart.o: uart.c uart.h
	$(CC) $(CFLAGS) -c $< -o $@

main.o: main.c uart.h
	$(CC) $(CFLAGS) -c $< -o $@

lab1.dis: lab1.elf
	$(OBJDUMP) -d lab1.elf > lab1.dis

run: lab1.elf
	qemu-system-riscv64 -M virt -m 512M -nographic -bios none -kernel lab1.elf

trace: lab1.elf
	qemu-system-riscv64 -M virt -m 512M -nographic -bios none -kernel lab1.elf \
	    -d in_asm,exec,cpu,guest_errors -D qemu.log

clean:
	rm -f *.o *.elf *.dis qemu.log
```

> **Почему `-O0 -g`:** для первой лабораторной так легче читать дизассемблированный код и отлаживать переходы.

---

# Часть 7. Первый запуск

## Шаг 9. Соберите проект

```bash
make clean
make
```

Ожидаемые артефакты: `lab1.elf`, `lab1.dis`

> **Скриншот:** успешная сборка
> `<!-- вставить assets/step9_build.png -->`

## Шаг 10. Запустите проект

```bash
make run
```

Ожидаемый вывод:

```text
=== Lab1: RISC-V privilege demo ===
[S] entered supervisor mode
[S] switching to user mode...
[U] user_entry started

[S-TRAP] supervisor trap received
  scause  = 0x0000000000000008
  sepc    = 0x0000000080000xxx
  sstatus = 0x0000000000000xxx
[U] returned from ecall, a0 = 0x0000000000001234
```

> **Скриншот:** вывод программы в QEMU
> `<!-- вставить assets/step10_run.png -->`

Ключевой момент: **`scause = 8`** соответствует `environment call from U-mode` в привилегированной архитектуре RISC-V.

---

# Часть 8. Трассировка и изучение результата

## Шаг 11. Запуск с трассировкой

```bash
make trace
```

> **Скриншот:** запуск make trace
> `<!-- вставить assets/step11_trace.png -->`

После этого появится файл `qemu.log`. Что искать:

- адреса инструкций перед `MRET`;
- адреса перед `SRET`;
- выполнение `ECALL`;
- изменение `pc` после trap.

```bash
grep -n "ecall\|mret\|sret" qemu.log | head -n 20
```

> **Скриншот:** содержимое qemu.log, найденные строки
> `<!-- вставить assets/step11_qemu_log.png -->`

---

# Часть 9. Задания

## Задание 1. Базовые переходы

Соберите и запустите базовый вариант программы, добившись корректного перехода:
- `M-mode` → `S-mode`;
- `S-mode` → `U-mode`;
- `U-mode` → `S-mode` по `ECALL`;
- `S-mode` → `U-mode` по `SRET`.

> **Скриншот:** вывод программы с переходами
> `<!-- вставить assets/task1_result.png -->`

**Результаты:**

Переход `M-mode → S-mode` происходит в `_start`. Устанавливается поле `MPP = 01` (Supervisor mode) в регистре `mstatus`, в `mepc` записывается адрес `s_mode_main`, затем инструкция `mret`:
- переключает процессор из M-mode в режим MPP;
- загружает `pc ← mepc`.

> **Скриншот:** фрагмент start.S с MPP и mret
> `<!-- вставить assets/task1_mmode_to_smode.png -->`

Переход `S-mode → U-mode` происходит в `s_mode_main`. Очищается бит `SPP` в `sstatus` (SPP = 0 → sret перейдёт в U-mode), в `sepc` записывается адрес `user_entry`, выполняется `sret`.

> **Скриншот:** фрагмент s_mode_main с SPP и sret
> `<!-- вставить assets/task1_smode_to_umode.png -->`

Переход `U-mode → S-mode` происходит через `ecall` в `user_entry`. Делегирование настроено в `_start` (`medeleg = 1 << 8`, бит 8 = ECALL from U-mode), поэтому управление переходит в `s_trap_vector → handle_s_trap`.

> **Скриншот:** фрагмент user_entry с ecall
> `<!-- вставить assets/task1_umode_ecall.png -->`

Переход `S-mode → U-mode` по `SRET` происходит в конце `s_trap_vector`. Обработчик корректирует `sepc += 4`, при входе в trap из U-mode аппаратно `SPP = 0`, поэтому `sret` возвращает выполнение в U-mode.

> **Скриншот:** конец trap.S с sret
> `<!-- вставить assets/task1_sret.png -->`

---

## Задание 2. Анализ CSR

Зафиксируйте в отчёте значения `scause`, `sepc`, `sstatus` и объясните, почему trap произошёл именно в этом месте и почему после возврата продолжилось исполнение со следующей инструкции.

> **Скриншот:** вывод trap с CSR значениями
> `<!-- вставить assets/task2_csr.png -->`

**Результаты:**

```
scause  = 0x0000000000000008
sepc    = 0x000000008000044e
sstatus = 0x0000000200000000
```

Trap произошёл на инструкции `8000044e: ecall` потому что:
- инструкция `ecall` предназначена для программного вызова исключения;
- при выполнении в U-mode она вызывает synchronous exception с кодом 8;
- в `_start` настроена делегация: `medeleg = 256 = 1 << 8` → исключение №8 (`ECALL from U-mode`) делегируется в S-mode.

> **Скриншот:** делегирование в start.S
> `<!-- вставить assets/task2_medeleg.png -->`

Почему продолжение со следующей инструкции: при trap аппаратно `sepc = адрес ecall`. Без `sepc += 4` после `sret` процессор снова попадёт на `ecall` → бесконечный цикл. Обработчик увеличивает `sepc` на 4 (размер инструкции `ecall`).

> **Скриншот:** sepc += 4 в handle_s_trap
> `<!-- вставить assets/task2_sepc.png -->`

---

## Задание 3. Возвращаемое значение зависит от аргументов

Измените обработчик `handle_s_trap()` так, чтобы `a0 = a0 + a1 + a2`.

**Результаты:**

В `trap.S` передаём указатель на trap frame через `a0`:

```asm
mv a0, sp
call handle_s_trap
```

В `main.c` используем структуру trap frame:

```c
struct trap_frame { uint64_t ra, a0, a1, a2, a3, a4, a5, a6, a7; };

void handle_s_trap(struct trap_frame *tf) {
    csr_write_sepc(csr_read_sepc() + 4);
    tf->a0 = tf->a0 + tf->a1 + tf->a2;  /* 0x1111 + 0x2222 + 0x3333 = 0x6666 */
}
```

> **Скриншот:** результат — a0 = 0x6666
> `<!-- вставить assets/task3_result.png -->`

---

## Задание 4. Несколько системных вызовов через `a7`

Реализуйте диспетчеризацию:

| `a7` | Результат |
|------|-----------|
| `1` | `a0 + a1` |
| `2` | `a0 ^ a1 ^ a2` |
| `42` | `0x1234` |

**Результаты:**

```c
switch (tf->a7) {
    case 1:  tf->a0 = tf->a0 + tf->a1;          break;
    case 2:  tf->a0 = tf->a0 ^ tf->a1 ^ tf->a2; break;
    case 42: tf->a0 = 0x1234;                    break;
    default: tf->a0 = 0;                         break;
}
```

> **Скриншоты:** проверка каждого syscall
> `<!-- вставить assets/task4_syscall1.png -->`
> `<!-- вставить assets/task4_syscall2.png -->`
> `<!-- вставить assets/task4_syscall42.png -->`

Это первый шаг от ISA-level trap к простейшему ABI системных вызовов.

---

## Задание 5. Трассировка QEMU

Включите трассировку и покажите где происходят `mret`, `sret`, `ecall`.

```bash
make trace
grep -n "ecall\|mret\|sret" qemu.log | head -n 20
```

**Результаты:**

| Инструкция | Адрес | Режим | Переход |
|-----------|-------|-------|---------|
| `mret` | `0x80000050` | M-mode (Priv=3) | M → S |
| `sret` (первый) | `0x8000054a` | S-mode (Priv=1) | S → U |
| `ecall` | `0x800004aa` | U-mode (Priv=0) | U → S trap |
| `sret` (из trap) | `0x800000b2` | S-mode (Priv=1) | S → U (возврат) |

Полный цикл: **M → S → U → S → U**

> **Скриншоты:** mret, sret, ecall в qemu.log
> `<!-- вставить assets/task5_mret.png -->`
> `<!-- вставить assets/task5_sret.png -->`
> `<!-- вставить assets/task5_ecall.png -->`

---

# Дополнительные задания

## Доп. 1. Вывод из U-mode только через ECALL

Уберите прямой UART из U-mode. Весь вывод — через `sys_write_char`.

**Результаты:**

Определяем номер syscall:

```c
#define SYS_WRITE_CHAR 10
```

В `handle_s_trap`:

```c
case SYS_WRITE_CHAR:
    uart_putc((char)tf->a0);
    break;
```

Обёртка в U-mode:

```c
static void sys_write_char(char c) {
    register uint64_t a0 asm("a0") = (uint64_t)c;
    register uint64_t a7 asm("a7") = SYS_WRITE_CHAR;
    asm volatile("ecall" : "+r"(a0) : "r"(a7) : "memory");
}

static void sys_write_string(const char *s) {
    while (*s) sys_write_char(*s++);
}
```

> **Скриншот:** вывод через syscall (каждый символ — отдельный trap с scause=8)
> `<!-- вставить assets/extra1_result.png -->`

---

## Доп. 2. Убрать `medeleg`

Уберите или обнулите `medeleg` в `start.S` и объясните что изменится.

```asm
/* было: li t0, (1 << 8) / csrw medeleg, t0 */
/* стало: ничего */
```

**Результат:** trap от `ECALL` больше не делегируется в S-mode, а попадает в `m_trap_vector`. Выводится `[M-TRAP] unexpected machine trap` с `mcause = 0x8`.

> **Скриншот:** вывод без medeleg
> `<!-- вставить assets/extra2_result.png -->`

**Вывод:** `medeleg` — это механизм делегирования. Без него все исключения обрабатываются в M-mode, что не соответствует модели «ядро в S-mode».

---

## Доп. 3. Убрать `sepc += 4`

Закомментируйте `csr_write_sepc(sepc + 4)` и объясните результат.

**Результат:** после `sret` процессор возвращается на ту же инструкцию `ecall`. Она снова генерирует trap. Обработчик снова вызывается, снова делает `sret` без сдвига `sepc`. Получается бесконечный цикл: `U-mode ecall → S-mode trap → sret → U-mode ecall → ...`

> **Скриншот:** бесконечный поток trap-событий
> `<!-- вставить assets/extra3_result.png -->`

---

# Часть 10. Что делать, если не работает

## Нет никакого вывода

Проверьте:
- адрес UART `0x10000000` для `virt`;
- QEMU запущен с `-nographic`;
- `_start` находится в `0x80000000`.

```bash
riscv64-unknown-elf-objdump -h lab1.elf
riscv64-unknown-elf-objdump -d lab1.elf | head -n 80
```

## Сразу попадаем в M-trap

Проверьте `mstatus.MPP`, `mepc`, `mtvec`, выравнивание адресов trap vector.

## После `ECALL` система зависает

Проверьте:
- делегирован ли cause code через `medeleg`;
- корректно ли установлен `stvec`;
- увеличивается ли `sepc` на 4 перед возвратом.

## Возврат в U-mode не происходит

Проверьте:
- очищен ли `sstatus.SPP` перед `SRET`;
- не повреждён ли стек trap-обработчиком.

---

# Часть 11. ISA, ABI и API

После выполнения лабораторной студент должен разложить увиденное на три слоя:

### ISA

К ISA относятся:
- режимы `M`, `S`, `U`;
- инструкции `ECALL`, `MRET`, `SRET`;
- CSR: `mstatus`, `sstatus`, `mepc`, `sepc`, `mtvec`, `stvec`, `medeleg`.

Это «аппаратный язык» привилегий.

### ABI

К ABI в этой работе относится договорённость:
- пользовательский код кладёт номер вызова в `a7`;
- аргументы — в `a0-a2`;
- trap handler интерпретирует регистры и возвращает результат в `a0`.

Это не стандартный Linux ABI, но уже **мини-ABI лабораторной ОС**.

### API

Если поверх `ECALL` написать функцию:

```c
long sys_add(long x, long y) {
    register long a0 asm("a0") = x;
    register long a1 asm("a1") = y;
    register long a7 asm("a7") = 1;
    asm volatile("ecall" : "+r"(a0) : "r"(a1), "r"(a7) : "memory");
    return a0;
}
```

то это уже простейший **API-слой** поверх ABI и ISA.

---

# Часть 12. Требования к отчёту

Отчёт должен содержать:

1. **Цель работы** — изучение уровней привилегий RISC-V и механизма системного вызова через `ECALL`.
2. **Описание платформы** — `qemu-system-riscv64`, машина `virt`, toolchain `riscv64-unknown-elf-gcc`.
3. **Структура проекта** — назначение файлов `start.S`, `trap.S`, `main.c`, `uart.c`, `linker.ld`.
4. **Основные фрагменты кода** — переход M→S, переход S→U, код `ECALL`, обработчик trap.
5. **Вывод программы** — скриншот или текст из терминала.
6. **Анализ** — ответы на вопросы:
   - почему старт происходит в M-mode;
   - зачем нужен `medeleg`;
   - почему `sepc` нужно увеличивать на 4;
   - что относится к ISA и что — к ABI.
7. **Вывод** — пользовательский код не может напрямую выполнять привилегированные действия и использует trap-механизм как архитектурную основу системных вызовов.

---

# Часть 13. Чек-лист преподавателя

Работу можно считать выполненной, если студент:

- [ ] собрал проект без готового IDE-шаблона;
- [ ] показал корректный переход M→S→U;
- [ ] показал `ECALL` из U-mode;
- [ ] показал trap в S-mode;
- [ ] объяснил `scause` и `sepc`;
- [ ] объяснил роль `medeleg` и `SRET`.

**Повышенный уровень:**

- [ ] реализовал несколько syscall numbers через `a7`;
- [ ] убрал прямой UART из U-mode;
- [ ] показал разницу между обработкой trap в M-mode и S-mode.

---

# Часть 14. Что делать дальше

Эта лабораторная — фундамент для следующей работы с seL4 на RISC-V:

- в **Лабе 1** студент вручную строит путь `U → trap → S → return`;
- в **Лабе 2** он смотрит, как этот путь организован в реальном микроядре seL4 на `qemu-riscv-virt`.

---

## Примечание по скриншотам

> Места для скриншотов обозначены комментариями вида:
> ```
> <!-- вставить assets/имя_файла.png -->
> ```
> Сохраните скриншоты в папку `assets/` рядом с этим файлом и замените комментарии на:
> ```markdown
> ![описание](assets/имя_файла.png)
> ```
