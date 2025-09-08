# 🤖 Tarea 4: Pong
######Garcia Cortez Juan David · Arai Erazo Sumie ·  Sistemas Embebidos 1  ·  01/09/2025

## Programar un mini-Pong con 5 LEDs en línea y 2 botones usando interrupciones (ISR) para registrar el “golpe” del jugador exactamente cuando la “pelota” (un LED encendido) llega al extremo de su lado.
*  Reglas del juego
Pelota: es un único LED encendido que se mueve automáticamente de un extremo al otro (L1→L5→L1…) a un ritmo fijo.

Golpe con ISR: cada botón genera una interrupción.

El BTN_L solo cuenta si, en el instante de la ISR, la pelota está en L1.

El BTN_R solo cuenta si, en el instante de la ISR, la pelota está en L5.

Si coincide, la pelota rebota: invierte su dirección.

Si no coincide (la pelota no está en el último LED de ese lado), el botón se ignora.

Fallo y punto: si la pelota alcanza L1 y no hubo golpe válido del lado izquierdo en ese momento, anota el jugador derecho. Análogamente, si alcanza L5 sin golpe válido, anota el jugador izquierdo.

Indicador de punto: al anotar, se parpadea el LED de punto 3 veces del jugador que metió el punto .

Reinicio tras punto: después del parpadeo, la pelota se reinicia en el centro (L3) y comienza a moverse hacia el jugador que metió el punto.

Inicio del juego: al encender, la pelota inicia en L3 y no se mueve hasta que se presione un boton y debera moverse a la direccion opuesta del boton presionado.
### Código
```bash
// tarea4.c

#include "pico/stdlib.h"
#include "hardware/gpio.h"

// Pines
#define P1 4
#define P2 5
#define WIN1 6
#define LED1 7
#define LED2 8
#define LED3 9
#define LED4 10
#define LED5 11
#define WIN2 12

// Flags de interrupción
volatile bool flag_p1 = false;
volatile bool flag_p2 = false;

// Función para parpadear LED ganador
void parpadear_led(uint led) {
    for (int i = 0; i < 3; i++) {
        gpio_put(led, 1);
        sleep_ms(200);
        gpio_put(led, 0);
        sleep_ms(200);
    }
}

// Callback de interrupción
void gpio_callback(uint gpio, uint32_t events) {
    if (events & GPIO_IRQ_EDGE_RISE) {
        if (gpio == P1) flag_p1 = true;
        else if (gpio == P2) flag_p2 = true;
    }
}

int main() {
    // Inicializar LEDs
    const uint32_t LED_MASK = (1u << WIN1) | (1u << LED1) | (1u << LED2) | (1u << LED3) |
                               (1u << LED4) | (1u << LED5) | (1u << WIN2);
    gpio_init_mask(LED_MASK);
    gpio_set_dir_masked(LED_MASK, LED_MASK);

    // Inicializar botones
    gpio_init(P1); gpio_set_dir(P1, GPIO_IN); gpio_pull_down(P1); //tambien tiene pulldown externo
    gpio_init(P2); gpio_set_dir(P2, GPIO_IN); gpio_pull_down(P2);
    gpio_set_irq_enabled_with_callback(P1, GPIO_IRQ_EDGE_RISE, true, &gpio_callback);
    gpio_set_irq_enabled(P2, GPIO_IRQ_EDGE_RISE, true);

    int ultima_direccion = 1; // 1: derecha (jugador 2), -1: izquierda (jugador 1)
    bool primer_jugada = true; // <--- NUEVO

    while (true) {
        gpio_put_masked(LED_MASK, 0); // Apagar todos los LEDs
        gpio_put(LED3, 1); // Encender LED3 del medio

        // Esperar a que se presione cualquier botón antes de iniciar la secuencia
        while (gpio_get(P1) == 0 && gpio_get(P2) == 0) {
            sleep_ms(10);
        }

        // Apagar LED3 antes de iniciar la secuencia
        gpio_put(LED3, 0);

        int direccion_inicial;
        if (primer_jugada) {
            // Primera jugada: dirección hacia el jugador que NO presionó el botón
            if (gpio_get(P1) == 1) {
                direccion_inicial = 1;  // P1 presionó, va hacia P2
            } else if (gpio_get(P2) == 1) {
                direccion_inicial = -1; // P2 presionó, va hacia P1
            } else {
                direccion_inicial = ultima_direccion; // Por si acaso
            }
            primer_jugada = false;
        } else {
            // Siguientes jugadas: dirección hacia el ganador anterior
            direccion_inicial = ultima_direccion;
        }

        int posicion = LED3;
        int direccion = direccion_inicial;
        flag_p1 = false;
        flag_p2 = false;

        int prev_p1 = 1, prev_p2 = 1;

        while (1) {
            gpio_put_masked(LED_MASK, 0);

            if (posicion >= LED1 && posicion <= LED5) {
                gpio_put(posicion, 1);
            }
            sleep_ms(400);

            int curr_p1 = gpio_get(P1);
            int curr_p2 = gpio_get(P2);

            if (posicion == LED1 && prev_p1 == 1 && curr_p1 == 0) {
                direccion = 1;
            } else if (posicion == LED5 && prev_p2 == 1 && curr_p2 == 0) {
                direccion = -1;
            }

            prev_p1 = curr_p1;
            prev_p2 = curr_p2;

            // Verificar victoria
            if (posicion == WIN1) {
                gpio_put_masked(LED_MASK, 0);
                parpadear_led(WIN2);
                ultima_direccion = 1; // Ahora la siguiente ronda va hacia jugador 2
                break;
            } else if (posicion == WIN2) {
                gpio_put_masked(LED_MASK, 0);
                parpadear_led(WIN1);
                ultima_direccion = -1; // Ahora la siguiente ronda va hacia jugador 1
                break;
            }

            posicion += direccion;
        }
        sleep_ms(500); // Espera antes de reiniciar el juego
    }
}
```

