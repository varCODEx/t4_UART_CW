```
	; Task #42
	
	
	
   .nolist				; директива відключає генерацію коду у лістинг, 
                                        ;тобто далі у файлі *.lss не буде фіксуватися асемблерний код
   .include "m8515def.inc"		;підключення стандартного заголовочного файлу для ATmega8
   .list				 ;директива включає генерацію коду у лістинг, тобто далі у файлі *.lss буде фіксуватися асемблерний код


   .equ fCK = 8000000			;clock frequency in Hz
   .equ BAUD = 4800			;UART baudrate
   .equ UBRR_value = (fCK / (BAUD * 16)) - 1	; розрахунок значення для регістра UBRR

   .cseg				;директива визначення, що далі іде код програми
   .org 0				;директива визначення, що  код програми у FLASH буде розміщено з нульової адреси 


main:
   rcall init_USART	; (блок 1) відносний виклик підпрограми 
   
   ;set port C as OUT
   LDI R16, 0xFF		; (блок 2) R16 <-0xFF 
   OUT DDRC, R16		; (блок 2) DDRC <- R16 
   
   rcall USART_receive	; receive from UART [to R16]
   
   ;OUT the received data from port C (LEDs connected)
   OUT PORTC, R16	; (блок 4) PORTC <-R16 
   
   ;set port A as IN
   LDI R16, 0x00		; (блок 5) R16<-0x00 
   OUT DDRA, R16		; (блок 5) DDRА<- R16 
   
   
   ; read buttons into R16
   in R16, PINA		; (блок 6) R16<- PINA 
   ; out read to C (LEDs)
   OUT PORTC, R16	; (блок 7) PORTC<-R16 

   rcall USART_send	; send read data (R16) by UART

init_USART:
   ldi   R16, high(UBRR_value)	; (блок 1) R16 <-старший байт UBRR_value 
   out   UBRRH, R16		         ; (блок 1) UBRRH <-R16 
   ldi   R16, low(UBRR_value)       ; (блок 1) R16 <-молодший байт UBRR_value 
   out   UBRRL, R16		         ; (блок 1) UBRRL <-R16 
   
   ldi   R16, (1<<RXEN)|(1<<TXEN)|(0<<RXCIE)|(1<<TXCIE)|(1<<UDRIE)
   ; RXEN - UART Receiver enable
   ; TXEN - UART Transmitter enable
   ; RXCIE - UART RX Complete Interrupt Enable (0 - disabled)
   ; TXCIE - UART TX Complete Interrupt Enable
   ; UDRIE - UART Data Register Empty Interrupt Enable 
   out   UCSRB, R16		          ; write settings above to their corresponding UCSRB register
   
   
   ; 7 бітів даних, перевірка на непарність, 2 стоп-біти
   ldi   R16, (1<< URSEL)|(1<<UPM1)|(1<<UPM0)|(1<< UCSZ1)|(0<< UCSZ0)|(1<<USBS) 
   ; URSEL 1 means we write to UCSRC (not UBRRH)
   ; UPM1,0 set to 1,1 - odd parity (outputs 1 if ODD | 0 if EVEN)
   ; UCSZ2,1,0 set to 0,1,0 - 7 data bits
   ; USBS set to 1 - 2 stop bits
   out   UCSRC, R16			; (блок 1) UCSRC <- R16 
   
   ret				       ; return [to main]

; передача через модуль USART
USART_send:
   out   UDR, R16			     ; populate UDR (UART Data register) with data from R16
   
   sending:	;loop while TXC (USART Transmit Complete Flag) != 1 (1 would mean send is complete)
      sbis  UCSRA, TXC        ; (блок 8) якщо біт TXC = 1, то наступна ;команда пропускається, інакше  –  наступна команда 
      rjmp  sending		; loop
   ret			          ; return

USART_receive:
      sbis  UCSRA, RXC          ; loop while receiving
   rjmp  USART_receive	; loop

   in	 R16, UDR		; populate R16 with read data
   ret			        ; return



  
```