# Arquitectura de Sincronización: ZCD → PLL → NCO

> **DOCUMENTO DE DISEÑO PARA EVALUAR.** Amplía la arquitectura del zero-cross para
> usarlo no solo como sensor de presencia de red, sino como referencia de
> SINCRONIZACIÓN: igualar la frecuencia y la fase del inversor con la red, usando
> el acumulador de fase (NCO) que ya existe en el código, cerrado en un lazo de
> enganche de fase (PLL).
>
> Se conecta a las variables reales del código (`phase_acc`, `phase_increment`,
> `PHASE_INC`), verificadas en `inversor_acumulador_fase_LCD.ino`.
>
> **Este documento sube la barra de validación. La sincronización con la red
> maneja la interacción entre dos fuentes de AC y, en la reconexión, puede causar
> eventos destructivos si se hace mal. Debe validarse con instrumentos de potencia,
> no solo lógicos, y revisarse con competencia eléctrica antes de construirse.**

---

## 1. Qué problema resuelve la sincronización

El inversor genera su onda por su lado. La red tiene la suya. Si las dos no están
de acuerdo —distinta frecuencia, o misma frecuencia pero desfasadas— juntarlas en
una transferencia causa un **transitorio**: salto de voltaje, golpe de corriente,
estrés en la carga y el equipo.

Analogía: empujar un columpio. Si empujas en el momento correcto del vaivén, sumas
suave. Si empujas a destiempo, chocas. Sincronizar es empujar en el momento justo.

La sincronización tiene dos niveles, de menos a más exigente:
- **Igualar frecuencia:** que el inversor corra a los Hz REALES de la red en ese
  momento (no 60.000 teóricos, sino los 59.97 o 60.04 que la red tenga).
- **Alinear fase:** que el cruce por cero del inversor coincida con el de la red.
  Cuando ambas cruzan cero al mismo tiempo, están en fase, y la transferencia es
  suave.

---

## 2. Cómo el NCO existente ya está preparado para esto

El código ya tiene el motor que la sincronización necesita. Del código real:

```
  #define ACC_BITS     32                 // acumulador de fase de 32 bits
  #define PHASE_INC(f)  (...)             // convierte frecuencia -> incremento
  volatile uint32_t phase_acc       = 0;  // acumulador (la "posicion" en la onda)
  volatile uint32_t phase_increment = 0;  // cuanto avanza por paso (la frecuencia)
  // en la ISR:  phase_acc += phase_increment;  idx = phase_acc >> ACC_SHIFT;
```

Esto es un **NCO (oscilador controlado numéricamente)**, y es exactamente la pieza
que un PLL controla:
- `phase_increment` fija la **frecuencia**. Cambiarlo = cambiar la frecuencia del
  inversor con precisión fina. Esto es la perilla que el PLL ajusta para IGUALAR
  la frecuencia de la red.
- `phase_acc` es la **posición instantánea** en la onda (la fase). Compararla con
  el cruce de la red dice si vamos adelantados o atrasados. Esto es lo que el PLL
  mide para ALINEAR la fase.

**El NCO se dejó listo para PLL desde el diseño inicial.** El ZCD es la entrada que
faltaba. Esta arquitectura cierra ese lazo.

---

## 3. El lazo PLL (enganche de fase) — cómo funciona

Un PLL es un lazo de control que ajusta el NCO hasta que el inversor queda
enganchado a la red (misma frecuencia, misma fase). Los tres elementos:

```
   Red ──[ZCD aislado]──> "la red cruzo cero AQUI" (timestamp en la ISR)
                                      │
                                      ▼
                       DETECTOR DE FASE (en el loop)
            compara: ¿donde esta phase_acc cuando la red cruza cero?
                       ¿adelantados o atrasados? ¿cuanto?
                                      │
                                      ▼
                            FILTRO DE LAZO (suave)
              convierte el error de fase en un ajuste pequeno
              y gradual (nunca brusco) del phase_increment
                                      │
                                      ▼
                    NCO (ya existente): phase_increment ajustado
              el inversor acelera o frena un pelin hasta enganchar
```

**Principios de diseño del lazo (críticos):**
- **El ajuste es suave y gradual, nunca brusco.** El PLL corrige poco a poco. Un
  salto brusco del `phase_increment` deformaría la onda. El filtro de lazo limita
  cuánto puede cambiar por vez (igual que el VREG_STEP limita la regulación).
- **Vive en el loop(), no en la ISR del MCPWM.** La ISR del SPWM sigue intocable.
  El ZCD marca el tiempo en SU ISR (corta); el cálculo del PLL y el ajuste de
  `phase_increment` se hacen en el loop, sin prisa crítica. La escritura de
  `phase_increment` es atómica (32 bits) y la ISR la lee en el siguiente ciclo.
