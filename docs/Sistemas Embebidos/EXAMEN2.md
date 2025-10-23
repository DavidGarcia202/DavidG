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
#include "hardware/gpio.h"
#include <string>
#include <vector>
#include <cctype>

#define SERVO_PIN 15
#define BTN_MODE 2
#define BTN_NEXT 3
#define BTN_PREV 4

using namespace std;

// --- Variables globales ---
vector<int> posiciones;
int indice_actual = 0;
volatile int modo_actual = 1;
volatile bool ciclo_activo = false;

// --- Funciones auxiliares ---
bool es_numero(const string& str) {
    if (str.empty()) return false;
    for (char c : str) {
        if (!isdigit(c) && c != '-') return false;
    }
    return true;
}

int string_a_int(const string& str) {
    int resultado = 0;
    int signo = 1;
    size_t inicio = 0;
    
    if (str[0] == '-') {
        signo = -1;
        inicio = 1;
    }
    
    for (size_t i = inicio; i < str.length(); i++) {
        if (isdigit(str[i])) {
            resultado = resultado * 10 + (str[i] - '0');
        } else {
            return -9999;
        }
    }
    
    return resultado * signo;
}

void mostrar_bienvenida() {
    printf("\n");
    printf("=========================================\n");
    printf("    SISTEMA DE CONTROL DE SERVO\n");
    printf("         Raspberry Pi Pico\n");
    printf("=========================================\n");
    printf("Modos disponibles:\n");
    printf("  1 - Entrenamiento\n");
    printf("  2 - Continuo\n");
    printf("  3 - Step\n");
    printf("\n");
    printf("Comandos disponibles:\n");
    printf("  escribir, write - Guardar posiciones\n");
    printf("  borrar, clear   - Borrar lista\n");
    printf("  reemplazar, replace - Modificar posicion\n");
    printf("  help, ayuda     - Mostrar este mensaje\n");
    printf("\n");
    printf("Presiona BTN_MODE para cambiar modos\n");
    printf("=========================================\n");
    printf("\n");
}

void borrar_lista() {
    posiciones.clear();
    indice_actual = 0;
    printf("OK\n");
}

uint16_t angle_to_level(uint16_t angle) {
    // Para servo estándar: 0° = 1000us, 180° = 2000us
    float min_pulse_us = 1000.0f;   // 0 grados
    float max_pulse_us = 2000.0f;   // 180 grados  
    float period_us = 20000.0f;     // Periodo de 20ms (50Hz)
    
    // Calcular pulso para el ángulo
    float pulse_us = min_pulse_us + (angle * (max_pulse_us - min_pulse_us) / 180.0f);
    
    // Convertir a nivel PWM
    return (uint16_t)((pulse_us / period_us) * 65535.0f);
}

void mover_servo(uint slice, uint chan, int ang) {
    if (ang < 0) ang = 0;
    if (ang > 180) ang = 180;
    pwm_set_chan_level(slice, chan, angle_to_level(ang));
}

// Interrupción para cambiar modo
void gpio_callback(uint gpio, uint32_t events) {
    if (gpio == BTN_MODE) {
        modo_actual++;
        if (modo_actual > 3) modo_actual = 1;
        ciclo_activo = (modo_actual == 2);
        
        printf("\n=== Modo cambiado ===\n");
        switch(modo_actual) {
            case 1: printf("MODO ENTRENAMIENTO activado\n"); break;
            case 2: printf("MODO CONTINUO activado\n"); break;
            case 3: printf("MODO STEP activado\n"); break;
        }
        printf("=====================\n");
        
        if (!posiciones.empty()) {
            indice_actual = 0;
        }
    }
}

