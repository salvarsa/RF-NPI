# RF-NPI — Diagrama de conexiones (pin a pin)

Documento de referencia de hardware para el proyecto **RF-NPI** (basado en ESP32).
Todas las asignaciones de pines fueron verificadas contra el firmware en
[`src/config.h`](src/config.h), [`src/config.cpp`](src/config.cpp) y
[`src/main.cpp`](src/main.cpp).

> **Placa base:** ESP32 DevKit (WROOM-32) — numeración GPIO.

---

## 1. Tabla resumen de GPIO

| GPIO | Función                         | Componente        | Definido en código |
|------|---------------------------------|-------------------|--------------------|
| 2    | CSN                             | Radio C (nRF24)   | `NRF_CSN_PIN_C` ⚠️ strapping |
| 4    | CSN                             | Radio B (nRF24)   | `NRF_CSN_PIN_B`    |
| 5    | CE                              | Radio A (nRF24)   | `NRF_CE_PIN_A` ⚠️ strapping  |
| 14   | Data (DIN)                      | NeoPixel WS2812   | *(comentado / deshabilitado)* |
| 15   | CE                              | Radio C (nRF24)   | `NRF_CE_PIN_C` ⚠️ strapping  |
| 16   | CE                              | Radio B (nRF24)   | `NRF_CE_PIN_B`    |
| 17   | CSN                             | Radio A (nRF24)   | `NRF_CSN_PIN_A`    |
| 18   | SPI SCK (reloj)                 | Bus SPI compartido| *(default VSPI)*   |
| 19   | SPI MISO (datos de regreso)     | Bus SPI compartido| *(default VSPI)*   |
| 21   | I2C SDA                         | Pantalla OLED     | *(default Wire)*   |
| 22   | I2C SCL                         | Pantalla OLED     | *(default Wire)*   |
| 23   | SPI MOSI (datos de salida)      | Bus SPI compartido| *(default VSPI)*   |
| 25   | Botón Derecho                   | Pulsador          | `PIN_BTN_R`        |
| 26   | Botón Selección                 | Pulsador          | `PIN_BTN_S`        |
| 27   | Botón Izquierdo                 | Pulsador          | `PIN_BTN_L`        |

> ✅ Esta tabla **confirma** la lista de conexiones proporcionada: todos los pines
> coinciden con el firmware.

---

## 2. Radios nRF24L01+ (x3) — Bus SPI compartido

Las tres radios comparten el mismo bus SPI (SCK / MOSI / MISO) y se diferencian
mediante sus pines **CE** y **CSN** individuales.

### Pines compartidos (bus SPI — VSPI del ESP32)

| Señal nRF24 | GPIO ESP32 | Notas                          |
|-------------|-----------|---------------------------------|
| SCK         | 18        | Reloj SPI compartido            |
| MOSI        | 23        | Datos hacia las radios          |
| MISO        | 19        | Datos desde las radios          |
| IRQ         | — (sin usar) | No conectado en el firmware  |
| VCC         | **3.3 V** | ⚠️ **NUNCA 5 V** (daña el módulo) |
| GND         | GND       | Común                           |

### Pines individuales (control por radio)

| Radio   | CE  (GPIO) | CSN (GPIO) | Objeto en código |
|---------|-----------|------------|------------------|
| Radio A | 5         | 17         | `RadioA`         |
| Radio B | 16        | 4          | `RadioB`         |
| Radio C | 15        | 2          | `RadioC`         |

**Recomendación eléctrica:** colocar un **capacitor de desacople de 10 µF**
(electrolítico o cerámico) entre **VCC y GND de cada módulo nRF24**, lo más cerca
posible del pin. Los nRF24 generan picos de corriente al transmitir y sin este
capacitor son propensos a fallos intermitentes o a no ser detectados por
`radio.begin()`.

---

## 3. Pantalla OLED SSD1306 (128×64, I2C)

Controlada con la librería U8g2 en modo hardware I2C
(`U8G2_SSD1306_128X64_NONAME_F_HW_I2C`). Usa el bus I2C por defecto del ESP32.

| Señal OLED | GPIO ESP32 |
|------------|-----------|
| SDA        | 21        |
| SCL        | 22        |
| VCC        | 3.3 V     |
| GND        | GND       |

> El firmware inicializa I2C con `Wire.begin()` (sin argumentos), por lo que toma
> los pines por defecto **21 (SDA)** y **22 (SCL)**. El bus corre a 400 kHz
> (`Wire.setClock(400000)`).

---

## 4. Botones (x3)

Configurados como `INPUT_PULLUP` con interrupciones por flanco de bajada
(`FALLING`). Cada botón conecta su GPIO a **GND** al presionarse.

