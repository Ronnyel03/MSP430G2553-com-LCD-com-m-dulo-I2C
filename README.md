#include <msp430g2553.h>
#include <stdlib.h>
#include <string.h>

#define SDA_PIN BIT7          // Definição do pino SDA
#define SCL_PIN BIT6          // Definição do pino SCL
#define PCF8574AT_ADDRESS 0x3F  // Endereço I2C do PCF8574AT

#define LCD_COMMAND 0  // Modo comando para LCD
#define LCD_DATA 1     // Modo dado para LCD

// Declaração das funções
void i2c_init(void);
void i2c_write(unsigned char address, unsigned char data);
void lcd_send(unsigned char value, unsigned char mode);
void lcd_init(void);
void lcd_command(unsigned char cmd);
void lcd_write_char(char data);
void lcd_set_cursor(unsigned char row, unsigned char col);
void lcd_print(char *str);
void lcd_scroll_text(char *str, unsigned char row);

// Inicialização do I2C
void i2c_init(void) {
    // Configuração dos pinos SDA e SCL
    P1SEL |= SDA_PIN + SCL_PIN;
    P1SEL2 |= SDA_PIN + SCL_PIN;

    UCB0CTL1 |= UCSWRST;                     // Habilita reset de software
    UCB0CTL0 = UCMST + UCMODE_3 + UCSYNC;    // Configura como mestre I2C, modo síncrono
    UCB0CTL1 = UCSSEL_2 + UCSWRST;           // Seleciona SMCLK, mantém reset de software
    UCB0BR0 = 10;                            // Configura clock I2C (SMCLK/10 = ~100kHz)
    UCB0BR1 = 0;
    UCB0I2CSA = PCF8574AT_ADDRESS;           // Define o endereço do dispositivo escravo (PCF8574AT)
    UCB0CTL1 &= ~UCSWRST;                    // Limpa reset de software, retoma operação
}

// Função para escrita I2C
void i2c_write(unsigned char address, unsigned char data) {
    while (UCB0CTL1 & UCTXSTP);              // Garante que a condição de parada foi enviada
    UCB0CTL1 |= UCTR + UCTXSTT;              // Modo de transmissão I2C, condição de start
    while (!(IFG2 & UCB0TXIFG));             // Espera até o buffer de transmissão estar pronto
    UCB0TXBUF = data;                        // Envia dados
    while (!(IFG2 & UCB0TXIFG));             // Espera até a transmissão completar
    UCB0CTL1 |= UCTXSTP;                     // Envia condição de parada
    while (UCB0CTL1 & UCTXSTP);              // Garante que a condição de parada foi enviada
}

// Função para enviar comandos ou dados para o LCD
void lcd_send(unsigned char value, unsigned char mode) {
    unsigned char high_nibble = value & 0xF0;
    unsigned char low_nibble = (value << 4) & 0xF0;

    unsigned char data = high_nibble | mode | 0x08;  // Envia nibble alto
    i2c_write(PCF8574AT_ADDRESS, data);
    i2c_write(PCF8574AT_ADDRESS, data | 0x04); // Envia enable alto
    i2c_write(PCF8574AT_ADDRESS, data & ~0x04); // Envia enable baixo

    data = low_nibble | mode | 0x08;  // Envia nibble baixo
    i2c_write(PCF8574AT_ADDRESS, data);
    i2c_write(PCF8574AT_ADDRESS, data | 0x04); // Envia enable alto
    i2c_write(PCF8574AT_ADDRESS, data & ~0x04); // Envia enable baixo
}

// Inicialização do LCD
void lcd_init(void) {
    __delay_cycles(50000); // Espera por 50ms para garantir a inicialização
    lcd_send(0x03, LCD_COMMAND);
    lcd_send(0x03, LCD_COMMAND);
    lcd_send(0x03, LCD_COMMAND);
    lcd_send(0x02, LCD_COMMAND);

    lcd_command(0x28);   // Interface 4-bit, 2 linhas, 5x7 pixels
    lcd_command(0x0C);   // Display ligado, cursor desligado
    lcd_command(0x06);   // Incrementa cursor
    lcd_command(0x01);   // Limpa display
    __delay_cycles(2000); // Espera por 2ms para garantir a execução dos comandos
}

// Envia comando para o LCD
void lcd_command(unsigned char cmd) {
    lcd_send(cmd, LCD_COMMAND);
}

// Escreve um caractere no LCD
void lcd_write_char(char data) {
    lcd_send(data, LCD_DATA);
}

// Define o cursor do LCD
void lcd_set_cursor(unsigned char row, unsigned char col) {
    unsigned char address;

    if(row == 0)
        address = 0x80 + col;    // Linha 0, coluna col
    else
        address = 0xC0 + col;    // Linha 1, coluna col

    lcd_command(address);
}

// Imprime uma string no LCD
void lcd_print(char *str) {
    while(*str) {
        lcd_write_char(*str++);
    }
}

// Função para rolar o texto no LCD
void lcd_scroll_text(char *str, unsigned char row) {
    unsigned char len = strlen(str);
    unsigned char display_len = 16;  // Comprimento do display LCD

    if (len <= display_len) {
        lcd_set_cursor(row, 0);
        lcd_print(str);
    } else {
        unsigned char i, j;
        for (i = 0; i < len - display_len + 1; i++) {
            lcd_set_cursor(row, 0);
            for (j = 0; j < display_len; j++) {
                if (i + j < len) {
                    lcd_write_char(str[i + j]);
                } else {
                    lcd_write_char(' ');
                }
            }
            __delay_cycles(500000); // Ajuste o valor para a velocidade desejada
        }
    }
}

int main(void) {
    WDTCTL = WDTPW | WDTHOLD;                // Para o watchdog timer

    // Configuração dos pinos SDA e SCL
    P1DIR |= SDA_PIN + SCL_PIN;
    P1OUT |= SDA_PIN + SCL_PIN;

    i2c_init();                              // Inicializa I2C
    lcd_init();                              // Inicializa LCD

    char *line1 = "Escreva o que voce quer que apareça na LInha 1";
    char *line2 = "Escreva o que voce quer que apareça na LInha 2";

    while (1) {
        lcd_scroll_text(line1, 0); // Exibe e rola a mensagem na linha 1
        lcd_scroll_text(line2, 1); // Exibe e rola a mensagem na linha 2
    }
}
