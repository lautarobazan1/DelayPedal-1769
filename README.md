# DelayPedal-1769
Firmware para LPC1769 que captura audio por ADC, procesa efectos de delay y saturacion en tiempo real, y reproduce por DAC usando GPDMA y control UART.

# Eco de Audio ADC-DAC con GPDMA en LPC1769

Proyecto final de sistemas embebidos orientado al procesamiento de audio en
tiempo real. El firmware toma una senal analogica por el ADC del LPC1769, la
procesa en bloques y la reproduce por el DAC usando transferencias GPDMA con
buffers ping-pong.

## Tabla de contenidos

- [Descripcion](#descripcion)
- [Caracteristicas](#caracteristicas)
- [Arquitectura general](#arquitectura-general)
- [Hardware utilizado](#hardware-utilizado)
- [Comandos UART](#comandos-uart)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Funcionamiento interno](#funcionamiento-interno)
- [Memoria AHB SRAM](#memoria-ahb-sram)
- [Compilacion e integracion](#compilacion-e-integracion)
- [Validacion recomendada](#validacion-recomendada)

## Descripcion

El objetivo del proyecto es implementar un efecto de eco configurable sobre una
placa con microcontrolador LPC1769. La entrada de audio se captura por `AD0.0`,
se procesa en la CPU y se envia a la salida analogica `AOUT`.

El sistema cuenta con dos modos de audio:

- **Delay**: mezcla la senal directa con una senal retardada entre 0 y 500 ms.
- **Saturacion**: aplica ganancia y clipping duro sobre la senal de entrada.

Los parametros principales se modifican en tiempo real mediante `UART0`, sin
detener la captura ni la reproduccion de audio.

## Caracteristicas

- Captura de audio por ADC0 canal 0.
- Salida de audio por DAC integrado.
- Frecuencia de muestreo de `22000 Hz`.
- Transferencias por GPDMA en canales 2 y 3.
- Buffers ping-pong para entrada y salida.
- Linea circular de retardo en memoria AHB SRAM.
- Control por UART a `115200 8N1`.
- Cambio de modo mediante boton por interrupcion externa `EINT0`.
- Procesamiento con aritmetica entera y ganancias en formato Q15.
- Rutinas de consulta para revisar estado desde el entorno de desarrollo.

## Arquitectura general

```text
Entrada analogica
      |
      v
ADC0.0 / P0.23
      |
      v
GPDMA CH2
      |
      v
Buffer ADC ping-pong en AHB SRAM
      |
      v
Procesamiento por CPU
      |
      v
Buffer DAC ping-pong en AHB SRAM
      |
      v
GPDMA CH3
      |
      v
DAC AOUT / P0.26
```

Timer0 genera el ritmo de muestreo del ADC. El DAC usa su contador interno para
pedir datos por DMA con la misma base temporal, manteniendo sincronizada la
captura con la reproduccion.

## Hardware utilizado

| Recurso | Funcion | Pin / canal |
| --- | --- | --- |
| ADC0 canal 0 | Entrada de audio | P0.23 / AD0.0 |
| DAC | Salida de audio | P0.26 / AOUT |
| UART0 TX | Mensajes hacia la terminal | P0.2 |
| UART0 RX | Recepcion de comandos | P0.3 |
| EINT0 | Boton de cambio de modo | P2.10 |
| Timer0 MAT0.1 | Disparo periodico del ADC | Interno |
| GPDMA CH2 | ADC a memoria | Interno |
| GPDMA CH3 | Memoria a DAC | Interno |

## Comandos UART

La UART se usa para cambiar parametros mientras el firmware esta corriendo. Los
comandos se envian por una terminal serial configurada en `115200 8N1` y se
confirman con Enter.

| Comando | Accion | Rango |
| --- | --- | --- |
| `T<ms>` | Cambia el tiempo de delay | 0 a 500 ms |
| `R<porcentaje>` | Cambia el repeat del eco | 0 a 90 % |
| `G<ganancia>` | Cambia el drive del saturador | 1x a 32x |
| `L<limite>` | Cambia el limite de clipping | 32 a 511 LSB |
| `D` | Activa modo delay | Sin valor |
| `S` | Activa modo saturacion | Sin valor |
| `?` | Consulta el estado actual | Sin valor |

Al iniciar, el sistema informa el modo actual y muestra la lista de comandos
disponibles. Hay una explicacion mas detallada del sistema UART en
[`README_UART.md`](README_UART.md).

## Estructura del repositorio

```text
ProyectoFinal/
|-- main.c.txt
|-- README.md
`-- README_UART.md
```

| Archivo | Contenido |
| --- | --- |
| `main.c.txt` | Codigo principal del firmware |
| `README.md` | Documentacion general del proyecto |
| `README_UART.md` | Explicacion especifica del sistema UART |

> Nota: si el IDE requiere extension `.c`, el archivo `main.c.txt` puede
> incorporarse como fuente C manteniendo el mismo contenido.

## Funcionamiento interno

### Inicializacion

`AudioEcho_Init()` configura el sistema completo:

1. Limpia las variables globales.
2. Define los divisores de reloj para Timer0, DAC, ADC y UART0.
3. Limpia la zona AHB SRAM usada por DMA.
4. Inicializa buffers de salida con media escala para evitar golpes de audio.
5. Prepara las listas enlazadas del GPDMA.
6. Configura UART0, boton, canales DMA, DAC, Timer0 y ADC.

### Arranque

`AudioEcho_Start()` habilita los canales DMA, activa las solicitudes DMA del DAC
y arranca Timer0. Desde ese momento el sistema comienza a capturar, procesar y
reproducir audio de forma continua.

### Procesamiento por bloque

`AudioEcho_ProcessBlock()` trabaja sobre bloques de `256` muestras:

1. Lee la muestra capturada por ADC.
2. La convierte a una muestra centrada en cero.
3. Aplica el modo activo: delay o saturacion.
4. Limita el resultado al rango del DAC.
5. Guarda la palabra lista para escribir en `LPC_DAC->DACR`.

### Interrupciones

| Interrupcion | Funcion |
| --- | --- |
| `DMA_IRQHandler()` | Detecta fin de bloque y errores de GPDMA |
| `UART0_IRQHandler()` | Recibe caracteres de comandos UART |
| `EINT0_IRQHandler()` | Cambia entre delay y saturacion con debounce |

El procesamiento del comando UART se realiza en el lazo principal para no
bloquear las interrupciones de audio.

## Memoria AHB SRAM

Los buffers DMA, las LLI del GPDMA y la linea de delay se ubican en la zona AHB
SRAM desde `0x2007C000`.

El proyecto reserva hasta `32 KiB` para:

- `adcLli[2]`
- `dacLli[2]`
- `adcDmaBuffer[2][256]`
- `dacDmaBuffer[2][256]`
- `delayLine[]`

El codigo incluye una comprobacion en compilacion para asegurar que la
estructura completa entra en la memoria disponible.

## Compilacion e integracion

Requisitos principales:

- Microcontrolador LPC1769.
- Toolchain C para ARM Cortex-M3.
- Librerias `LPC17xx`.
- Headers de ADC, DAC, GPDMA, Timer, UART, EXTI y CLKPWR.
- Script de linker compatible con el mapa de memoria usado.

Puntos a revisar en el entorno de desarrollo:

- Agregar `main.c.txt` como archivo fuente C o renombrarlo segun la
  configuracion del IDE.
- Verificar que los headers `LPC17xx.h` y `lpc17xx_*.h` esten disponibles.
- Confirmar que la region AHB SRAM usada por DMA no se solape con pila, heap u
  otras variables.
- Asegurar que las rutinas `DMA_IRQHandler`, `UART0_IRQHandler` y
  `EINT0_IRQHandler` queden vinculadas a la tabla de vectores.

## Validacion recomendada

1. Conectar una senal analogica adecuada al pin `P0.23 / AD0.0`.
2. Conectar la salida `P0.26 / AOUT` a un osciloscopio, filtro o etapa de audio.
3. Abrir una terminal serial en `UART0` a `115200 8N1`.
4. Encender el sistema y verificar que se informe el modo inicial.
5. Enviar comandos `T`, `R`, `G`, `L`, `D`, `S` y `?`.
6. Confirmar que `AudioEcho_GetDmaErrorFlags()` permanezca en cero.
7. Confirmar que `AudioEcho_GetProcessedBlocks()` aumente de forma continua.

## Resumen tecnico

El firmware usa perifericos del LPC1769 para mantener el flujo de audio en
hardware y dejar a la CPU solo el procesamiento de muestras. El ADC y el DAC se
sincronizan por temporizacion, el DMA mueve los bloques de datos y la CPU aplica
el efecto seleccionado. La UART permite ajustar parametros en tiempo real y el
boton externo permite alternar el modo de procesamiento.
