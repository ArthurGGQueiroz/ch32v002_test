# Montar o Ardulink

Repositório: https://gitlab.com/BlueSyncLine/arduino-ch32v003-swio

### 1. Hardware

O firmware tem os tempos do protocolo contados em ciclos de clock, então exige um **ATmega328P a 16 MHz** (Arduino Uno R3 ou Nano).

| Arduino | Ligação | CH32V002 |
|---|---|---|
| **D8** (PB0) | resistor de **1 kΩ** em série | **SWIO** (PD1 na maioria dos encapsulamentos) |
| **D9** (PB1) | alimenta o alvo (opcional) | VDD |
| GND | | GND |

- **O resistor de 1 kΩ é necessário na prática**: para ler os dados, o firmware cria de propósito um conflito no barramento, e o resistor limita a corrente.
- **Alimentação:** o D9 alimenta só o chip, sem periféricos. Se a placa tiver LEDs ou outros periféricos, ligue o VDD no **5V** do Arduino. Não use 3,3 V, porque o SWIO trabalha em 5 V.
- **Confira o pino SWIO no datasheet** do seu encapsulamento (SOP8, TSSOP20, QFN etc.).

### 2. Gravar o firmware no Arduino

```bash
sudo pacman -S avr-gcc avr-libc avrdude
git clone https://gitlab.com/BlueSyncLine/arduino-ch32v003-swio.git ardulink
cd ardulink
make
avrdude -carduino -P /dev/ttyACM0 -p m328p -Uflash:w:main.elf:e
```

- **Uno:** porta costuma ser `/dev/ttyACM0`.
- **Nano clone com CH340:** porta costuma ser `/dev/ttyUSB0`. Com o bootloader antigo, acrescente `-b 57600` no avrdude.
- Erro de permissão na porta: `sudo usermod -aG uucp $USER`, depois logout e login.

### 3. Compilar o minichlink

```bash
sudo pacman -S libusb
git clone https://github.com/cnlohr/ch32fun.git
cd ch32fun/minichlink
make
```

### 4. Usar

```bash
# testar a conexão e identificar o chip
./minichlink -c /dev/ttyACM0 -C ardulink -i

# gravar um binário
./minichlink -c /dev/ttyACM0 -C ardulink -w firmware.bin flash -b
```

A porta também pode vir da variável de ambiente `MINICHLINK_SERIAL`.

### Limitações

- **CH32V002 não testado com o Ardulink.** O minichlink reconhece o CH32V002 (detecção da família V00x), mas o Ardulink foi feito e testado para o CH32V003.
- **É lento**: cada acesso a registrador é uma ida e volta pela serial a 115200 baud, e toda conexão tem uma espera fixa de 3 segundos.
- **Só gravação, sem debug**: implementa apenas as operações básicas de registrador; não conte com GDB nem com o printf por SWIO do ch32fun.
