;**************************
; Universidad del Valle de Guatemala
; Programación de Microcrontroladores
; Proyecto: Interrupciones
; Archivo: Lab3.asm
; Hardware: ATMEGA328p
; Created: 30/01/2024 18:11:46	
; Author : James Ramírez
;******************************************************************************
; Encabezado: 
;******************************************************************************
.include "M328PDEF.inc"
.cseg
.org 0x0000
	JMP MAIN				;Salta al código principal
.org 0x0006
	JMP ISR_PCINT0			;Salta a las interrupciones por pines
.org 0x0020
	JMP ISR_TIMER0_OVF		;Salta a la interrución por overflow del timer 0

MAIN:
//***************************************************************
// STACK POINTER
//***************************************************************
LDI R16, LOW(RAMEND)
OUT SPL, R16
LDI R17, HIGH(RAMEND)
OUT SPH, R17

//***************************************************************
// PROGRAMA PRINCIPAL I/O
//***************************************************************
SETUP:
	SEI	;Habilita las interrupciones globales

	LDI R16, 0b1000_0000     ;El timer se establece a 8MHz
	LDI R16, (1 << CLKPCE)
	STS CLKPR, R16

	LDI R16, 0b0000_0001
	STS CLKPR, R16

	LDI R16, 0b1111_1111 ;METEMOS ESE VALOR EN BINARIO EN EL REGISTRO R16
	OUT DDRD, R16 ;DEFINIMOS CÓMO SALIDA LOS PUERTOS D

	;Se configura el puerto C como salidas (LEDS)
	LDI R16, 0xFF
	OUT DDRC, R16

	;Se configura como pull ups PB0 y PB1
	LDI R16, 0b0000_0011 
	OUT	PORTB, R16
	LDI R16, 0b0000_1100
	OUT DDRB, R16

	;Habilitando los PCINT
	LDI R16, (1<<PCINT0) | (1<<PCINT1)
	STS PCMSK0, R16

	;Habilitando las ISR de los PCINT
	LDI R16, (1<<PCIE0)
	STS PCICR, R16

	LDI R20, 0 ;Contador

	;Registros utilizados para los display
	LDI R25, 0
	LDI R26, 0
	LDI R27, 0
	LDI R28, 0
	
	;Inicio del timer0 e iniciamos la multiplexación
	CALL INIT_T0
	SBI PINB, 2


//***************************************************************
//  CONFIGURACIÓN:
//***************************************************************

LOOP:

	CPI R21, 6
	BREQ RESET

	CPI R22, 10
	BREQ DECENAS		;Establecemos una comparación para aumentar el valor del segundo display cuando el primero llega a 10
	CPI R23, 50
	BREQ UNIDADES		;Se realiza una comparación para que el valor mostrado por los display se reinicie

	CALL RETARDO
	SBI PINB, 2			;Multiplexación
	SBI PINB, 3

	LDI ZH, HIGH(TABLA<<1)
	LDI ZL, LOW(TABLA<<1)
	ADD	ZL, R21			;Se realiza el corrimiento en la posición de la tabla del display
	LPM	R24, Z
	OUT	PORTD, R24		;Se muestra el número correspondiente a la posición en la tabla
	CALL RETARDO

	SBI PINB, 2			;Multiplexación
	SBI PINB, 3

	LDI ZH, HIGH(TABLA<<1)
	LDI ZL, LOW(TABLA<<1)
	ADD	ZL, R22
	LPM	R24, Z
	OUT	PORTD, R24
	CALL RETARDO

	OUT PORTC, R20
	RJMP LOOP

	RETORNO:              //se realizan varios retardos para alcanzar el tiempo de los 1000ms 
	LDI R19, 255
	RETORNO1:
	DEC R19
	BRNE RETORNO1 
	LDI R19, 255
	RETORNO2:
	DEC R19
	BRNE RETORNO2
	LDI R19, 255
	RETORNO3:
	DEC R19
	BRNE RETORNO3
	LDI R19, 255
	RETORNO4:
	DEC R19
	BRNE RETORNO4
	RET

	DECENAS:
	LDI R22, 0
	INC R21
	RJMP LOOP

	UNIDADES:
	INC R22
	LDI R21, 0
	RJMP LOOP

	RESET:
	CALL RETARDO
	LDI R21, 0
	LDI R22,0
	RJMP LOOP


//***************************************************************
// SUBRUTINAS DE INTERRUPCIONES
//***************************************************************
ISR_PCINT0:
	PUSH R16
	IN R16, SREG
	PUSH R16

	IN R18, PINB	;Leemos el puerto b

ANTIRREBOTE:
	LDI R17, 215

	DELAY:
		DEC R17
		BRNE DELAY

	SBIS PINB, PB0 ;Se lee de nuevo el estado de boton
	RJMP ANTIRREBOTE

	SBRC R18, PB0	;verificamos si el botn esta presionado
	RJMP PUSH1


	INC R20		;incrementar el contador
	CPI R20, 16		;compara si el contador llego a 16
	BRNE EXIT
	LDI R20, 15		;cuando el contador llego a 16 lo reinicia
	RJMP EXIT

ANTIRREBOTE2:
	LDI R17, 215

	DELAY2:
		DEC R17
		BRNE DELAY2

	SBIS PINB, PB1
	RJMP ANTIRREBOTE2

PUSH1:
	SBRC R18, PB1	;Verifica el botn 1
	RJMP EXIT

	DEC R20
	CPI R20, -1
	BRNE EXIT
	LDI R20, 0

EXIT:
	OUT PORTC, R20
	SBI PCIFR, PCIF0

	POP R16
	OUT SREG, R16
	POP R16
	RETI

//***********************************************
//DISPLAY
//***********************************************
TABLA: .DB 0x7F, 0x0E, 0xB7, 0x9F, 0xCE, 0xDB, 0xFB, 0x0F, 0xFF, 0xCF, 0xEF, 0xFA, 0x73, 0xBE, 0xF3, 0xE3

//***********************************************
// SUBRUTINA PARA INICIAR EL TIMER 0
//***********************************************
INIT_T0:
	LDI R26, 0
	OUT TCCR0A, R26					;Configuración del modo de operación 

	LDI R16, (1<<CS02) | (1<<CS00)	;Configuración del preescaler
	OUT TCCR0B, R16

	LDI R16, 100					;Valor de inicio del timer0
	OUT TCNT0, R16

	LDI R16, (1<<TOIE0)				;Se activa la interrupción por overflow
	STS TIMSK0, R16
	RET


//***********************************************
// SUBRUTINA DE ISR TIMER 0
//***********************************************
ISR_TIMER0_OVF:
	PUSH R16
	IN R16, SREG
	PUSH R16

	LDI R16, 100
	OUT TCNT0, R16
	SBI TIFR0, TOV0	;Se restauran las banderas 
	INC R23			;Se incrementan las unidades por cada overflow

	POP R16
	OUT SREG, R16
	POP R16
	RETI
//***********************************************
