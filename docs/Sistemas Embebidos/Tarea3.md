# 🤖 Tarea 3: Inputs
> Garcia Cortez Juan David · Arai Erazo Sumie ·  Sistemas Embebidos 1  ·  01/09/2025.

---

## Compuertas básicas AND / OR / XOR con 2 botones
*  Con dos botones A y B (pull-up; presionado=0) enciende tres LEDs que muestren en paralelo los resultados de AND, OR y XOR. En el video muestra las 4 combinaciones (00, 01, 10, 11).

### Código
```bash
#include "pico/stdlib.h" // Para usar las funciones de GPIO
// #include "hardware/gpio.h"        // Para usar las funciones de GPIO

#define BotonX 4
#define BotonY 5
#define LedX 6
#define LedY 7
#define LedZ_AND 8
#define LedZ_OR 9
#define LedZ_XOR 10

int main()
{
    const uint32_t Mascara = (1u << BotonX | 1u << BotonY | 1u << LedX | 1u << LedY | 1u << LedZ_AND | 1u << LedZ_OR | 1u << LedZ_XOR);

    gpio_init_mask(Mascara);                                                                                                                 // Inicializa los pines
    gpio_set_dir_masked(Mascara, (0u << BotonX | 0u << BotonY | 1u << LedX | 1u << LedY | 1u << LedZ_AND | 1u << LedZ_OR | 1u << LedZ_XOR)); // Configura los pines como salida
    gpio_pull_up(BotonX);                                                                                                                    // Activa la resistencia pull-up interna del pin 4
    gpio_pull_up(BotonY);

    while (true)
    {
        int Entrada_X = !gpio_get(BotonX);
        int Entrada_Y = !gpio_get(BotonY);
        int Salida_AND, Salida_OR, Salida_XOR;

        Salida_AND = Entrada_X & Entrada_Y; // Pin 8 = Pin 4 AND Pin 5
        Salida_OR = Entrada_X | Entrada_Y;  // Pin 9 = Pin 4 OR Pin 5
        Salida_XOR = Entrada_X ^ Entrada_Y; // Pin 10 = Pin 4 XOR Pin 5

        gpio_put_masked(Mascara, (1 << 11) | (Entrada_X << LedX) | (Entrada_Y << LedY) | (Salida_AND << LedZ_AND) | (Salida_OR << LedZ_OR) | (Salida_XOR << LedZ_XOR));
        sleep_ms(100); // Pausa de 100 ms
    }
}
```

### Esquemático
![Esquemático](imgs/ESQtarea3%20(1).png)

### Video
<iframe width="560" height="315" src="https://www.youtube.com/embed/v0AorhLYsz0" frameborder="0" allowfullscreen></iframe>


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
