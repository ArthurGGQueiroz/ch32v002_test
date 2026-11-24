# Como programar o CH32V002 (C + ch32fun + Ardulink)

Passo a passo para escrever um programa em C, compilar e gravar no CH32V002
usando um Arduino como gravador (Ardulink).

Estrutura desta pasta:

```
ch32v003/
├── ch32fun/          # SDK (ch32fun) + minichlink (gravador)
├── ardulink/         # firmware do Arduino que vira gravador
├── teste_blink/      # exemplo: pisca pinos + status na RAM
└── teste_gpio/       # exemplo: acende um pino por vez (teste de GPIO)
```

---

## 1. Instalar as ferramentas (uma vez só)

```bash
sudo pacman -S riscv64-elf-gcc make libusb avr-gcc avr-libc avrdude
```

- `riscv64-elf-gcc`: compilador para o CH32V002 (RISC-V)
- `avr-gcc` / `avrdude`: só para gravar o firmware Ardulink no Arduino

Se der erro de permissão na porta serial:

```bash
sudo usermod -aG uucp $USER    # depois faça logout/login
```

## 2. Preparar o gravador (uma vez só)

### 2.1 Gravar o Ardulink no Arduino

```bash
cd ardulink
make
avrdude -carduino -P /dev/ttyUSB0 -p m328p -Uflash:w:main.elf:e
```

- Nano com CH340: porta `/dev/ttyUSB0` (com o bootloader antigo, acrescente `-b 57600`)
- Uno: porta `/dev/ttyACM0`

### 2.2 Ligar o Arduino no CH32V002

| Arduino | Ligação              | CH32V002          |
|---------|----------------------|-------------------|
| D8      | resistor de 1 kΩ     | SWIO (PD1)        |
| D9      | alimenta o chip      | VDD               |
| GND     |                      | GND               |

Se a placa do CH32V002 tiver LEDs ou outros componentes, ligue o VDD no **5V**
do Arduino em vez do D9.

### 2.3 Compilar o minichlink

```bash
cd ch32fun/minichlink
make
```

> O minichlink desta pasta tem correções locais para o Ardulink:
> o `-e` (resume) agora funciona e preserva os registradores da CPU.
> Se clonar o ch32fun de novo, essas correções se perdem.

### 2.4 Testar a conexão

```bash
./ch32fun/minichlink/minichlink -c /dev/ttyUSB0 -C ardulink -i
```

Tem que aparecer `Detected CH32V002`. Cada conexão leva ~3 s (o Arduino reinicia).

---

## 3. Criar um projeto novo

O jeito mais fácil é copiar um exemplo:

```bash
cp -r teste_blink meu_projeto
cd meu_projeto
rm -f *.elf *.bin *.hex *.lst *.map
```

Um projeto precisa de 3 arquivos:

### `Makefile`

```make
all : flash

# nome do arquivo .c, sem extensão (main.c)
TARGET:=main
TARGET_MCU?=CH32V002
CH32FUN:=../ch32fun/ch32fun
MINICHLINK:=../ch32fun/minichlink
PORT?=/dev/ttyUSB0

include $(CH32FUN)/ch32fun.mk

flash : $(TARGET).bin
	$(MINICHLINK)/minichlink -c $(PORT) -C ardulink -w $(TARGET).bin flash -b

clean : cv_clean
```

(Atenção: no Makefile, a linha do comando começa com **TAB**, não espaços.)

### `funconfig.h`

```c
#ifndef _FUNCONFIG_H
#define _FUNCONFIG_H
// configurações opcionais do ch32fun (ver ch32fun/ch32fun/ch32fun.h)
#endif
```

### `main.c`

```c
#include "ch32fun.h"

#define LED PC0

int main()
{
	SystemInit();                 // clock a 48 MHz, SysTick etc.
	funGpioInitAll();             // liga o clock de todos os GPIOs

	funPinMode(LED, GPIO_Speed_10MHz | GPIO_CNF_OUT_PP);   // saída

	while (1)
	{
		funDigitalWrite(LED, FUN_HIGH);
		Delay_Ms(500);
		funDigitalWrite(LED, FUN_LOW);
		Delay_Ms(500);
	}
}
```

Funções mais usadas:

| Função | O que faz |
|---|---|
| `funPinMode(PC0, GPIO_Speed_10MHz \| GPIO_CNF_OUT_PP)` | pino como saída |
| `funPinMode(PC0, GPIO_CNF_IN_PUPD)` + `funDigitalWrite(PC0, FUN_HIGH)` | entrada com pull-up |
| `funDigitalWrite(PC0, FUN_HIGH / FUN_LOW)` | escreve no pino |
| `funDigitalRead(PC0)` | lê o pino |
| `Delay_Ms(n)` / `Delay_Us(n)` | espera |

Pinos: `PA1`, `PA2`, `PC0`…`PC7`, `PD0`…`PD7` (depende do encapsulamento).
**Não use PD1** (é o SWIO, usado para gravar) nem PD7 (NRST).

Mais exemplos prontos: `ch32fun/examples/` (ADC, PWM, UART, I2C…). Para usá-los,
copie a pasta e troque o `TARGET_MCU` para `CH32V002`.

---

## 4. Compilar

```bash
make main.bin
```

Gera `main.bin` (o que vai pro chip), `main.elf`, `main.lst` (assembly) e
`main.map`. No fim aparece o uso de memória (FLASH de 16 KB, RAM de 4 KB).

## 5. Gravar

```bash
make            # ou: make flash
```

Compila (se precisar) e grava. Termina com `Image written.` e o chip já
reinicia rodando o programa novo.

Outra porta: `make flash PORT=/dev/ttyACM0`

---

## 6. Ver se está rodando (sem LED)

Guarde o estado do programa numa variável global e leia pela SWIO:

```c
volatile uint32_t contador;   // incremente no loop
```

```bash
# endereço da variável
riscv64-elf-nm main.elf | grep contador

# ler 4 bytes desse endereço (ex.: 0x20000000); -e deixa o chip rodando depois
../ch32fun/minichlink/minichlink -c /dev/ttyUSB0 -C ardulink -r + 0x20000000 4 -e
```

Os exemplos `teste_blink` e `teste_gpio` já fazem isso com `make check`.

Comandos úteis do minichlink (sempre com `-c /dev/ttyUSB0 -C ardulink`):

| Opção | O que faz |
|---|---|
| `-i` | mostra informações do chip |
| `-w arq.bin flash` | grava |
| `-b` | reinicia o chip |
| `-e` | continua a execução (depois de ler) |
| `-r + 0xENDEREÇO N` | lê N bytes da memória (RAM ou registradores) |
| `-E` | apaga o chip inteiro |
| `-u` | destrava o chip (se gravou algo que travou o SWIO) |

---

## 7. Problemas comuns

| Sintoma | Solução |
|---|---|
| `Could not initialize any supported programmers` | faltou `-c /dev/ttyUSB0 -C ardulink`, ou porta errada (`ls /dev/ttyUSB* /dev/ttyACM*`) |
| `Permission denied` na porta | `sudo usermod -aG uucp $USER` e relogar |
| `Setup chip failed` | confira os fios D8/SWIO, o resistor de 1 kΩ e o GND |
| Grava mas não roda | confira se o `TARGET_MCU` é `CH32V002` no Makefile |
| Chip não responde mais após gravar | o programa usou o PD1; rode `minichlink ... -u` |
| `make` reclama de "missing separator" | o Makefile tem espaços no lugar de TAB |
