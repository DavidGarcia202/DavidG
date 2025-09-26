# 🤖 Tarea 2: Outputs Básicos
> Garcia Cortez Juan David · Arai Erazo Sumie ·  Sistemas Embebidos 1  ·  01/09/2025.

---


## Contador Binario 4 bits
* En 4 leds debe mostrarse cada segundo de la presentación binaria del 0 al 15

### Código
```bash
#include "pico/stdlib.h"

#define Led1 6   // (Less Significant Bit)
#define Led2 7   // 
#define Led3 8   // 
#define Led4 9   // (Most Significant Bit)

int main() {
    // Máscara con los 4 LEDs
    const uint32_t Mascara = (1u << Led1 | 1u << Led2 | 1u << Led3 | 1u << Led4); //Aqui se define que leds vamos a ocupar

    gpio_init_mask(Mascara); // Inicializamos los pines
    gpio_set_dir_masked(Mascara, Mascara); // Definimos todos como salida, utilizando 1u que se mostro arriba

    int contador = 0;  // Iniciamos el contador en 0

    while (true) {
        // Poner en LEDs el valor del contador
        gpio_put_masked(Mascara, (contador << Led1));

        sleep_ms(1000); // 1 segundo

        contador++;
        if (contador > 15) { // Reinicia al llegar a 15
            contador = 0;
        }
    }
}
```
### Video
<iframe width="560" height="315" src="https://www.youtube.com/embed/cVW_jtcMMA4" frameborder="0" allowfullscreen></iframe>


## Barrido de leds
* Correr un “1” por cinco LEDs P0..P3 y regresar (0→1→2→3→2→1…)

### Código
```bash
#include "pico/stdlib.h" // Para usar las funciones de GPIO
#include "hardware/gpio.h"        // Para usar las funciones de GPIO


#define Led1 6
#define Led2 7
#define Led3 8
#define Led4 9
#define Led5 10

int main()
{
    const uint32_t Mascara = (1u << Led1 | 1u << Led2 | 1u << Led3 | 1u << Led4 | 1u << Led5);

    gpio_init_mask(Mascara);                                                                                                                 // Inicializa los pines
    gpio_set_dir_masked(Mascara, (1u << Led1 | 1u << Led2 | 1u << Led3 | 1u << Led4 | 1u << Led5)); // Configura los pines como salida
    

    int posicion = 6;

    int direccion = 1; // 1 para derecha, -1 para izquierda

    while (true)
    {
        gpio_put_masked(Mascara, (1u << posicion)); // Enciende el LED en la posición actual
        sleep_ms(200);                             // Pausa de 200 ms

        posicion += direccion; // Actualiza la posición
        if (posicion == Led5) direccion = -1;
        else if (posicion == Led1) direccion = 1;

    }
}
```
### Video
<iframe width="560" height="315" src="https://www.youtube.com/embed/AlaYZ4KBXfA" frameborder="0" allowfullscreen></iframe>

## Secuencia en codigo Gray
* Mostrar la secuencia del 0 al 15 en forma de código Gray

### Código
```bash
#include "pico/stdlib.h"

#define Led1 6   // (LSB)
#define Led2 7   
#define Led3 8   
#define Led4 9   // (MSB)

int main() {
    const uint32_t Mascara = (1u << Led1 | 1u << Led2 | 1u << Led3 | 1u << Led4);

    gpio_init_mask(Mascara);
    gpio_set_dir_masked(Mascara, Mascara);

    int contador = 0;

    while (true) {
        // Convertir de binario a Gray
        int gray = contador ^ (contador >> 1);

        // Mandar a los LEDs
        gpio_put_masked(Mascara, (gray << Led1));

        sleep_ms(1000);

        contador++;
        if (contador > 15) {
            contador = 0;
        }
    }
}

```
### Video
<iframe width="560" height="315" src="https://www.youtube.com/embed/pw941NrEIGA" frameborder="0" allowfullscreen></iframe>
