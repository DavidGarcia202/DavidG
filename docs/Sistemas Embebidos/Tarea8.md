# 📚 TAREA 8

## 1) Resumen

- *Nombre del proyecto:* Comunicacion uart y usb  
- *Equipo / Autor(es):* Juan David García Cortéz y Sumie Arai Erazo  
- *Curso / Asignatura:* Sistemas embebidos 1  
- *Fecha:* 22/10/25  
- *Descripción breve:* _En equipos de 4 conectar dos picos por medio de rx y tx bidireccional para intercambiar mensajes escritos en la consolas de las computadoras. _


## 2) Objetivos

- *General:* Aprender las utilidades del PWM
- *Específicos:*
  - Desensamblar y ensamblar strings sin que se corrompan los mensajes_
  - Utilizar frecuencias específicas para hacer notas musicales
  - Crear una señal sinusoidal a parir de una PWM y un filtro con capacitor y leerla con un osciloscopio

## 3) Alcance y Exclusiones

- *Incluye:* 
  - 1 capacitor de 1uF, resistencias de 1k y 680 ohms, osciloscopio
  -1 buzzer pasivo, 1 resistencia de 220.
  -2 botones, resistencias pulldown de 1k, 1 motor DC._

 


---

## 4) Requisitos

*Software*
- SO compatible (Windows/Linux/macOS)
- Python 3.x / visual studio / raspberry pi pico.
- "pico/stdlib.h", "hardware/structs/sio.h"

*Conocimientos previos*
- Programación básica en C
- Electrónica básica
- Git/GitHub

---

## 5) Códigos
```bash
#include "pico/stdlib.h"
#include "hardware/uart.h"
#include <stdio.h>
#include <string>

#define UART_ID uart0
#define BAUD_RATE 115200
#define TX_PIN 0
#define RX_PIN 1
#define button_pin 17
#define led_PIN 16
using namespace std; //USO DE STRING en la terminal 

int main() {
    stdio_init_all();

    gpio_set_function(TX_PIN, GPIO_FUNC_UART); // DEFINE TX Y RX
    gpio_set_function(RX_PIN, GPIO_FUNC_UART);

    uart_init(UART_ID, BAUD_RATE); //VELOCIDAD DE TRANSMISIÓN, 
    uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE); //NO QUEREMOS ENCONTRAR ERRORES DE TRANSMISIÓN, para saber como empieza el msj

    gpio_init(button_pin);
    gpio_set_dir(button_pin, GPIO_IN);
    gpio_pull_up(button_pin);
    gpio_init(led_PIN);
    gpio_set_dir(led_PIN, GPIO_OUT);

    string c = ""; // DEFINE C Y P COMO VARIIABLE DE RECONSTRUCCION, C LA RECIBE LA PALABRA  
    string p=""; //RECONSTRUYE LA PALABRA QUE VAMOS A ENVIAR
    while (true){

        int ch = getchar_timeout_us(0); //LEE EL CARACTER
        if (ch != PICO_ERROR_TIMEOUT) { //SI NO HAY ERROR, ENTONCES IMPRIME EL CARACTER
            printf("Eco: %c\n", (char)ch); //IMPRIME EL CARACTER DE LA PALABRA RECIBIDA, 
            p+= (char)ch; //RECONSTRUYE LA PALABRA EN P
            
            if(ch=='.' || ch=='\n'){ // CUANDO ENTRA UN PUNTO O UN ENTER 
                uart_puts(UART_ID, p.c_str()); //MANDA LA PALABRA P, AL OTRO PICO, CONVIERTE UN STRING EN UN ARREGLO DE CARACTERES
                p=""; //VACIAMOS LA PALBRA P, PARA ENVIAR UNA NUEVA
            }
        }
        int a;
        if (gpio_get(button_pin) == 0 && a == 1) {
            printf("Button pressed!\n");//HACE QUE FUNCIONE EL BOTON CUANDO SE PRESIONA
            uart_puts(UART_ID, "LEDON\n");
            sleep_ms(200); 
        }
         a= gpio_get(button_pin); //TOMA EL VALOR DEL BOTON PIN

        if (uart_is_readable(uart0)) { //SI HAY DATOS PARA LEER
            char character = uart_getc(uart0); //LEE EL CARACTER
            printf(character+"\n"); //IMPRIME EL CARACTER RECIBIDO
           
            if(character=='\n' || character=='.'){ //CUANDO ENTRA UN PUNTO O UN ENTER
                if (c == "LEDON"){
                    gpio_put(led_PIN, 1);
                    printf("LED is ON\n");
                }
                else if (c == "LEDOFF"){
                    gpio_put(led_PIN, 0);
                    printf("LED is OFF\n");
                
                } 
                else if (c == "Invalid Command"){
                    printf("Invalid Command\n");

                }
                else{
                    uart_puts(UART_ID, "Invalid Command\n");
                }
                c = "";
                continue;
            }
            else{
                c += character;
            }
        }
    }
}
```