| Botón      | GPIO ESP32 | Constante   | Conexión                 |
|------------|-----------|-------------|--------------------------|
| Izquierdo  | 27        | `PIN_BTN_L` | GPIO 27 ── botón ── GND  |
| Derecho    | 25        | `PIN_BTN_R` | GPIO 25 ── botón ── GND  |
| Selección  | 26        | `PIN_BTN_S` | GPIO 26 ── botón ── GND  |

> No se necesitan resistencias externas: se usan los pull-ups internos del ESP32.

---

## 5. NeoPixel WS2812 (deshabilitado)

Un LED direccionable en **GPIO 14** (`Data / DIN`). El código está actualmente
**comentado** ([`src/neopixel.cpp`](src/neopixel.cpp),
[`src/config.cpp`](src/config.cpp)) porque no se usa este hardware. Si en el futuro
se reactiva, la conexión sería:

| Señal NeoPixel | GPIO ESP32 |
|----------------|-----------|
| DIN (Data)     | 14        |
| VCC            | 5 V (o 3.3 V según el módulo) |
| GND            | GND       |

---

## 6. Alimentación — TP4056 + interruptor físico

El dispositivo se alimenta con una celda Li-ion/LiPo a través de un módulo de carga
**TP4056**, con un **interruptor de 2 posiciones (3 pines)** en serie en la línea de
salida para encender/apagar físicamente el equipo.

```
Batería (+) ──► TP4056 [B+]
Batería (−) ──► TP4056 [B−]

TP4056 [OUT+] ──► [pin lateral] ┐
                                 SWITCH (SPDT)
ESP32 [VIN/5V] ◄── [pin central/común] ┘

TP4056 [OUT−] ─────────────────────► ESP32 [GND]
```

| Conexión                    | Detalle                                        |
|-----------------------------|------------------------------------------------|
| TP4056 `OUT+` → Switch      | A un pin lateral del interruptor               |
| Switch (común) → ESP32 `VIN`| El pin central del interruptor alimenta VIN    |
| TP4056 `OUT-` → ESP32 `GND` | Tierra común (no se interrumpe)                |
| Tercer pin del switch       | Sin usar                                       |

### ⚠️ Importante sobre el pin de alimentación

- El interruptor debe ir al pin **VIN / 5V** del ESP32, **NO al pin 3V3**.
- El pin `3V3` de las DevKit es la **salida** del regulador interno de la placa.
  Inyectar ahí el voltaje de batería (3.0–4.2 V, sin regular) **puede dañar el
  ESP32** (máximo absoluto ≈ 3.6 V).
- Al ir a `VIN`, la batería pasa por el regulador de la placa antes de llegar al
  chip. Es el camino seguro.

### Funcionamiento del interruptor

- **OFF** → VIN sin tensión → ESP32 completamente apagado (no se requiere código).
- **ON**  → VIN recibe la batería → ESP32 arranca y ejecuta `setup()` normalmente.

Es control **puramente físico**: no hay lógica de encendido/apagado en el firmware.

### Nota de autonomía

Alimentando por `VIN` desde una sola celda LiPo, cuando la batería baje de
~3.5 V el regulador de la placa (dropout ≈ 1 V) puede empezar a resetear el ESP32
antes de que la celda esté realmente agotada. Si aparecen reinicios con la batería
a media carga, ésta es la causa (tema de autonomía, no del interruptor).

---

## 7. Notas sobre pines de arranque (strapping pins) ⚠️

Los GPIO **2, 5 y 15** son *strapping pins* del ESP32 (determinan el modo de
arranque). En este diseño se usan como salidas para CE/CSN de las radios, lo cual
**funciona correctamente después del boot**, pero conviene tener en cuenta:

- Si algún módulo o cableado fuerza estos pines a un nivel lógico incorrecto
  **durante el encendido**, el ESP32 podría no arrancar o entrar en modo de descarga.
- Los módulos nRF24 en estado normal no interfieren, pero si observas fallos de
  arranque intermitentes, estos pines son el primer lugar a revisar.

| GPIO | Nivel esperado en boot | Uso aquí   |
|------|------------------------|------------|
| 2    | debe estar bajo/flotante| CSN Radio C |
| 5    | debe estar alto        | CE Radio A |
| 15   | debe estar alto        | CE Radio C |

---

## 8. Resumen de tensiones

| Componente     | Alimentación        |
|----------------|---------------------|
| ESP32          | VIN (batería vía regulador) / USB 5 V |
| nRF24L01+ (x3) | **3.3 V** (obligatorio) |
| OLED SSD1306   | 3.3 V               |
| NeoPixel WS2812| 5 V / 3.3 V (deshabilitado) |
| TP4056         | Batería Li-ion/LiPo 1S (3.0–4.2 V) |
