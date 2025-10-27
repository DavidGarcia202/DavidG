# 📚 Examen 2do Parcial

## 1) Resumen

- *Equipo / Autor(es):* Juan David García Cortéz y Sumie Arai Erazo  
- *Curso / Asignatura:* Sistemas embebidos 1  
- *Fecha:* 23/10/25  
- *Descripción breve:* _Examen 2do Parcial. _


## 2) Instrucciones

   - Hardware mínimo: 1 × servomotor en un pin PWM (50 Hz).
    - Hardware mínimo: 3 × botones: BTN_MODE (cambia modo), BTN_NEXT (siguiente), BTN_PREV (anterior).
    - Usar Raspberry Pi Pico como controlador.
    - Modo Entrenamiento: recibir comandos por USB-serial (borrar/clear, escribir/write, reemplazar/replace) y responder OK o errores según validez de argumentos.
    - Modo Continuo: recorrer todas las posiciones de la lista en orden y mostrar cada 1.5 s el texto "posX: V"; si la lista está vacía, imprimir "Error no hay pos" cada 1.5 s y no mover el servo.
    - Modo Step: BTN_NEXT avanza una posición (no pasa de la última); BTN_PREV retrocede (no baja de la primera); en cada cambio imprimir "posX: V" y mover el servo; si la lista está vacía, al presionar los botones imprimir "Error no hay pos" y no mover el servo.
    - Comandos seriales (aceptar mayúsculas/minúsculas y aliases en inglés):
        - Borrar / clear: sintaxis "Borrar" — elimina la lista completa de posiciones — respuesta: OK.
        - Escribir / write: sintaxis "Escribir, v1, v2, ..., vn" con valores 0–180 — sobrescribe la lista — respuesta: OK si válidos; si alguno inválido o lista vacía → "Error argumento invalido".
        - Reemplazar / replace: sintaxis "Reemplazar, i, v" con i base 1 y v en 0–180 — reemplaza posición i por v — respuesta: OK; si i no existe → "Error indice invalido"; si v fuera de rango → "Error argumento invalido".
    - Formato de salida serial en modos: imprimir "posX: V" donde X es índice base 1 y V el ángulo.
    - Nota de conexión y PWM: el movimiento del servo requiere alimentación 5–6 V; la señal de control se conecta al pin de señal del servo; la frecuencia PWM debe ser 50 Hz y el pulso de control entre 1 ms (0°) y 2 ms (180°).

## 3) Materiales

- *Incluye:* 
  -1 servomotor.
  -3 botones, resistencias pulldown de 1k, 1 motor DC._

 