// Procesar comando de consola
void procesar_comando(const string& cmd, uint slice, uint chan) {
    string cmd_lower = cmd;
    for (char &c : cmd_lower) c = tolower(c);

    if (cmd_lower == "write" || cmd_lower == "escribir") {
        printf("Ingresa valores separados por comas (ej: 0,90,130): ");
        
        string entrada = "";
        absolute_time_t timeout = make_timeout_time_ms(10000);
        
        while (true) {
            int c = getchar_timeout_us(100000);
            if (c != PICO_ERROR_TIMEOUT) {
                if (c == '\n' || c == '\r') break;
                entrada += (char)c;
                printf("%c", c);
            }
            if (time_reached(timeout)) {
                printf("\nError: timeout en entrada\n");
                return;
            }
        }
        printf("\n");
        
        if (entrada.empty()) {
            printf("Error argumento invalido\n");
            return;
        }
        
        vector<int> temp_pos;
        string temp = "";
        
        for (char c : entrada) {
            if (c == ',') {
                if (!temp.empty()) {
                    if (!es_numero(temp)) {
                        printf("Error argumento invalido\n");
                        return;
                    }
                    int val = string_a_int(temp);
                    if (val < 0 || val > 180) {
                        printf("Error argumento invalido\n");
                        return;
                    }
                    temp_pos.push_back(val);
                    temp = "";
                }
            } else if (c != ' ') {
                temp += c;
            }
        }
        
        if (!temp.empty()) {
            if (!es_numero(temp)) {
                printf("Error argumento invalido\n");
                return;
            }
            int val = string_a_int(temp);
            if (val < 0 || val > 180) {
                printf("Error argumento invalido\n");
                return;
            }
            temp_pos.push_back(val);
        }
        
        if (temp_pos.empty()) {
            printf("Error argumento invalido\n");
            return;
        }
        
        posiciones = temp_pos;
        indice_actual = 0;
        printf("OK - %d posiciones guardadas: ", (int)posiciones.size());
        for (size_t i = 0; i < posiciones.size(); i++) {
            printf("%d", posiciones[i]);
            if (i < posiciones.size() - 1) printf(", ");
        }
        printf("\n");
        
    } else if (cmd_lower == "clear" || cmd_lower == "borrar") {
        borrar_lista();
        
    } else if (cmd_lower == "replace" || cmd_lower == "reemplazar") {
        printf("Ingresa posicion,valor (ej: 1,130): ");
        string entrada = "";
        absolute_time_t timeout = make_timeout_time_ms(10000);
        
        while (true) {
            int c = getchar_timeout_us(100000);
            if (c != PICO_ERROR_TIMEOUT) {
                if (c == '\n' || c == '\r') break;
                entrada += (char)c;
                printf("%c", c);
            }
            if (time_reached(timeout)) {
                printf("\nError: timeout en entrada\n");
                return;
            }
        }
        printf("\n");
        
        if (entrada.empty()) {
            printf("Error argumento invalido\n");
            return;
        }
        
        int pos = -1, val = -1;
        string temp = "";
        bool sep = false;
        
        for (char c : entrada) {
            if (c == ',' && !sep) {
                if (!es_numero(temp)) {
                    printf("Error argumento invalido\n");
                    return;
                }
                pos = string_a_int(temp) - 1;
                if (pos == -10000) {
                    printf("Error argumento invalido\n");
                    return;
                }
                temp = "";
                sep = true;
            } else if (c != ' ') {
                temp += c;
            }
        }
        
        if (sep && !temp.empty()) {
            if (!es_numero(temp)) {
                printf("Error argumento invalido\n");
                return;
            }
            val = string_a_int(temp);
            if (val == -9999) {
                printf("Error argumento invalido\n");
                return;
            }
        }
        
        if (pos < 0 || pos >= (int)posiciones.size()) {
            printf("Error indice invalido\n");
        } else if (val < 0 || val > 180) {
            printf("Error argumento invalido\n");
        } else {
            posiciones[pos] = val;
            printf("OK - Posicion %d actualizada a %d grados\n", pos + 1, val);
        }
        
    } else if (cmd_lower == "help" || cmd_lower == "ayuda" || cmd_lower == "?") {
        mostrar_bienvenida();
        
    } else {
        printf("Comando no reconocido: '%s'\n", cmd.c_str());
        printf("Escribe 'help' para ver comandos disponibles\n");
    }
}

