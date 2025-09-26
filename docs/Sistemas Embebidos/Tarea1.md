# 🤖 Tarea 1: Tabla comparativa
> Garcia Cortez Juan David ·  Sistemas Embebidos 1  ·  25/08/2025.

---


No.| Microcontrolador | Periféricos | Memoria     |Ecosistema |Costos   |Arquitectura|Velocidad de Trabajo|
--:|-----------------:|:-----------:|:-----------:|:---------:|:-------:|:----------:|:------------------:|
01 | ESP32 (Espressif) |45 GPIO programables, SPI, I2C, I2S, UART, PWM, ADC, DAC, RMT, SD/MMC, CAN, sensores táctiles, Hall sensor|512 KB de SRAM interna, 448 KB de ROM, memoria Flash externa|Compatible con Arduino, ESP-IDF, MicroPython, FreeRTOS|$120 - $200 MXN|Xtensa LX6 (32 bits, dual core)|Hasta 240 MHz|
02 | Raspberry Pi Pico (RP2040) |26 GPIO, SPI, I2C, UART, PWM, ADC, temporizadores|264 KB de SRAM, memoria Flash externa (2 MB típica)|Compatible con C/C++, MicroPython, CircuitPython|$100 - $150 MXN|ARM Cortex-M0+ (32 bits, dual core)|Hasta 133 MHz|
03 | STM32F103C8T6  |37 GPIO, SPI, I2C, USART, PWM, ADC, DAC, temporizadores, CAN, USB|20 KB SRAM, 64 KB Flash|Compatible con STM32Cube, Arduino, Mbed, PlatformIO|$60 - $120 MXN|ARM Cortex-M3 (32 bits, single core)|Hasta 72 MHz|
04 | ATmega328 |23 GPIO, SPI, I2C, UART, PWM, ADC, temporizadores|2 KB SRAM, 32 KB Flash, 1 KB EEPROM|Compatible con Arduino, AVR-GCC|$50 - $90 MXN|AVR (8 bits, single core)|Hasta 20 MHz|


> Periféricos: Módulos Integrados que permiten interactuar con el mundo físico.

## Conclusiones
*   Para un proyecto como una conosla de mezclas de música, la ESP32, resulta como la mejor opción, dado a su versatilidad y conectividad WiFi/Bluetooth integrado, facilitando la integración y comunicación con otros dispositivos. La Raspberry Pi Pico también es una opción flexible y potente, especialmente si se requiere procesamiento en paralelo o facilidad de programación con MicroPython. Seguido de, tenemos la STM32F103C8T6 que es ideal para apliacaciones más industriales,pero no cuenta con una conectividad inalámbrica, y su programación puede llegar a ser más compleja. Finalmente, el ATmega328P puede quedarse corto para este tipo de proyectos, ya que se dificulta al manejar múltiples dispositivos.