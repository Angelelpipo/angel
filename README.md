# Manipulator Engine

Estrategia de trading construida sobre una premisa invertida: en vez de predecir
hacia dónde irá el precio, mapear hacia dónde lo llevaría alguien que necesita
liquidez para llenar tamaño.

No se trata de creer que hay un señor moviendo el gráfico. Se trata de que un
stop-loss es una orden de mercado dormida, un grupo de stops es un depósito de
liquidez, y las órdenes grandes solo se llenan donde hay contraparte. El precio
viaja hacia donde está esa contraparte. Eso no es teoría de conspiración: es
microestructura, y es medible.

## Estado

| Módulo | Archivo | Estado |
|---|---|---|
| 01 · Mapa de liquidez | `pine/01_liquidity_map.pine` | ✅ |
| 02 · Índice de manipulabilidad | `pine/02_manipulability_index.pine` | ✅ |
| 03 · Escalera (step trailing stop) | `pine/03_ladder_exit.pine` | ✅ |
| 04 · Motor de señal (trampa + desplazamiento + entrada escalonada) | — | pendiente |
| 05 · Ensamblaje y scoring 0-100 | — | pendiente |

Todo escrito desde primitivas: precio, volumen, tiempo, open interest y funding.
Sin osciladores, sin medias como señal, sin plantillas ni scripts de terceros.
Las funciones matemáticas del lenguaje (máximo, suma, desviación, percentil) se
usan como lo que son: aritmética, no indicadores.

---

## Módulo 01 · Mapa de liquidez

Detecta y puntúa los depósitos de stops de los últimos N días (30 por defecto).

**Cómo se construye un pool.** Se detectan extremos locales con una
implementación propia de swing que compara de forma estricta a los lados, para
que los máximos *iguales* no se descarten — son justamente los que más importan,
porque un doble techo perfecto es un cartel de neón que anuncia dónde están los
stops. Extremos dentro de una tolerancia se fusionan en un solo pool y suman
toques. Se añaden además los niveles estructurales que ve cualquiera con una
plataforma: máximo y mínimo del día previo y de la semana previa.

**Scoring de magnetismo (0-100).** Cinco preguntas ponderadas:

| Componente | Peso | Qué mide |
|---|---|---|
| Toques | 25 | Cuánta gente dejó órdenes en ese nivel |
| Igualdad | 20 | Cuán obvio y limpio se ve — cuanto más de manual, más cebo |
| Volumen atrapado | 20 | Cuánto capital quedó del lado equivocado (volumen × mecha) |
| Número redondo | 15 | Los precios que la gente recuerda son donde la gente deja órdenes |
| Proximidad | 20 | Decaimiento exponencial con la distancia, en unidades de volatilidad |

**Barrido vs rotura.** La distinción que decide todo lo demás:

- **BARRIDO** — el precio perforó el nivel y cerró de vuelta dentro. La liquidez
  se cobró y el movimiento era mentira. Es la trampa.
- **ROTURA** — el precio perforó y se quedó fuera. La liquidez se cobró pero el
  movimiento era real. Es continuación.

Confundirlas es la forma más rápida de perder dinero con esta lógica, así que
el módulo las separa explícitamente con una ventana de confirmación.

**Salidas.** Rango operativo de 30 días con su equilibrio (premium/descuento),
mejor objetivo a cada lado, y el **sesgo de imán**: `(combustible arriba −
combustible abajo) / total × 100`. Responde a la pregunta operativa real — dónde
hay más comida sin cobrar. No es una señal de entrada, es el sesgo del entorno.

---

## Módulo 02 · Índice de manipulabilidad

Mide si un activo tiende trampas *antes* de operarlo, y rankea una lista de
símbolos. El activo manipulable rota; fijar un símbolo para siempre es una
decisión arbitraria, dejar que la data elija es una decisión medible.

| Componente | Peso | Fuente |
|---|---|---|
| Combustible | 22 | Open Interest / volumen — apalancamiento por dólar real |
| Posicionamiento | 18 | Violencia e inestabilidad del funding |
| Barridos | 22 | Perforaciones del extremo de N barras con cierre de vuelta, por 100 barras |
| Rechazo | 14 | Dominancia de mecha media |
| Ineficiencia | 12 | Camino recorrido frente a distancia neta |
| Obviedad | 12 | Densidad de extremos iguales |

Los cuatro últimos salen solo de precio y volumen, así que son portables a
cualquier símbolo — eso es lo que hace posible el escáner multi-activo (10
símbolos, ordenados por índice).