- **Rango de enganche limitado.** El PLL solo intenta engancharse si la red está
  dentro de un rango razonable (ej. 59-61Hz). Fuera de eso, declara la red
  inválida y no intenta sincronizar (red sucia o inestable).

---

## 4. LAS DOS DIRECCIONES DE TRANSFERENCIA (distinto riesgo)

**Esto es lo más importante para la seguridad. No son igual de peligrosas.**

### 4.1 GRID → OFF-GRID (la red cae, el inversor toma) — MÁS SEGURA
- Cuando la red cae, el inversor sigue solo. La red ya no está, así que no hay dos
  fuentes peleando.
- La sincronización aquí AYUDA a que la carga no sienta el salto (el inversor ya
  venía enganchado en fase, sigue de corrido), pero no es crítica para evitar daño:
  aunque hubiera un pequeño salto, no hay red con quien chocar.
- Es la dirección que la pregunta original plantea. Defendible y relativamente
  segura. El hueco de transferencia (~30-65ms, ver doc de arquitectura) aplica.

### 4.2 OFF-GRID → GRID (vuelve la red, reconectar) — PELIGROSA
- Reconectar el inversor a la red SIN sincronización exacta puede causar
  **corrientes destructivas**: las dos fuentes a destiempo se pelean con toda su
  energía detrás. Puede dañar el inversor, disparar protecciones, o peor.
- Aquí el ZCD + PLL NO es una mejora de confort: es OBLIGATORIO y es una
  protección. **No se reconecta a la red hasta que se cumplan TODAS estas
  condiciones, confirmadas y sostenidas:**
  1. Frecuencia enganchada (inversor = red, dentro de tolerancia estrecha).
  2. Fase alineada (cruces por cero coincidentes, dentro de pocos grados).
  3. Amplitud compatible (voltajes similares).
  4. Red estable y presente de forma sostenida (no parpadeando).
- Solo cuando las cuatro se cumplen a la vez, y por un tiempo de confirmación, se
  habilita el cierre del relay de reconexión. Si cualquiera falla, NO se reconecta.

#### 4.2.1 EL BONDING N-G EN LA RECONEXIÓN (crítico, no omitir)
En off-grid el inversor tiene el **bonding N-G UNIDO** (es la fuente, crea la
referencia a tierra). La red, al volver, trae su PROPIO bonding del panel de la
acometida. Si se reconecta sin separar el bonding del inversor, quedan DOS bonding
simultáneos (inversor + red) = el **doble bonding peligroso** (lazos de corriente
de tierra, conductores de tierra energizados, protecciones confundidas). Ver
DISENO_BONDING_NEUTRO_TIERRA.md, sección 1.

**El bonding debe SEPARARSE como parte de la secuencia, en el orden correcto.**
La secuencia segura de reconexión offgrid→grid es:

```
  1. Detectar red presente y estable (sostenida).
  2. Enganchar el PLL: frecuencia y fase alineadas con la red.
  3. Verificar las 4 condiciones de sincronizacion (arriba), confirmadas.
  4. SEPARAR el bonding N-G del inversor (relay bonding -> ABRIR).
     -> ahora el inversor ya NO impone su referencia de tierra.
  5. CONFIRMAR que el bonding quedo abierto.
  6. SOLO ENTONCES cerrar el relay de transferencia a la red (reconectar).
     -> la red provee la unica referencia de tierra. Un solo bonding. Correcto.
```

**El orden es la seguridad:** primero separar el bonding del inversor, confirmar,
y DESPUÉS reconectar a la red. Nunca al revés, y nunca con ambos bonding cerrados
a la vez, ni siquiera por un instante. Si el bonding no se puede confirmar abierto,
NO se reconecta.

#### 4.2.2 EL BONDING EN LA DIRECCIÓN INVERSA (grid→offgrid, completar 4.1)
Al pasar de red a isla (sección 4.1), el bonding hace el camino contrario y también
en orden:

```
  1. Detectar caida de red (ausencia de cruces, sostenida).
  2. ABRIR el relay de transferencia (desconectar de la red).
  3. CONFIRMAR la desconexion de la red.
  4. SOLO ENTONCES cerrar el bonding N-G del inversor (relay bonding -> CERRAR).
     -> ahora el inversor es la fuente y crea su propia referencia de tierra.
```

De nuevo: primero desconectar de la red, confirmar, y DESPUÉS unir el bonding del
inversor. Nunca tener el bonding del inversor cerrado mientras aún se está
conectado a la red — eso es el doble bonding, aunque sea momentáneo.

**Resumen del bonding en transferencia:**
| Transición | Orden del bonding |
|------------|-------------------|
| grid→offgrid | desconectar red → confirmar → CERRAR bonding inversor |
| offgrid→grid | sincronizar → ABRIR bonding inversor → confirmar → reconectar red |