## 4) Código
```bash
#include <stdio.h>                          // include de C: funciones de entrada/salida (printf, getchar, etc.)
#include "pico/stdlib.h"                    // SDK de la Raspberry Pi Pico: sleep_ms, stdio_init_all, make_timeout_time_ms, funciones de tiempo y stdio USB
#include "hardware/pwm.h"                  // API hardware para PWM en la Pico (pwm_set_chan_level, pwm_set_wrap, pwm_set_clkdiv, etc.)
#include "hardware/gpio.h"                 // API hardware para GPIO (gpio_init, gpio_set_dir, gpio_pull_up, gpio_set_irq_enabled, gpio_get, etc.)
#include <string>                           // std::string (clase C++ para manejo de cadenas)
#include <vector>                           // std::vector (contenedor dinámico C++)
#include <cctype>                           // funciones para caracteres: isdigit, tolower, etc.

#define SERVO_PIN 15                       // Macro: nombre simbólico para pin del servo (GPIO 15). Reemplazo por el preprocesador.
#define BTN_MODE 2                         // Pin del botón que cambia el modo (GPIO 2)
#define BTN_NEXT 3                         // Pin del botón "siguiente" (GPIO 3)
#define BTN_PREV 4                         // Pin del botón "anterior" (GPIO 4)

using namespace std;                       // Evita escribir std:: antes de string, vector, etc. (útil pero puede contaminar el namespace)


// --- Variables globales ---
vector<int> posiciones;                    // Vector dinámico de enteros: lista de ángulos (0-180) que el servo debe usar.
int indice_actual = 0;                     // Índice actual dentro de 'posiciones' (base 0)
volatile int modo_actual = 1;              // Variable compartida con la ISR: modo actual (1=Entrenamiento,2=Continuo,3=Step)
volatile bool ciclo_activo = false;        // Bandera compartida: si el ciclo del modo continuo está activo o no
                                          // 'volatile' evita optimizaciones que asumirían que la variable no cambia fuera del flujo principal.


// --- Funciones auxiliares ---
bool es_numero(const string& str) {        // Función: devuelve true si 'str' contiene solo dígitos o un '-' al inicio (permite negativos)
    if (str.empty()) return false;         // Si cadena vacía -> no es número
    for (char c : str) {                   // Recorre cada carácter de la cadena
        if (!isdigit(c) && c != '-') return false; // isdigit() viene de <cctype>; si no es dígito ni '-', no es número válido
    }
    return true;                           // Si todos los caracteres pasaron la prueba, es número (o probable número con signo)
}

int string_a_int(const string& str) {      // Convierte cadena decimal a int SIN usar stoi (control manual de errores)
    int resultado = 0;                     // Acumulador del número
    int signo = 1;                         // Factor de signo (1 o -1)
    size_t inicio = 0;                     // Posición de inicio para lectura (si hay '-', empezamos en 1)
    if (str[0] == '-') {                   // Si el primer carácter es '-', manejamos negativo
        signo = -1;
        inicio = 1;
    }
    for (size_t i = inicio; i < str.length(); i++) { // Recorre desde 'inicio' hasta el final
        if (isdigit(str[i])) {             // Si el carácter es dígito
            resultado = resultado * 10 + (str[i] - '0'); // Convierte carácter a número: '3' -> 3 sumándolo en base 10
        } else {
            return -9999;                  // Código de error: se encontró carácter inválido (evita excepciones)
        }
    }
    return resultado * signo;              // Devuelve el valor con signo
}

void mostrar_bienvenida() {                // Imprime por serial el menú y comandos disponibles
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

void borrar_lista() {                      // Borra el vector de posiciones y reinicia el índice
    posiciones.clear();                    // .clear() borra todos los elementos del vector (tamaño=0)
    indice_actual = 0;                     // Reinicia índice
    printf("OK\n");                        // Confirma la operación por serial
}

uint16_t angle_to_level(uint16_t angle) {  // Convierte ángulo (0-180) a nivel PWM (0-65535)
    // Para servo estándar: 0° = 1000us, 180° = 2000us
    float min_pulse_us = 1000.0f;          // Pulso mínimo en microsegundos (1 ms) equivalente a 0°
    float max_pulse_us = 2000.0f;          // Pulso máximo en microsegundos (2 ms) equivalente a 180°
    float period_us = 20000.0f;            // Periodo total en microsegundos: 20 ms = 1 / 50 Hz (50 Hz típico de servos)

    // Calcular pulso para el ángulo (interpolación lineal)
    float pulse_us = min_pulse_us + (angle * (max_pulse_us - min_pulse_us) / 180.0f);
    // Explicación: (max - min) es la amplitud del pulso; se escala por angle/180 para mapear 0..180 a 0..(max-min).

    // Convertir a nivel PWM de 16 bits:
    // PWM level = (pulse_us / period_us) * 65535
    // 65535 es el valor máximo del contador cuando wrap=65535 (resolución 16-bit).
    return (uint16_t)((pulse_us / period_us) * 65535.0f);
}

void mover_servo(uint slice, uint chan, int ang) { // Envía el valor PWM correspondiente al ángulo
    if (ang < 0) ang = 0;                  // Clampa ángulo mínimo
    if (ang > 180) ang = 180;              // Clampa ángulo máximo
    pwm_set_chan_level(slice, chan, angle_to_level(ang)); // Llama al SDK: fija el nivel del canal PWM
    // pwm_set_chan_level(slice,chan,level) escribe el valor (0..wrap) que se mantiene activo en cada ciclo PWM.
}

// Interrupción para cambiar modo
void gpio_callback(uint gpio, uint32_t events) { // Función llamada por la IRQ del GPIO (callback)
    if (gpio == BTN_MODE) {                  // Si la interrupción vino del pin BTN_MODE
        modo_actual++;                       // Incrementa modo cíclicamente
        if (modo_actual > 3) modo_actual = 1;// Vuelve a 1 después de 3 (1->2->3->1)
        ciclo_activo = (modo_actual == 2);   // Si modo==2 (Continuo), activa el ciclo; si no, lo desactiva

        printf("\n=== Modo cambiado ===\n");  // Mensaje de cambio de modo
        switch(modo_actual) {                // Muestra texto según el modo
            case 1: printf("MODO ENTRENAMIENTO activado\n"); break;
            case 2: printf("MODO CONTINUO activado\n"); break;
            case 3: printf("MODO STEP activado\n"); break;
        }
        printf("=====================\n");

        if (!posiciones.empty()) {           // Si hay posiciones cargadas, reinicia el índice a 0
            indice_actual = 0;
        }
    }
}

// Procesar comando de consola
void procesar_comando(const string& cmd, uint slice, uint chan) { // Recibe el comando como string y los ids PWM slice/chan
    string cmd_lower = cmd;                 // Copia del comando
    for (char &c : cmd_lower) c = tolower(c); // Convierte todos los caracteres a minúsculas para comparar sin case-sensitivity

    if (cmd_lower == "write" || cmd_lower == "escribir") { // Comando para escribir posiciones
        printf("Ingresa valores separados por comas (ej: 0,90,130): ");
        string entrada = "";

        absolute_time_t timeout = make_timeout_time_ms(10000); // Crea un timeout de 10 s (evita bloqueo indefinido)
        while (true) {                       // Bucle para leer caracteres desde stdio (USB serial)
            int c = getchar_timeout_us(100000); // Lee con timeout en microsegundos (100000us = 100ms)
            if (c != PICO_ERROR_TIMEOUT) {   // Si hay dato disponible
                if (c == '\n' || c == '\r') break; // Enter -> fin de entrada
                entrada += (char)c;         // Agrega carácter a la cadena
                printf("%c", c);            // Eco del carácter (para ver lo que se escribe)
            }
            if (time_reached(timeout)) {    // Si el tiempo de entrada expiró
                printf("\nError: timeout en entrada\n");
                return;                     // Sale sin procesar (evita bloquear)
            }
        }
        printf("\n");

        if (entrada.empty()) {              // Si no se ingresó nada
            printf("Error argumento invalido\n");
            return;
        }

        vector<int> temp_pos;               // Vector temporal donde guardaremos los valores parseados
        string temp = "";

        for (char c : entrada) {            // Parseo manual de la lista separada por comas
            if (c == ',') {                 // Si encontramos separador
                if (!temp.empty()) {
                    if (!es_numero(temp)) { // Validación: temp debe ser numérico
                        printf("Error argumento invalido\n");
                        return;
                    }
                    int val = string_a_int(temp); // Conversión segura a int
                    if (val < 0 || val > 180) {   // Validación de rango de ángulo
                        printf("Error argumento invalido\n");
                        return;
                    }
                    temp_pos.push_back(val);  // Añade al vector temporal
                    temp = "";                // Resetea temp para el siguiente número
                }
            } else if (c != ' ') {          // Ignora espacios
                temp += c;                  // Acumula carácter
            }
        }
        if (!temp.empty()) {                // Si quedó un último número tras el último separador
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

        if (temp_pos.empty()) {             // Si no se logró parsear nada válido
            printf("Error argumento invalido\n");
            return;
        }

        posiciones = temp_pos;              // Asigna el vector parseado a la variable global
        indice_actual = 0;                  // Reinicia índice
        printf("OK - %d posiciones guardadas: ", (int)posiciones.size());
        for (size_t i = 0; i < posiciones.size(); i++) { // Muestra las posiciones guardadas
            printf("%d", posiciones[i]);
            if (i < posiciones.size() - 1) printf(", ");
        }
        printf("\n");

    } else if (cmd_lower == "clear" || cmd_lower == "borrar") { // Comando borrar lista
        borrar_lista();

    } else if (cmd_lower == "replace" || cmd_lower == "reemplazar") { // Reemplazar una posición dada
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

        int pos = -1, val = -1;             // Inicializa variables para índice (pos) y valor (val)
        string temp = "";
        bool sep = false;                   // Indica si ya leímos la coma separadora
        for (char c : entrada) {
            if (c == ',' && !sep) {         // En la primera coma separamos posición y valor
                if (!es_numero(temp)) {
                    printf("Error argumento invalido\n");
                    return;
                }
                pos = string_a_int(temp) - 1; // La interfaz pide posiciones 1-based, internamente se usa 0-based => restamos 1
                if (pos == -10000) {        // Verifica códigos de error (aunque string_a_int devuelve -9999, es chequeo defensivo)
                    printf("Error argumento invalido\n");
                    return;
                }
                temp = "";
                sep = true;
            } else if (c != ' ') {
                temp += c;                  // Acumula el valor después de la coma
            }
        }
        if (sep && !temp.empty()) {         // Si hubo coma y quedó valor para leer
            if (!es_numero(temp)) {
                printf("Error argumento invalido\n");
                return;
            }
            val = string_a_int(temp);
            if (val == -9999) {            // Código de error desde string_a_int
                printf("Error argumento invalido\n");
                return;
            }
        }

        if (pos < 0 || pos >= (int)posiciones.size()) { // Validación del índice dentro del vector
            printf("Error indice invalido\n");
        } else if (val < 0 || val > 180) {  // Validación del nuevo valor
            printf("Error argumento invalido\n");
        } else {
            posiciones[pos] = val;          // Reemplaza la posición solicitada
            printf("OK - Posicion %d actualizada a %d grados\n", pos + 1, val); // Muestra confirmación (pos+1 para usuario)
        }

    } else if (cmd_lower == "help" || cmd_lower == "ayuda" || cmd_lower == "?") { // Comando ayuda
        mostrar_bienvenida();

    } else {                               // Comando no reconocido
        printf("Comando no reconocido: '%s'\n", cmd.c_str());
        printf("Escribe 'help' para ver comandos disponibles\n");
    }
}

// --- Programa principal ---
int main() {
    // Inicializar USB serial
    stdio_init_all();                      // Inicializa stdio (incluye USB-Serial si está habilitado en la build)

    // Esperar a que se conecte el monitor serial
    sleep_ms(3000);                        // Pausa 3000 ms para dar tiempo al host a abrir el puerto serie

    // Mostrar mensaje de bienvenida
    mostrar_bienvenida();
    printf("Sistema inicializado. Esperando comandos...\n");
    printf("> ");                          // Prompt para el usuario

    // Configurar PWM para servo
    gpio_set_function(SERVO_PIN, GPIO_FUNC_PWM); // Asigna la función PWM al pin físico SERVO_PIN (GPIO 15)
    uint slice = pwm_gpio_to_slice_num(SERVO_PIN); // Obtiene el número de "slice" PWM asociado a ese pin
    uint chan = pwm_gpio_to_channel(SERVO_PIN); // Obtiene el canal (A/B) dentro del slice para ese pin
    pwm_set_wrap(slice, 65535);            // 'wrap' define el valor máximo del contador PWM (aquí 65535 → 16 bits)

    // Configurar el divisor para 50Hz
    // Explicación matemática:
    // PWM_freq = sys_clock_hz / (clkdiv * (wrap + 1))
    // => clkdiv = sys_clock_hz / (PWM_freq * (wrap + 1))
    // En la Pico típicamente sys_clock_hz = 125000000 (125 MHz)
    // Con wrap = 65535 (=> wrap+1 = 65536) y PWM_freq = 50 Hz:
    // clkdiv ≈ 125e6 / (50 * 65536) ≈ 38.146...
    // El autor escribió la fórmula manipulando unidades para obtener el mismo resultado:
    float div = 125.0f / (50.0f * 65535.0f / 1000000.0f);
    // - 125.0f representa 125 MHz (la frecuencia base en MHz)
    // - 65535.0f / 1000000.0f convierte el 'wrap' a "mega-unidades" para que las unidades cuadren.
    // Resultado: div ≈ 38.15 -> valor de clkdiv que produce ~50 Hz.
    pwm_set_clkdiv(slice, div);            // Ajusta el divisor del reloj PWM (acepta float)
    pwm_set_enabled(slice, true);          // Habilita el slice PWM

    // Configurar botones (modo: entrada con pull-up)
    gpio_init(BTN_MODE);                   // Inicializa pin BTN_MODE
    gpio_set_dir(BTN_MODE, GPIO_IN);       // Configura como entrada
    gpio_pull_up(BTN_MODE);                // Habilita resistencia interna pull-up (estado inactivo = 1)

    gpio_init(BTN_NEXT);                   // Repite para BTN_NEXT
    gpio_set_dir(BTN_NEXT, GPIO_IN);
    gpio_pull_up(BTN_NEXT);

    gpio_init(BTN_PREV);                   // Repite para BTN_PREV
    gpio_set_dir(BTN_PREV, GPIO_IN);
    gpio_pull_up(BTN_PREV);

    // Configurar interrupción para BTN_MODE
    gpio_set_irq_enabled(BTN_MODE, GPIO_IRQ_EDGE_FALL, true); // Habilita IRQ en flanco de bajada (botón presionado -> GND)
    gpio_set_irq_callback(gpio_callback); // Registra la función callback para IRQs GPIO (llamada global en este SDK)
    irq_set_enabled(IO_IRQ_BANK0, true);  // Habilita las interrupciones del banco de IO (necesario para que se ejecuten)

    string mensaje_usb = "";               // Buffer para recibir caracteres desde USB (comando acumulado)
    bool btn_next_prev = false;            // Estado previo del botón NEXT (para detectar flancos)
    bool btn_prev_prev = false;            // Estado previo del botón PREV (para detectar flancos)

    while (true) {                         // Bucle principal infinito
        // Leer comandos por USB
        int ch = getchar_timeout_us(1000); // Lee 1 carácter con timeout de 1000 us (1 ms)
        if (ch != PICO_ERROR_TIMEOUT) {   // Si se leyó un carácter válido
            if (ch == '\n' || ch == '\r') { // Si se presionó Enter
                if (!mensaje_usb.empty()) { // Si hay texto acumulado
                    procesar_comando(mensaje_usb, slice, chan); // Procesa el comando
                    mensaje_usb = "";      // Limpia el buffer
                    printf("> ");         // Muestra prompt
                }
            } else {
                mensaje_usb += (char)ch;   // Acumula carácter en el buffer
            }
        }

        // Modo Step (modo 3): mueve al siguiente/anterior cuando se detecta flanco de pulsación
        if (modo_actual == 3) {
            bool btn_next_actual = !gpio_get(BTN_NEXT); // gpio_get devuelve 1 cuando no presionado (pull-up),
                                                      // invertimos (!) para interpretar pulsado como true.
            bool btn_prev_actual = !gpio_get(BTN_PREV); // Igual para PREV

            if (btn_next_actual && !btn_next_prev) { // Detecta flanco de subida (presión nueva)
                if (posiciones.empty()) {
                    printf("Error no hay pos\n");
                } else {
                    if (indice_actual < (int)posiciones.size() - 1) { // Si no estamos al final, avanzamos
                        indice_actual++;
                    }
                    mover_servo(slice, chan, posiciones[indice_actual]); // Mueve servo a la nueva posición
                    printf("pos%d: %d\n", indice_actual + 1, posiciones[indice_actual]); // Imprime la pos (1-based)
                }
            }

            if (btn_prev_actual && !btn_prev_prev) { // Flanco para PREV
                if (posiciones.empty()) {
                    printf("Error no hay pos\n");
                } else {
                    if (indice_actual > 0) {     // Si no estamos al inicio, retrocedemos
                        indice_actual--;
                    }
                    mover_servo(slice, chan, posiciones[indice_actual]); // Mueve servo
                    printf("pos%d: %d\n", indice_actual + 1, posiciones[indice_actual]);
                }
            }

            btn_next_prev = btn_next_actual;     // Guarda estado actual para detectar flancos en la siguiente iteración
            btn_prev_prev = btn_prev_actual;
        }

        // Modo Continuo (modo 2): recorre todas las posiciones mientras ciclo_activo siga true
        if (modo_actual == 2 && ciclo_activo) {
            if (posiciones.empty()) {
                printf("Error no hay pos\n");
                sleep_ms(1500);                // Espera 1.5 s antes de intentar de nuevo (evita loop rápido de errores)
            } else {
                for (int i = 0; i < (int)posiciones.size(); i++) { // Recorre todas las posiciones
                    if (!ciclo_activo) break;   // Si ciclo_activo cambió (por pulsar mode), salimos
                    mover_servo(slice, chan, posiciones[i]); // Mover servo a la posición i
                    printf("pos%d: %d\n", i + 1, posiciones[i]);

                    absolute_time_t start_time = get_absolute_time(); // Marca tiempo de inicio
                    // Espera 1.5 s (1500000 us) de forma no bloqueante absoluta:
                    while (absolute_time_diff_us(start_time, get_absolute_time()) < 1500000) {
                        sleep_ms(100);         // Pausa pequeña dentro del loop para que el MCU no consuma 100% CPU
                        if (!ciclo_activo) break; // Permite salir rápido si se desactiva ciclo_activo
                    }
                }
            }
        }

        // Modo Entrenamiento (modo 1): permite mover con botones pero no iterar automáticamente
        if (modo_actual == 1 && !posiciones.empty()) {
            bool btn_next_actual = !gpio_get(BTN_NEXT); // Lectura botón NEXT (presionado -> true)
            bool btn_prev_actual = !gpio_get(BTN_PREV); // Lectura botón PREV

            if (btn_next_actual && !btn_next_prev) { // Flanco nuevo
                if (indice_actual < (int)posiciones.size() - 1) {
                    indice_actual++;
                }
                mover_servo(slice, chan, posiciones[indice_actual]); // Mueve servo a la posición actual
                printf("Servo a %d° (pos%d)\n", posiciones[indice_actual], indice_actual + 1);
            }

            if (btn_prev_actual && !btn_prev_prev) { // Flanco nuevo para PREV
                if (indice_actual > 0) {
                    indice_actual--;
                }
                mover_servo(slice, chan, posiciones[indice_actual]);
                printf("Servo a %d° (pos%d)\n", posiciones[indice_actual], indice_actual + 1);
            }

            btn_next_prev = btn_next_actual;       // Actualiza estados anteriores
            btn_prev_prev = btn_prev_actual;
        }

        sleep_ms(10);                             // Pequeña pausa para evitar busy-loop agresivo
    }

    return 0; // No se alcanza porque while(true) es infinito, pero es buena práctica devolver int
}
```