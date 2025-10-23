# 📚 Examen 2do Parcial

## 1) Resumen

- *Equipo / Autor(es):* Juan David García Cortéz y Sumie Arai Erazo  
- *Curso / Asignatura:* Sistemas embebidos 1  
- *Fecha:* 23/10/25  
- *Descripción breve:* _Examen 2do Parcial. _


## 2) Objetivos

- *General:* Aprender las utilidades del PWM
- *Específicos:*
  - Desensamblar y ensamblar strings sin que se corrompan los mensajes_
  - Utilizar frecuencias específicas para hacer notas musicales
  - Crear una señal sinusoidal a parir de una PWM y un filtro con capacitor y leerla con un osciloscopio

## 3) Alcance y Exclusiones

- *Incluye:* 
  -1 servomotor.
  -3 botones, resistencias pulldown de 1k, 1 motor DC._

 


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

## 5) Código
```bash
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/pwm.h"
#include <string>
#include <vector>

#define UART_ID uart0
#define BAUD_RATE 115200
#define UART_TX_PIN 0
#define UART_RX_PIN 1
#define SERVO_PIN 15
#define BTN_MODE 2   // Cambiar modo
#define BTN_NEXT 3   // Modo entrenamiento/step: siguiente
#define BTN_PREV 4   // Modo entrenamiento/step: anterior

using namespace std;

// --- Variables globales ---
vector<int> posiciones;       // Lista de ángulos guardados
int indice_actual = 0;        // Posición actual
volatile int modo_actual = 1; // 1: Entrenamiento, 2: Continuo, 3: Step
volatile bool ciclo_activo = false;

// --- Funciones ---
void borrar_lista() {
    posiciones.clear();
    indice_actual = 0;
    printf("Lista borrada\n");
}

uint16_t angle_to_level(uint16_t angle) {
    float pulse_us = 1000.0f + (angle * 1000.0f / 180.0f);
    return (uint16_t)((pulse_us / 20000.0f) * 65535);
}

void mover_servo(uint slice, uint chan, int ang) {
    if (ang < 0) ang = 0;
    if (ang > 180) ang = 180;
    pwm_set_chan_level(slice, chan, angle_to_level(ang));
}

// Interrupción: cambiar modo
void cambiar_modo(uint gpio, uint32_t events) {
    if (gpio == BTN_MODE) {
        modo_actual++;
        if (modo_actual > 3) modo_actual = 1;
        ciclo_activo = (modo_actual == 2);
        printf("\nModo actual: %d\n", modo_actual);
    }
}

// Procesar comando de consola
void procesar_comando(string cmd, uint slice, uint chan) {
    // Normalizar a minúsculas
    for (char &c : cmd) c = tolower(c);

    if (cmd == "write" || cmd == "escribir") {
        printf("Ingresa 3 valores separados por comas (ej: 0,90,130):\n");
        string entrada = "";
        while (true) {
            int c = getchar_timeout_us(0);
            if (c != PICO_ERROR_TIMEOUT) {
                if (c == '\n' || c == '\r') break;
                entrada += (char)c;
            }
        }
        posiciones.clear();
        string temp = "";
        int i=0;
        for (char c : entrada) {
            if (c == ',') {
                int val = stoi(temp);
                if (val<0 || val>180) { printf("Error: valores entre 0 y 180\n"); posiciones.clear(); return; }
                posiciones.push_back(val);
                temp=""; i++;
            } else temp+=c;
        }
        if (!temp.empty()) {
            int val = stoi(temp);
            if (val<0 || val>180) { printf("Error: valores entre 0 y 180\n"); posiciones.clear(); return; }
            posiciones.push_back(val);
        }
        indice_actual = 0;
        mover_servo(slice, chan, posiciones[indice_actual]);
        printf("Posiciones guardadas: ");
        for (int v : posiciones) printf("%d ", v);
        printf("\n");
    }
    else if (cmd == "clear" || cmd == "borrar") borrar_lista();
    else if (cmd == "replace" || cmd == "reemplazar") {
        printf("Formato: replace:posicion,valor (ej: replace:1,130)\n");
        string entrada=""; 
        while(true){
            int c = getchar_timeout_us(0);
            if(c!=PICO_ERROR_TIMEOUT){
                if(c=='\n'||c=='\r') break;
                entrada+=(char)c;
            }
        }
        int pos=-1,val=-1; string temp=""; bool sep=false;
        for(char c:entrada){
            if(c==','&&!sep){pos=stoi(temp)-1; temp=""; sep=true;}
            else temp+=c;
        }
        if(sep&&!temp.empty()) val=stoi(temp);
        if(pos<0||pos>=(int)posiciones.size()) printf("Error: posición inválida\n");
        else if(val<0||val>180) printf("Error: valor inválido\n");
        else { posiciones[pos]=val; printf("Reemplazado: pos%d=%d\n", pos+1,val);}
    }
    else printf("Comando no reconocido\n");
}

// --- Programa principal ---
int main() {
    stdio_init_all();
    sleep_ms(2000);
    printf("Sistema iniciado. Modo entrenamiento por defecto\n");

    // PWM servo
    gpio_set_function(SERVO_PIN, GPIO_FUNC_PWM);
    uint slice = pwm_gpio_to_slice_num(SERVO_PIN);
    uint chan  = pwm_gpio_to_channel(SERVO_PIN);
    pwm_set_wrap(slice, 65535);
    float f_clk = 125000000.0f;
    float div = f_clk / (50.0f * 65536.0f);
    pwm_set_clkdiv(slice, div);
    pwm_set_enabled(slice, true);

    // Botones
    gpio_init(BTN_MODE); gpio_set_dir(BTN_MODE, GPIO_IN); gpio_pull_up(BTN_MODE);
    gpio_init(BTN_NEXT); gpio_set_dir(BTN_NEXT, GPIO_IN); gpio_pull_up(BTN_NEXT);
    gpio_init(BTN_PREV); gpio_set_dir(BTN_PREV, GPIO_IN); gpio_pull_up(BTN_PREV);
    gpio_set_irq_enabled_with_callback(BTN_MODE, GPIO_IRQ_EDGE_FALL, true, &cambiar_modo);

    // Variables de control
    string mensaje_usb="";
    bool btn_next=false, btn_prev=false;

    while(true){
        // --- Lectura consola ---
        int ch = getchar_timeout_us(0);
        if(ch!=PICO_ERROR_TIMEOUT){
            if(ch=='\n'||ch=='\r'){
                if(!mensaje_usb.empty()){
                    procesar_comando(mensaje_usb, slice, chan);
                    mensaje_usb="";
                }
            } else mensaje_usb+=(char)ch;
        }

        // --- Modo entrenamiento ---
        if(modo_actual==1 && !posiciones.empty()){
            bool next = gpio_get(BTN_NEXT)==0;
            bool prev = gpio_get(BTN_PREV)==0;

            if(next && !btn_next){ // siguiente
                if(indice_actual<(int)posiciones.size()-1) indice_actual++;
                mover_servo(slice, chan, posiciones[indice_actual]);
                printf("Servo a %d°\n", posiciones[indice_actual]);
                btn_next=true;
            } else if(!next) btn_next=false;

            if(prev && !btn_prev){ // anterior
                if(indice_actual>0) indice_actual--;
                mover_servo(slice, chan, posiciones[indice_actual]);
                printf("Servo a %d°\n", posiciones[indice_actual]);
                btn_prev=true;
            } else if(!prev) btn_prev=false;
        }

        // --- Modo step ---
        if(modo_actual==3 && !posiciones.empty()){
            bool next = gpio_get(BTN_NEXT)==0;
            bool prev = gpio_get(BTN_PREV)==0;

            if(next && !btn_next){
                if(indice_actual<(int)posiciones.size()-1) indice_actual++;
                mover_servo(slice, chan, posiciones[indice_actual]);
                printf("Step: Servo a %d°\n", posiciones[indice_actual]);
                btn_next=true;
            } else if(!next) btn_next=false;

            if(prev && !btn_prev){
                if(indice_actual>0) indice_actual--;
                mover_servo(slice, chan, posiciones[indice_actual]);
                printf("Step: Servo a %d°\n", posiciones[indice_actual]);
                btn_prev=true;
            } else if(!prev) btn_prev=false;
        }

        // --- Modo continuo ---
        if(modo_actual==2 && ciclo_activo && !posiciones.empty()){
            for(int i=0;i<(int)posiciones.size();i++){
                mover_servo(slice, chan, posiciones[i]);
                printf("Continuo: pos%d=%d\n", i+1, posiciones[i]);
                for(int t=0;t<15;t++){sleep_ms(100); if(!ciclo_activo) break;}
                if(!ciclo_activo) break;
            }
        }

        sleep_ms(10);
    }
}
```