En ambas, la regla es la misma: **nunca dos bonding a la vez, ni un instante.** El
bonding del inversor solo está cerrado cuando el inversor es la única fuente.

**Regla:** grid→offgrid es desconexión + cierre de bonding (más simple). offgrid→grid
es apertura de bonding + re-enganche sincronizado (lo más delicado de todo el
sistema). Se construyen y validan por separado, y la reconexión solo después de la
desconexión sólida.

---

## 5. Mapeo de pines (referencia completa)

### 5.1 Pin del zero-cross para la sincronización
- **ZEROCROSS_PIN = GPIO39** (solo-entrada, interrupción), señal **ópticamente
  aislada** de la red. Nunca la línea AC directo al pin.
- La ISR del ZCD es mínima: marca `micros()`, levanta bandera, sale. El PLL se
  calcula en el loop().

### 5.2 Pines OCUPADOS por el inversor (verificados contra el código real)

| GPIO | Señal | Función | Tipo |
|------|-------|---------|------|
| 2  | ALERT_LED_PIN | LED de alerta | salida |
| 4  | FAN_PIN | Control del ventilador (vía transistor) | salida |
| 5  | FAULT_DRIVE | Señal de fault al driver (modo CYC) | salida |
| 18 | DIAG_CRUCE | Diagnóstico del cruce por cero (interno SPWM) | salida |
| 19 | LO2 | Puente H — pierna lenta, bajo | salida potencia |
| 21 | HO2 | Puente H — pierna lenta, alto | salida potencia |
| 22 | LO1 | Puente H — pierna rápida, bajo | salida potencia |
| 23 | HO1 | Puente H — pierna rápida, alto | salida potencia |
| 25 | LCD_SDA_PIN | I2C datos (LCD) — remapeado | I2C |
| 26 | LCD_SCL_PIN | I2C reloj (LCD) — remapeado | I2C |
| 32 | TEMP1_PIN | Termistor NTC batería (ADC1_CH4) | entrada ADC |
| 33 | TEMP2_PIN | Termistor NTC MOSFETs (ADC1_CH5) | entrada ADC |
| 34 | FB_ADC_PIN | Feedback de tensión AC (ADC1_CH6) | entrada ADC, solo-entrada |
| 35 | BAT_ADC_PIN | Tensión de batería DC (ADC1_CH7) | entrada ADC, solo-entrada |

Nota: los pines I2C por defecto (GPIO21/22) NO se usan para I2C porque chocan con
la potencia (HO2=21, LO1=22). El I2C del LCD se remapeó a GPIO25/26.

### 5.3 Pines PROPUESTOS — relays AC + zero-cross (para el modo red/transferencia)

| GPIO | Señal | Función | Tipo |
|------|-------|---------|------|
| 16 | RELAY_BONDING_PIN | Relay vínculo neutro-tierra (bonding N-G) | salida limpia |
| 17 | RELAY_XFER_L1_PIN | Relay transferencia Línea 1 | salida limpia |
| 27 | RELAY_XFER_N_PIN | Relay transferencia Neutro | salida limpia |
| 39 | ZEROCROSS_PIN | Detección de cruce por cero (AISLADA) | solo-entrada, interrupción |

- GPIO16, 17, 27 son salidas limpias sin strapping (no afectan el arranque).
- **La tierra de protección NO se conmuta y NO aparece aquí.** El relay "bonding"
  conmuta el VÍNCULO neutro-tierra (ver sección 4.2.1 y el documento de bonding).
- El relay de bonding (GPIO16) participa en las secuencias de transferencia de la
  sección 4.2: se separa antes de reconectar a la red, se une al pasar a isla.

### 5.4 Pines LIBRES de reserva (tras integrar todo)

| GPIO | Aptitud | Notas |
|------|---------|-------|
| 13 | salida/entrada | libre, limpio — reserva |
| 14 | salida/entrada | libre, limpio — reserva |
| 36 | solo-entrada | libre — reserva para otro sensor |

Quedan tres pines limpios de reserva. (Evitar: GPIO0, 12, 15 strapping; GPIO1, 3
UART consola; GPIO6-11 flash.)

### 5.5 Mapa visual de ocupación

```
  ESP32 WROOM — ocupacion de GPIO (con relays + zero-cross integrados)
  --------------------------------------------------------------
  0   strapping (evitar)        | 16  RELAY bonding N-G (nuevo)
  1   UART TX (consola)         | 17  RELAY xfer L1     (nuevo)
  2   ALERT_LED                 | 18  DIAG_CRUCE
  3   UART RX (consola)         | 19  LO2 (potencia)
  4   FAN                       | 21  HO2 (potencia)
  5   FAULT_DRIVE               | 22  LO1 (potencia)
  6-11  FLASH (no usar)         | 23  HO1 (potencia)
  12  strapping (evitar)        | 25  LCD SDA
  13  LIBRE (reserva)           | 26  LCD SCL
  14  LIBRE (reserva)           | 27  RELAY xfer N      (nuevo)
  15  strapping (cuidado)       | 32  TEMP1 (bateria)
                                | 33  TEMP2 (MOSFETs)
                                | 34  FB_ADC (feedback AC)
                                | 35  BAT_ADC (bateria)
                                | 36  LIBRE (reserva, solo-entrada)
                                | 39  ZEROCROSS (nuevo, aislado)
  --------------------------------------------------------------
```