### Esquemático
![Esquemático](imgs/TAREA4.jpeg)

### Video
<iframe width="560" height="315" src="https://www.youtube.com/embed/jyWKDoAtEeA" frameborder="0" allowfullscreen></iframe>


## Selector cíclico de 5 LEDs con avance/retroceso
* Mantén un único LED encendido entre LED0..LED3. Un botón AVANZA (0→1→2→3→4→0) y otro RETROCEDE (0→4→3→2→1→0). Un push = un paso (antirrebote por flanco: si dejas presionado no repite). En el video demuestra en ambos sentidos.

### Código
```bash
#include "pico/stdlib.h" // Para usar las funciones de GPIO
// #include "hardware/gpio.h"        // Para usar las funciones de GPIO

#define BotonX 4
#define BotonY 5
#define Led1 6
#define Led2 7
#define Led3 8
#define Led4 9
#define Led5 10

int main()
{
    const uint32_t Mascara = (1u << BotonX | 1u << BotonY | 1u << Led1 | 1u << Led2 | 1u << Led3 | 1u << Led4 | 1u << Led5);

    gpio_init_mask(Mascara);                                                                                                                 // Inicializa los pines
    gpio_set_dir_masked(Mascara, (0u << BotonX | 0u << BotonY | 1u << Led1 | 1u << Led2 | 1u << Led3 | 1u << Led4 | 1u << Led5)); // Configura los pines como salida
    gpio_pull_up(BotonX);                                                                                                                    // Activa la resistencia pull-up interna del pin 4
    gpio_pull_up(BotonY);

    int posicion = 6;
    bool presionado = false;

    while (true)
    {
        int Boton_Izq = !gpio_get(BotonX);
        int Boton_Der = !gpio_get(BotonY);

        if (Boton_Izq == 1 && presionado == false)
        {
            presionado = true;
            if (posicion == 6)
                posicion = 11;
            posicion--;
        }
        if (Boton_Der == 1 && presionado == false)
        {
            presionado = true;
            if (posicion == 10)
                posicion = 5;
            posicion++;
        }
        if (Boton_Izq == 0 && Boton_Der == 0)
            presionado = false;
        gpio_put_masked(Mascara, (1u << posicion)); // Enciende el LED en la posición actual
        sleep_ms(10);                              // Pausa de 100 ms
    }
}
```

### Esquemático
![Esquemático](imgs/ESQtarea3%20(1).png)

### Video
<iframe width="560" height="315" src="https://www.youtube.com/embed/kTSAZdiHBfg" frameborder="0" allowfullscreen></iframe>