// --- Programa principal ---
int main() {
    // Inicializar USB serial
    stdio_init_all();
    
    // Esperar a que se conecte el monitor serial
    sleep_ms(3000);
    
    // Mostrar mensaje de bienvenida
    mostrar_bienvenida();
    printf("Sistema inicializado. Esperando comandos...\n");
    printf("> ");

    

    // Configurar PWM para servo
    // Configurar PWM para servo
    gpio_set_function(SERVO_PIN, GPIO_FUNC_PWM);
    uint slice = pwm_gpio_to_slice_num(SERVO_PIN);
    uint chan = pwm_gpio_to_channel(SERVO_PIN);
    pwm_set_wrap(slice, 65535);

    // Configurar el divisor para 50Hz
    float div = 125.0f / (50.0f * 65535.0f / 1000000.0f);
    pwm_set_clkdiv(slice, div);

    pwm_set_enabled(slice, true);

    // Configurar botones
    gpio_init(BTN_MODE);
    gpio_set_dir(BTN_MODE, GPIO_IN);
    gpio_pull_up(BTN_MODE);
    
    gpio_init(BTN_NEXT);
    gpio_set_dir(BTN_NEXT, GPIO_IN);
    gpio_pull_up(BTN_NEXT);
    
    gpio_init(BTN_PREV);
    gpio_set_dir(BTN_PREV, GPIO_IN);
    gpio_pull_up(BTN_PREV);

    // Configurar interrupción para BTN_MODE
    gpio_set_irq_enabled(BTN_MODE, GPIO_IRQ_EDGE_FALL, true);
    gpio_set_irq_callback(gpio_callback);
    irq_set_enabled(IO_IRQ_BANK0, true);

    string mensaje_usb = "";
    bool btn_next_prev = false;
    bool btn_prev_prev = false;

    while (true) {
        // Leer comandos por USB
        int ch = getchar_timeout_us(1000);
        if (ch != PICO_ERROR_TIMEOUT) {
            if (ch == '\n' || ch == '\r') {
                if (!mensaje_usb.empty()) {
                    procesar_comando(mensaje_usb, slice, chan);
                    mensaje_usb = "";
                    printf("> ");
                }
            } else {
                mensaje_usb += (char)ch;
            }
        }

        // Modo Step
        if (modo_actual == 3) {
            bool btn_next_actual = !gpio_get(BTN_NEXT);
            bool btn_prev_actual = !gpio_get(BTN_PREV);
            
            if (btn_next_actual && !btn_next_prev) {
                if (posiciones.empty()) {
                    printf("Error no hay pos\n");
                } else {
                    if (indice_actual < (int)posiciones.size() - 1) {
                        indice_actual++;
                    }
                    mover_servo(slice, chan, posiciones[indice_actual]);
                    printf("pos%d: %d\n", indice_actual + 1, posiciones[indice_actual]);
                }
            }
            
            if (btn_prev_actual && !btn_prev_prev) {
                if (posiciones.empty()) {
                    printf("Error no hay pos\n");
                } else {
                    if (indice_actual > 0) {
                        indice_actual--;
                    }
                    mover_servo(slice, chan, posiciones[indice_actual]);
                    printf("pos%d: %d\n", indice_actual + 1, posiciones[indice_actual]);
                }
            }
            
            btn_next_prev = btn_next_actual;
            btn_prev_prev = btn_prev_actual;
        }

        // Modo Continuo
        if (modo_actual == 2 && ciclo_activo) {
            if (posiciones.empty()) {
                printf("Error no hay pos\n");
                sleep_ms(1500);
            } else {
                for (int i = 0; i < (int)posiciones.size(); i++) {
                    if (!ciclo_activo) break;
                    
                    mover_servo(slice, chan, posiciones[i]);
                    printf("pos%d: %d\n", i + 1, posiciones[i]);
                    
                    absolute_time_t start_time = get_absolute_time();
                    while (absolute_time_diff_us(start_time, get_absolute_time()) < 1500000) {
                        sleep_ms(100);
                        if (!ciclo_activo) break;
                    }
                }
            }
        }

        // Modo Entrenamiento
        if (modo_actual == 1 && !posiciones.empty()) {
            bool btn_next_actual = !gpio_get(BTN_NEXT);
            bool btn_prev_actual = !gpio_get(BTN_PREV);
            
            if (btn_next_actual && !btn_next_prev) {
                if (indice_actual < (int)posiciones.size() - 1) {
                    indice_actual++;
                }
                mover_servo(slice, chan, posiciones[indice_actual]);
                printf("Servo a %d° (pos%d)\n", posiciones[indice_actual], indice_actual + 1);
            }
            
            if (btn_prev_actual && !btn_prev_prev) {
                if (indice_actual > 0) {
                    indice_actual--;
                }
                mover_servo(slice, chan, posiciones[indice_actual]);
                printf("Servo a %d° (pos%d)\n", posiciones[indice_actual], indice_actual + 1);
            }
            
            btn_next_prev = btn_next_actual;
            btn_prev_prev = btn_prev_actual;
        }

        sleep_ms(10);
    }

    return 0;
}
```