Para el detalle ampliado del mapeo y la arquitectura de prioridades de interrupción,
ver ARQUITECTURA_ZEROCROSS_Y_MAPEO_PINES.md.

---

## 6. Plan de construcción por fases (cada una validada antes de la siguiente)

1. **Fase 1 — ZCD como sensor de presencia.** Detectar si hay red o no (ya cubierto
   en el doc de arquitectura). Sin sincronización todavía. Validar detección estable.

2. **Fase 2 — Medición de fase (sin actuar).** Con el ZCD, MEDIR el error de fase
   entre el inversor y la red, y solo REPORTARLO por serial. No ajustar nada aún.
   Validar que la medición es correcta y estable contra el analizador lógico.

3. **Fase 3 — PLL en lazo (engancha frecuencia/fase).** Cerrar el lazo: ajustar
   `phase_increment` suavemente hasta enganchar. Validar el enganche midiendo,
   SIN conectar a la red todavía (inversor enganchado a una referencia, observado).

4. **Fase 4 — Transferencia grid→offgrid.** La dirección segura. Incluye la
   secuencia de bonding (desconectar red → confirmar → cerrar bonding inversor,
   sección 4.2.2). Validar el hueco, la secuencia de bonding, y que la carga no
   sufre.

5. **Fase 5 — Reconexión offgrid→grid (la peligrosa).** Solo después de todo lo
   anterior sólido, con validación de potencia (sonda de corriente) y las cuatro
   condiciones de sincronización MÁS la secuencia de bonding (separar bonding →
   confirmar → reconectar, sección 4.2.1). Nunca dos bonding a la vez. Esta fase es
   la que más cuidado exige.

**Nunca saltar fases.** Cada una valida lo que la siguiente da por hecho.

---

## 7. Lo que este diseño NO resuelve todavía (honestidad de alcance)

- **No es rectificación síncrona.** Sincronizar la onda (este doc) es distinto de
  conmutar los MOSFETs para cargar. Son fases separadas.
- **La validación es de POTENCIA, no solo lógica.** El analizador lógico confirma
  la medición de fase y el enganche, pero la transferencia real y, sobre todo, la
  reconexión a la red, deben medirse con sonda de corriente en el sistema real.
- **La reconexión a la red puede requerir cumplimiento normativo** (anti-isla,
  protecciones de interconexión) según la jurisdicción. Conectar una fuente propia
  a la red de la utility tiene reglas estrictas por seguridad de la red y del
  personal. Revisar la norma local antes de cualquier reconexión real.
- **El modo bidireccional completo sigue siendo el horizonte, no este paso.** Esto
  sincroniza; cargar es otra capa con su propia validación.

---

## 8. Preguntas para la evaluación

1. ¿El alcance se limita por ahora a las Fases 1-4 (sensor, medición, PLL,
   transferencia grid→offgrid), dejando la reconexión offgrid→grid (Fase 5) como
   fase futura con validación de potencia y revisión normativa?
2. ¿Qué tolerancias se fijan para declarar "enganchado" (rango de frecuencia, error
   de fase máximo, ventana de amplitud)?
3. ¿Qué tan suave debe ser el filtro de lazo del PLL (cuánto puede cambiar
   `phase_increment` por ciclo) para no deformar la onda mientras engancha?
4. ¿La reconexión a la red, cuando llegue, requiere protecciones anti-isla y
   cumplimiento de la norma local de interconexión? (Casi con certeza sí.)
5. ¿Cómo se CONFIRMA el estado real del relay de bonding (abierto/cerrado) antes de
   continuar la secuencia? Un relay puede fallar pegado o no responder. La
   secuencia de transferencia depende de saber con certeza que el bonding quedó en
   el estado correcto antes del siguiente paso. ¿Se necesita un contacto auxiliar
   de realimentación del relay, o una verificación eléctrica del estado del bonding?

---

*Documento de arquitectura de sincronización para evaluación. Guía abierta para la
transición energética. El NCO que se dejó listo para PLL desde el inicio por fin
cierra su lazo: el inversor aprende a bailar al compás de la red. Pero la
reconexión a la red es el paso más delicado del sistema — se mide con corriente
real, se valida dos veces, y se respeta la norma. Sincronizar mal con la red no es
un golpecito: es un accidente. Medir antes de creer, aquí más que nunca.*
