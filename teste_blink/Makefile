all : flash

TARGET:=teste
TARGET_MCU?=CH32V002
CH32FUN:=../ch32fun/ch32fun
MINICHLINK:=../ch32fun/minichlink
PORT?=/dev/ttyUSB0

include $(CH32FUN)/ch32fun.mk

flash : $(TARGET).bin
	$(MINICHLINK)/minichlink -c $(PORT) -C ardulink -w $(TARGET).bin flash -b

# Lê o bloco de status na RAM do chip (prova que o código está rodando)
check : $(TARGET).elf
	@ADDR=0x$$(riscv64-elf-nm $(TARGET).elf | awk '/ status$$/{print $$1}'); \
	echo "status @ $$ADDR"; \
	$(MINICHLINK)/minichlink -c $(PORT) -C ardulink -r + $$ADDR 16 -e -i -r + $$ADDR 16 -e -i -r + $$ADDR 16 -e -m 0x11 2>&1 | grep -E "^2000|REGISTER"; \
	echo "campos: magic(C0FFEE02) counter pins spins -- counter deve subir a cada leitura | DMSTATUS ...0c82 = rodando"

clean : cv_clean