**Sobre los feeds de derivados.** Open interest y funding son lo más cerca que
se puede estar de leer el posicionamiento de la multitud sin adivinar. Los
tickers por defecto son `<símbolo>_OI` y `<símbolo>_FUNDING`, que es la
convención habitual pero **debes verificarla en tu proveedor**: el panel muestra
`OI ✓/✗  Funding ✓/✗` para que se vea de inmediato si resolvieron. Si un feed no
existe, su peso se reparte entre los componentes disponibles en vez de
sustituirse por cero — un dato ausente no es un dato bajo.

El panel también da la lectura direccional del flujo de OI, que es la más
accionable de todas:

- Precio ↑ + OI ↑ → **largos apilándose**, el combustible está debajo.
- Precio ↓ + OI ↑ → **cortos apilándose**, el combustible está arriba.
- OI ↓ → solo cierres: movimiento sin gasolina, poco fiable.

---

## Módulo 03 · Escalera (step trailing stop)

```
Entrada 100 · SL 95 · TP1 105 · TP2 110 · TP3 115 · TP4 120

alcanza TP1  →  SL sube a 100  (la entrada)
alcanza TP2  →  SL sube a 105  (el antiguo TP1)
alcanza TP3  →  SL sube a 110  (el antiguo TP2)
alcanza TP4  →  SL sube a 115  (el antiguo TP3)
```

Regla general: **al conquistar el peldaño k, el SL pasa al peldaño k−2**, siendo
el peldaño 0 la propia entrada. El piso solo sube; nunca baja mientras el trade
vive.

Por qué encaja aquí: un trailing continuo te saca en el primer retroceso, y el
retroceso es exactamente lo que provoca quien maneja el gráfico. La escalera se
mueve por **conquista**, no por proximidad. Eso la hace inmune a las sacudidas
de diseño y compatible con dejar correr un tramo largo hasta el siguiente
depósito de liquidez.

**Modos de colocación de peldaños:**

- `Múltiplos de R` — 1R, 2R, 3R… con R = riesgo inicial. Con espaciado 1.0
  reproduce el ejemplo de arriba exactamente.
- `Porcentaje` — espaciado fijo en %.
- `Precios manuales` — seis niveles a mano.
- `Pools de liquidez` — cada peldaño en el siguiente depósito de stops. El
  objetivo deja de ser un número bonito y pasa a tener una razón de existir.

**Opciones relevantes:**

- `Cerrar en el último peldaño` — desactivado, la escalera sigue subiendo
  indefinidamente con el mismo espaciado y el trade solo acaba cuando el precio
  pierde el piso.
- `Cerrar % en cada peldaño` — 0 por defecto: la posición entera corre y solo
  sube el piso.
- `Modo intrabarra conservador` — **déjalo activado.** Pine no sabe si dentro de
  una barra ocurrió antes el máximo o el mínimo. Activado, un peldaño
  conquistado solo eleva el SL a partir de la barra siguiente, asumiendo siempre
  lo peor. Desactivarlo infla el backtest.

La señal de entrada incluida (barrido y recuperación del extremo de N barras) es
**provisional**: existe para que la escalera se pueda backtestear y auditar por
sí sola. El motor de señal real es el módulo 04.

---

## Uso

Cada archivo es un script independiente: copiar y pegar en el Pine Editor de
TradingView. No hay dependencias entre scripts ni librerías que publicar.

Orden recomendado:

1. Abre el **02** primero y mira el escáner. Si el activo puntúa por debajo de
   45, no hay materia prima y no hay estrategia que valga — cambia de símbolo.
2. Abre el **01** sobre el activo elegido. Lee el sesgo de imán y la zona del
   rango antes de mirar nada más.
3. El **03** se backtestea por separado para calibrar espaciado, número de
   peldaños y distancia de stop en ese activo.

Desarrollo recomendado en BTCUSDT perp aunque no sea donde está el edge más
gordo: el libro es profundo y la data limpia, así que el backtest no miente. Una
vez validado el motor, el escáner del módulo 02 dice a dónde llevarlo.

## Advertencias técnicas

- **Repintado.** Los valores de temporalidad mayor usan el idioma no repintable
  (valor ya cerrado `[1]` + `lookahead_on`). Si añades `request.security` propio,
  respétalo o el backtest saldrá espectacular y en vivo no funcionará. Es el
  error número uno en estrategias de este tipo.
- **Comisiones y slippage.** El módulo 03 arranca con 0.05% de comisión y 2
  ticks de slippage, valores realistas de perp. No los pongas a cero.
- **Muestra.** 30 días es el mapa, no el sesgo. La dirección macro tiene que
  venir de temporalidad mayor; con un mes de datos se mapea liquidez, no se
  determina tendencia.
- **Filtro de régimen (pendiente, módulo 04).** Un barrido en rango es
  reversión; en tendencia fuerte es continuación — el precio limpia stops
  contrarios y sigue. Sin ese filtro la lógica fadea tendencias y sangra.
