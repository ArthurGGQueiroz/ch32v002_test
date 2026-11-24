all : flash

TARGET:=gpio
TARGET_MCU?=CH32V002
CH32FUN:=../ch32fun/ch32fun
MINICHLINK:=../ch32fun/minichlink
PORT?=/dev/ttyUSB0
# o Ardulink não tem delay real; ~1500 leituras de DMSTATUS dão uns 2 s de espera
ESPERA:=-e $(foreach i,$(shell seq 1500),-m 0x11)

include $(CH32FUN)/ch32fun.mk

flash : $(TARGET).bin
	$(MINICHLINK)/minichlink -c $(PORT) -C ardulink -w $(TARGET).bin flash -b

# Lê o bloco de status na RAM do chip (prova que o código está rodando)
check : $(TARGET).elf
	@ADDR=0x$$(riscv64-elf-nm $(TARGET).elf | awk '/ status$$/{print $$1}'); \
	echo "status @ $$ADDR"; \
	$(MINICHLINK)/minichlink -c $(PORT) -C ardulink -r + $$ADDR 12 $(ESPERA) -r + $$ADDR 12 $(ESPERA) -r + $$ADDR 12 $(ESPERA) -r + $$ADDR 12 -e 2>&1 | grep -E "^2000"; \
	echo "campos: magic(C0FFEE03) volta pino -- pino (byte 8): A1=PA1, C3=PC3, D0=PD0, FF=todos (marcador)"

clean : cv_clean
