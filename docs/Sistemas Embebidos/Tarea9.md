# 📚 TAREA 9

## 1) Resumen

- *Nombre del proyecto:* ADC Luxometro y ADC Servo
- *Equipo / Autor(es):* Juan David García Cortez y Sumie Arai Erazo  
- *Curso / Asignatura:* Sistemas embebidos 1  
- *Fecha:* 22/10/25 

## 2) Objetivos

- *General:* Aprender las utilidades del ADC


## 3) Código del Luxómetro
```bash
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/adc.h"
 
int main() {
    stdio_init_all();
 
    // Inicializar ADC
    adc_init();
 
    // Elegir pin GP26 -> ADC0
    adc_gpio_init(26);
    adc_select_input(0);
 
    while (true) {
        uint16_t valor_adc = adc_read(); // Valor de 0 a 4095 (12 bits)
       
        // Calcular porcentaje (0% en 10, 100% en 4095)
        float porcentaje = 0.0f;
        if (valor_adc > 10) {
            porcentaje = ((valor_adc - 10) * 100.0f) / (4095.0f - 10.0f);
            if (porcentaje > 100.0f)
                porcentaje = 100.0f; // Limitar por seguridad
        }
 
        printf("Porcentaje: %.2f%%\n", porcentaje);
        sleep_ms(200);
    }
}
```
## Vídeo Luxómetro 

<div class="iframe-container">
<iframe width="560" height="315" src="https://www.youtube.com/embed/1XyUdpuH5Gg?si=zB4LDMP26-idrnsw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

## 4) Código del Servo 
```bash
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/adc.h"
#include "hardware/pwm.h"

#define SERVO_PIN 15
#define ADC_PIN 26

// Rango total del ADC
#define ADC_MIN 0
#define ADC_MAX 4095

// Límites seguros del servo (en microsegundos)
#define PULSO_MIN 500.0f
#define PULSO_MAX 2500.0f

// Promedio para suavizar lectura
#define NUM_MUESTRAS 1

// === Función para leer ADC suavizado ===
uint16_t leer_adc_promedio() {
    uint32_t suma = 0;
    for (int i = 0; i < NUM_MUESTRAS; i++) {
        suma += adc_read();
        sleep_us(1000);
    }
    return (uint16_t)(suma / NUM_MUESTRAS);
}

int main() {
    stdio_init_all();

    // Inicializar ADC
    adc_init();
    adc_gpio_init(ADC_PIN);
    adc_select_input(0);

    // Configurar PWM para el servo
    gpio_set_function(SERVO_PIN, GPIO_FUNC_PWM);
    uint slice_num = pwm_gpio_to_slice_num(SERVO_PIN);
    pwm_set_wrap(slice_num, 24999);      // 50 Hz
    pwm_set_clkdiv(slice_num, 100.0f);
    pwm_set_enabled(slice_num, true);

    sleep_ms(500); // estabiliza el ADC

    while (true) {
        uint16_t valor_adc = leer_adc_promedio();

        // Aseguramos que esté dentro del rango válido (0–4095)
        if (valor_adc > 4095) valor_adc = 4095;

        // Calcular porcentaje (0–1)
        float porcentaje = (float)valor_adc / (float)ADC_MAX;

        // Mapear a pulso y ángulo
        float pulso_us = PULSO_MIN + porcentaje * (PULSO_MAX - PULSO_MIN);
        uint16_t nivel_pwm = (uint16_t)((pulso_us / 20000.0f) * 24999);
        float angulo = porcentaje * 180.0f;

        // Enviar al servo
        pwm_set_gpio_level(SERVO_PIN, nivel_pwm);

        // Mostrar datos
        printf("ADC: %u | Pulso: %.1f us | Ángulo: %.1f°\n",
               valor_adc, pulso_us, angulo);

        sleep_ms(80);
    }
}
```
## Video del servo 

<div class="iframe-container">
<iframe width="560" height="315" src="https://www.youtube.com/embed/zImjOQUQME8?si=KFIuV3euc9Le2WlD" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

## Esquemáticos

<div class="iframe-container">
<iframe 
  src="https://wokwi.com/projects/446817926756467713" 
  width="800" 
  height="600" 
  allowfullscreen>
</iframe>
</div>

_Nota: El led será el fotoresistor ya que no existe el componente en wokwi_

<div class="iframe-container">
<iframe 
  src="https://wokwi.com/projects/446818621127736321" 
  width="800" 
  height="600" 
  allowfullscreen>
</iframe>
</div>