# Guía de Desarrollo — Ruleta Mágica

Documento de referencia para el equipo de desarrollo backend. El enfoque principal es la lógica de `game.js`, los puntos de integración con Laravel y el sistema de premiación.

---

## Estructura del proyecto

```
Ruleta_Magica/
├── index.html      → Estructura HTML (310 líneas)
├── styles.css      → Estilos visuales (1275 líneas)
├── game.js         → Toda la lógica del juego (977 líneas) ← foco de este documento
└── DEVELOPMENT.md  → Este documento
```

---

## Mapa rápido de CSS (`styles.css`)

Para cambios visuales puntuales sin leer todo el archivo:

| Sección | Líneas | Qué controla |
|---|---|---|
| Variables de color y fuente | 1–28 | Colores globales (`--gold`, `--red`, fondos, textos) |
| Layout 3 columnas | 30–41 | Estructura izquierda / centro / derecha |
| Panel balance e historial | 42–214 | Caja de saldo, resultados anteriores |
| Tablero de números y chips | 276–479 | Colores rojo/negro/verde, fichas, botones de apuesta |
| Panel derecho (ticket) | 480–634 | Ticket, input monto, botón Jugar |
| Modal resultado | 659–710 | Ventana de ganancia/pérdida post-giro |
| Responsive mobile | 898–1138 | Adaptaciones para pantallas pequeñas |
| Splash screen | 1139–1275 | Pantalla de bienvenida |

---

## Lógica completa de `game.js`

### Variables de estado global (líneas 40–49)

Estas variables controlan el estado del juego en todo momento. Son el "corazón" del sistema.

```javascript
let balance      = 25430;   // Saldo actual del usuario en Bs. — DEBE venir del backend
let bets         = [];       // Array de apuestas activas antes de confirmar
let betConfirmed = false;    // true = apuesta bloqueada y lista para girar
let betIdCounter = 0;        // Contador de ID para cada apuesta (se resetea cada ronda)
let currentBetType = 'pleno'; // Tipo de apuesta activo en la UI
let currentChip  = 250;      // Valor de la ficha seleccionada actualmente
let selectedNumber = null;   // Número seleccionado en modo pleno
let timerSeconds = 60;       // Segundos restantes del contador
let timerInterval = null;    // Handle del setInterval del timer
let spinning     = false;    // true = la rueda está girando, bloquea toda acción
```

**Regla crítica:** `betConfirmed` y `spinning` son los dos guardias del sistema. Casi todas las funciones de apuesta verifican uno o ambos antes de ejecutar.

---

### Constantes del juego (líneas 74–75)

```javascript
const MAX_BET = 100000; // Monto máximo por apuesta

const PAYOUTS = {
  pleno:   36,   // Número exacto — paga 36× el monto apostado (net 35:1)
  rojo:     2,   // Color rojo — paga 2× (net 1:1)
  negro:    2,   // Color negro — paga 2× (net 1:1)
  verde:   18,   // 0 o 00 — paga 18× (net 17:1, estándar ruleta americana)
  par:      2,   // Números pares — paga 2×
  impar:    2,   // Números impares — paga 2×
  docena1:  3,   // Números 1–12 — paga 3× (net 2:1)
  docena2:  3,   // Números 13–24 — paga 3×
  docena3:  3,   // Números 25–36 — paga 3×
};
```

**Este objeto es la fuente única de verdad para los pagos.** Se usa tanto en `renderBets()` (para mostrar el pago potencial estimado) como en `resolveGame()` (para calcular las ganancias reales). Si se necesita ajustar un multiplicador, se cambia aquí y aplica a todo.

---

### Definición de colores (líneas 2–3)

```javascript
const RED_NUMBERS   = [1,3,5,7,9,12,14,16,18,19,21,23,25,27,30,32,34,36]; // 18 números
const BLACK_NUMBERS = [2,4,6,8,10,11,13,15,17,20,22,24,26,28,29,31,33,35]; // 18 números
// Verde: solo '0' y '00' — total 38 números (ruleta americana)
```

Función `getNumColor(num)` (línea 5): recibe un número como string o entero, devuelve `'rojo'`, `'negro'` o `'verde'`.

---

## Ciclo completo de una jugada

Entender este flujo es esencial para saber dónde integrar cada llamada al backend.

```
┌─────────────────────────────────────────────────────────┐
│  1. Página carga → enterGame()                          │
│     └─ Iniciar timer (60s), música, audio               │
│     └─ ⚡ BACKEND: obtener balance real del usuario      │
├─────────────────────────────────────────────────────────┤
│  2. Usuario construye su ticket de apuestas             │
│     └─ selectChip() → acumula monto en el input         │
│     └─ selectNumber() / selectBetType() → addBet()      │
│     └─ bets[] se construye en memoria (solo frontend)   │
├─────────────────────────────────────────────────────────┤
│  3. Usuario hace clic en "JUGAR" → confirmBet()         │
│     └─ Valida saldo localmente                          │
│     └─ betConfirmed = true (bloquea cambios en ticket)  │
│     └─ ⚡ BACKEND: registrar ticket en BD               │
├─────────────────────────────────────────────────────────┤
│  4. Timer llega a 0 → triggerSpin() (automático)        │
│     └─ Genera número ganador (Math.random — ver nota)   │
│     └─ Anima rueda 6 segundos                           │
│     └─ Muestra resultado en pantalla 2.5 segundos       │
├─────────────────────────────────────────────────────────┤
│  5. resolveGame(totalBet, result)                       │
│     └─ Calcula ganancias usando PAYOUTS                 │
│     └─ Actualiza balance en pantalla                    │
│     └─ ⚡ BACKEND: registrar resultado y balance nuevo  │
│     └─ Resetea todo para la próxima ronda               │
├─────────────────────────────────────────────────────────┤
│  6. showResult() → modal de resultado 4 segundos        │
│     └─ Muestra número, color y ganancia/pérdida neta    │
│     └─ Cierre automático → regresa al paso 2            │
└─────────────────────────────────────────────────────────┘
```

---

## Detalle de cada función clave

### `confirmBet()` — línea 406

**Cuándo se ejecuta:** El usuario hace clic en el botón "JUGAR".

```javascript
function confirmBet() {
  if (bets.length === 0) return;                              // Sin apuestas, no hace nada
  const total = bets.reduce((s, b) => s + b.amount, 0);      // Suma todos los montos
  if (balance < total) { showToast('Saldo insuficiente'); return; } // Valida saldo

  betConfirmed = true;  // ← El flag más importante del sistema
  // Cambia el botón a "✓ APUESTA CONFIRMADA" y lo deshabilita
}
```

**Qué hace `betConfirmed = true`:** A partir de este momento, todas las funciones de apuesta (`selectBetType`, `addBet`, `selectNumber`, `addCategoryBet`, `selectDocena`) retornan inmediatamente sin hacer nada. El ticket queda congelado.

**⚡ Punto de integración backend:** Aquí se debe enviar el ticket al servidor para registrarlo en BD antes de que gire la rueda.

---

### `triggerSpin()` — línea 552

**Cuándo se ejecuta:** Cuando el timer llega a 0 (automáticamente). También puede llamarse manualmente.

**Dos escenarios importantes:**

| Escenario | `betConfirmed` | Qué pasa |
|---|---|---|
| Usuario confirmó apuesta | `true` | Se procesa el ticket, se calcula ganancia/pérdida |
| Timer expiró sin confirmar | `false` | La rueda gira igualmente pero el ticket no se procesa, balance no cambia |

```javascript
function triggerSpin() {
  if (spinning) return;   // Evita doble giro
  spinning = true;

  // total = 0 si no hay apuesta confirmada
  const total = betConfirmed ? bets.reduce((s, b) => s + b.amount, 0) : 0;

  // ⚠️ GENERACIÓN DEL NÚMERO GANADOR — actualmente en el frontend
  const allNums = ['00', ...Array.from({length:37}, (_,i) => i.toString())];
  const result = allNums[Math.floor(Math.random() * allNums.length)];
  // result es un string: '0', '00', '1', '2', ..., '36'

  // ... animación de 6 segundos ...

  // Al terminar la animación, espera 2.5s y llama:
  resolveGame(total, result);
}
```

**⚠️ Nota crítica sobre Math.random():** Para un sistema de apuestas real, el número ganador NO debe generarse en el navegador del usuario. El flujo correcto para producción es:
1. El frontend solicita al servidor iniciar un giro: `POST /api/ruleta/iniciar-giro`
2. El servidor genera el número con un RNG seguro y lo guarda en BD
3. El servidor devuelve el número al frontend
4. El frontend usa ese número para animar la rueda hasta la posición correcta

---

### `resolveGame(totalBet, result)` — línea 658

**Esta es la función más crítica del sistema.** Recibe el total apostado y el número ganador, calcula ganancias y actualiza todo.

**Paso 1 — Determinar color del resultado (líneas 659–664):**
```javascript
const numInt = result === '00' ? -1 : parseInt(result);
// numInt = -1 para '00', 0 para '0', 1-36 para los demás

if (result === '0' || result === '00')      colorClass = 'verde';
else if (RED_NUMBERS.includes(numInt))      colorClass = 'rojo';
else                                         colorClass = 'negro';
```

**Paso 2 — Caso sin apuesta confirmada (líneas 668–683):**
```javascript
if (!betConfirmed) {
  // Solo registra en historial local y reinicia el timer
  // El balance NO se modifica
  // El ticket de apuestas NO se limpia (el usuario puede seguir editándolo)
  spinning = false;
  timerSeconds = 60;
  return; // ← Sale aquí, no continúa con el cálculo
}
```

**Paso 3 — Cálculo de ganancias (líneas 686–698):**
```javascript
bets.forEach(bet => {
  const wins =
    (bet.type === 'pleno'   && bet.num.toString() === result)    ||
    (bet.type === 'rojo'    && colorClass === 'rojo')            ||
    (bet.type === 'negro'   && colorClass === 'negro')           ||
    (bet.type === 'verde'   && colorClass === 'verde')           ||
    (bet.type === 'par'     && numInt > 0 && numInt % 2 === 0)   ||
    (bet.type === 'impar'   && numInt > 0 && numInt % 2 !== 0)   ||
    (bet.type === 'docena1' && numInt >= 1  && numInt <= 12)      ||
    (bet.type === 'docena2' && numInt >= 13 && numInt <= 24)      ||
    (bet.type === 'docena3' && numInt >= 25 && numInt <= 36);

  if (wins) winnings += bet.amount * PAYOUTS[bet.type];
  // Ejemplo: apuesta pleno $500 al número 7, cae el 7
  //          winnings += 500 * 36 = 18.000 Bs.
});
```

**Casos especiales a tener en cuenta:**
- `bet.type === 'par'` y cae el `0` o `00`: `numInt` es 0 o -1, por lo tanto `numInt > 0` es `false` → la apuesta par **pierde** (correcto en ruleta)
- `bet.type === 'impar'` y cae el `0` o `00`: mismo caso → **pierde** (correcto)
- Es posible tener múltiples apuestas ganadoras en el mismo giro. Ejemplo: apostar a Rojo y también al número 7 rojo — ambas ganan si cae el 7

**Paso 4 — Actualización del balance (líneas 700–701):**
```javascript
balance = balance - totalBet + winnings;
// Si apostó 1000 y ganó 2000: balance = balance - 1000 + 2000 = balance + 1000
// Si apostó 1000 y no ganó:   balance = balance - 1000 + 0    = balance - 1000
updateBalance(); // Actualiza el DOM con el nuevo valor
```

**⚡ Punto de integración backend:** Antes de esta línea, enviar al servidor el resultado completo para registrarlo. El servidor debe responder con el balance oficial desde BD, que reemplaza el cálculo local.

**Paso 5 — Reset para la próxima ronda (líneas 703–727):**
```javascript
spinning     = false;
betConfirmed = false;
betIdCounter = 0;
bets         = [];          // ← El ticket se limpia AQUÍ
currentBetType = 'pleno';
selectedNumber = null;
timerSeconds = 60;          // ← El timer reinicia
// Se limpian también todas las selecciones visuales del tablero
```

---

### `showResult(num, colorName, colorClass, winnings, totalBet)` — línea 774

Muestra el modal de resultado al usuario. Calcula la ganancia **neta** (lo que realmente ganó o perdió):

```javascript
const netGain = winnings - totalBet;
// Ejemplo: apostó 1000, ganó 2000 → netGain = +1000 (ganancia real)
// Ejemplo: apostó 1000, no ganó   → netGain = -1000 (pérdida)
// Ejemplo: apostó 500 en rojo y 500 en negro, cae el rojo:
//          winnings = 1000, totalBet = 1000 → netGain = 0 (empata)

if (netGain > 0) → muestra "+Bs. X" en verde
else             → muestra "-Bs. X" en rojo
```

El modal se cierra automáticamente a los 4 segundos.

---

### `addBet()` — línea 223

Se llama cada vez que el usuario coloca una apuesta en el ticket. Construye el objeto `bet`:

```javascript
// Monto: usa el valor del input si existe, sino usa el chip seleccionado como fallback
const amount = inputVal > 0 ? inputVal : currentChip;

bets.push({
  id:        betIdCounter++,    // ID incremental (solo para la UI de esta ronda)
  type:      currentBetType,    // 'pleno', 'rojo', 'negro', 'verde', 'par', 'impar', 'docena1/2/3'
  num:       numVal,            // String del número si es pleno ('7', '00'), null si es categoría
  amount:    amount,            // Monto en Bs. (float)
  label:     label,             // Texto del badge en el ticket ('07', 'R', 'N', 'V', etc.)
  badgeClass: badgeClass,       // Clase CSS del badge ('rojo', 'negro', 'verde')
  display:   display            // Descripción ('Pleno', 'Rojo', 'Negro', '1ª Docena', etc.)
});
```

**Reglas de negocio en `addBet()`:**
- Si `betConfirmed === true`, retorna sin hacer nada (apuesta ya cerrada)
- Si `amount <= 0`, retorna sin hacer nada
- Para pleno, si no hay `selectedNumber`, retorna sin hacer nada
- Después de agregar, resetea el input a vacío (el usuario puede agregar otra apuesta)

---

### `renderBets()` — línea 290

Redibuja el ticket completo cada vez que cambia `bets[]`. También recalcula y muestra:
- **Total apostado** → `#totalApostado` (suma de todos los `bet.amount`)
- **Pago potencial** → `#pagoPotencial` (suma de `bet.amount * PAYOUTS[bet.type]` para cada apuesta)

El "Pago potencial" es el **máximo posible si todas las apuestas ganaran simultáneamente** — es orientativo, no garantizado.

---

### `updateBalance()` — línea 747

```javascript
function updateBalance() {
  const formatted = `Bs. ${balance.toLocaleString('es-VE', {minimumFractionDigits:2})}`;
  document.getElementById('balanceDisplay').textContent = formatted;       // desktop
  document.getElementById('mobileBalanceDisplay').textContent = formatted; // mobile
}
```

Se llama automáticamente después de cada `resolveGame()`. Para actualizar el balance desde el backend, simplemente se modifica la variable global `balance` y se llama a esta función.

---

### `addHistory(num, colorClass)` — línea 754

Agrega una entrada al historial local de los últimos 10 resultados (panel izquierdo). Solo es visual — no se persiste. El historial real debe cargarse desde el backend al iniciar.

---

## IDs del DOM que el JS controla

Estos IDs no deben renombrarse en `index.html` sin actualizar también `game.js`:

| ID | Qué muestra | Controlado por |
|---|---|---|
| `balanceDisplay` | Saldo del usuario (desktop) | `updateBalance()` |
| `mobileBalanceDisplay` | Saldo del usuario (mobile) | `updateBalance()` |
| `timerDisplay` | Contador regresivo 1:00 | `updateTimerDisplay()` |
| `betAmount` | Input de monto a apostar | `addBet()`, chips, reset |
| `activeBetsList` | Lista de apuestas en el ticket | `renderBets()` |
| `totalApostado` | Suma total del ticket | `renderBets()` |
| `pagoPotencial` | Máximo pago posible | `renderBets()` |
| `btnJugar` | Botón JUGAR / APUESTA CONFIRMADA | `confirmBet()`, `resolveGame()` |
| `resultOverlay` | Modal de resultado final | `showResult()` |
| `resultBig` | Número grande en el modal | `showResult()` |
| `resultColorName` | ROJO / NEGRO / VERDE | `showResult()` |
| `resultWinLoss` | +/- ganancia neta | `showResult()` |
| `lastResultDisplay` | Último número (esquina) | `resolveGame()` |
| `historyList` | Historial últimos 10 | `addHistory()` |
| `toastMsg` | Mensaje de error flotante | `showToast()` |

---

## Integración con Laravel — implementación completa

### Paso 0: Helper de peticiones HTTP

Agregar al inicio de `game.js`, después de las constantes (línea ~76):

```javascript
// ── HELPER LARAVEL ────────────────────────────────────────
async function apiPost(url, data) {
  const token = document.querySelector('meta[name="csrf-token"]')?.content
              ?? window.RULETA_CONFIG?.csrfToken ?? '';
  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'X-CSRF-TOKEN': token
      },
      body: JSON.stringify(data)
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (err) {
    console.error('[Ruleta API Error]', url, err);
    throw err;
  }
}
// ─────────────────────────────────────────────────────────
```

### Paso 1: Inyectar token CSRF

En `index.html`, dentro del `<head>` (si se usa Laravel Blade):
```html
<meta name="csrf-token" content="{{ csrf_token() }}">
```

### Paso 2: Inyectar balance real del usuario

**En `index.html`, antes del `<script src="game.js">`** (si se usa Blade):
```html
<script>
  window.RULETA_CONFIG = {
    balance:   {{ auth()->user()->balance }},
    userName:  "{{ auth()->user()->name }}",
    userId:    {{ auth()->user()->id }},
    csrfToken: "{{ csrf_token() }}"
  };
</script>
<script src="game.js"></script>
```

**En `game.js`, línea 40** — cambiar:
```javascript
// ANTES (hardcodeado):
let balance = 25430;

// DESPUÉS (desde backend):
let balance = window.RULETA_CONFIG?.balance ?? 0;
```

### Paso 3: Registrar ticket al confirmar — `confirmBet()`

Modificar `confirmBet()` en `game.js` línea 406:

```javascript
function confirmBet() {
  if (bets.length === 0) return;
  const total = bets.reduce((s, b) => s + b.amount, 0);
  if (balance < total) { showToast('Saldo insuficiente'); return; }

  // ── INTEGRACIÓN: registrar ticket en BD ───────────────────
  apiPost('/api/ruleta/confirmar-apuesta', {
    apuestas: bets.map(b => ({
      tipo:        b.type,
      numero:      b.num,        // null si no es pleno
      monto:       b.amount,
      descripcion: b.display
    })),
    total_apostado: total,
    timestamp:      new Date().toISOString()
  })
  .then(data => {
    window._ticketId = data.ticket_id; // Guardar para correlacionar con resultado
    // Ahora sí bloquear el ticket
    betConfirmed = true;
    document.getElementById('btnJugar').textContent = '✓ APUESTA CONFIRMADA';
    document.getElementById('btnJugar').style.background = 'linear-gradient(135deg, #166534 0%, #16a34a 100%)';
    document.getElementById('btnJugar').disabled = true;
  })
  .catch(() => {
    showToast('Error al registrar apuesta. Intente de nuevo.');
    // betConfirmed permanece false — el usuario puede reintentar
  });
  // ─────────────────────────────────────────────────────────
}
```

> **Nota:** En esta versión la confirmación es asíncrona. Si el servidor rechaza la apuesta (saldo insuficiente real, sesión expirada), el frontend lo maneja antes de bloquear el ticket.

**Payload enviado al servidor:**
```json
{
  "apuestas": [
    { "tipo": "pleno",  "numero": "7",  "monto": 500,  "descripcion": "Pleno" },
    { "tipo": "rojo",   "numero": null, "monto": 250,  "descripcion": "Rojo"  }
  ],
  "total_apostado": 750,
  "timestamp": "2026-03-28T14:30:00.000Z"
}
```

**Respuesta esperada:**
```json
{
  "status": "ok",
  "ticket_id": 1042
}
```

---

### Paso 4: Registrar resultado — `resolveGame()`

Modificar `resolveGame()` en `game.js` línea 658. Agregar **después del cálculo de `winnings` (línea 698) y antes del reset (línea 700)**:

```javascript
// ... cálculo de winnings (existente) ...

// ── INTEGRACIÓN: guardar resultado en BD ──────────────────
if (betConfirmed) {
  apiPost('/api/ruleta/registrar-resultado', {
    ticket_id:       window._ticketId ?? null,
    numero_ganador:  result,           // string: '0', '00', '1'...'36'
    color_ganador:   colorClass,       // 'rojo', 'negro' o 'verde'
    total_apostado:  totalBet,
    ganancia_bruta:  winnings,         // retorno total (incluye el monto apostado)
    ganancia_neta:   winnings - totalBet, // lo que realmente ganó o perdió
    balance_nuevo:   balance - totalBet + winnings,
    apuestas_detalle: bets.map(b => ({
      tipo:    b.type,
      numero:  b.num,
      monto:   b.amount,
      gano:    // calcular si ganó esta apuesta específica
        (b.type === 'pleno'   && b.num.toString() === result) ||
        (b.type === 'rojo'    && colorClass === 'rojo')       ||
        (b.type === 'negro'   && colorClass === 'negro')      ||
        (b.type === 'verde'   && colorClass === 'verde')      ||
        (b.type === 'par'     && parseInt(result) > 0 && parseInt(result) % 2 === 0) ||
        (b.type === 'impar'   && parseInt(result) > 0 && parseInt(result) % 2 !== 0) ||
        (b.type === 'docena1' && parseInt(result) >= 1  && parseInt(result) <= 12)   ||
        (b.type === 'docena2' && parseInt(result) >= 13 && parseInt(result) <= 24)   ||
        (b.type === 'docena3' && parseInt(result) >= 25 && parseInt(result) <= 36)
    })),
    timestamp: new Date().toISOString()
  })
  .then(data => {
    // El servidor es la fuente de verdad del balance
    if (data.balance !== undefined) {
      balance = data.balance;
      updateBalance();
    }
    window._ticketId = null;
  })
  .catch(err => {
    console.error('Error registrando resultado:', err);
    // El juego continúa aunque falle el registro
    // Registrar en un log local para reintentar después
  });
}
// ─────────────────────────────────────────────────────────

// A partir de aquí continúa el reset existente:
balance = balance - totalBet + winnings; // Solo como fallback si el servidor no responde
updateBalance();
// ... resto del reset ...
```

**Payload enviado al servidor:**
```json
{
  "ticket_id": 1042,
  "numero_ganador": "7",
  "color_ganador": "rojo",
  "total_apostado": 750,
  "ganancia_bruta": 18000,
  "ganancia_neta": 17250,
  "balance_nuevo": 42680,
  "apuestas_detalle": [
    { "tipo": "pleno", "numero": "7",  "monto": 500, "gano": true  },
    { "tipo": "rojo",  "numero": null, "monto": 250, "gano": true  }
  ],
  "timestamp": "2026-03-28T14:30:47.000Z"
}
```

**Respuesta esperada:**
```json
{
  "status": "ok",
  "balance": 42680.00
}
```

---

### Paso 5: Rutas Laravel sugeridas

```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    Route::get( '/ruleta/balance',              [RuletaController::class, 'obtenerBalance']);
    Route::post('/ruleta/confirmar-apuesta',    [RuletaController::class, 'confirmarApuesta']);
    Route::post('/ruleta/registrar-resultado',  [RuletaController::class, 'registrarResultado']);
    Route::post('/ruleta/iniciar-giro',         [RuletaController::class, 'iniciarGiro']); // para RNG server-side
});
```

**Estructura recomendada de BD:**

```sql
-- Ticket de apuesta
CREATE TABLE ruleta_tickets (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id       BIGINT NOT NULL,
  total_apostado DECIMAL(12,2) NOT NULL,
  confirmado_en TIMESTAMP,
  creado_en     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Detalle por tipo de apuesta
CREATE TABLE ruleta_apuestas (
  id         BIGINT PRIMARY KEY AUTO_INCREMENT,
  ticket_id  BIGINT NOT NULL,
  tipo       VARCHAR(10) NOT NULL,  -- pleno, rojo, negro, verde, par, impar, docena1/2/3
  numero     VARCHAR(2),            -- '0', '00', '7', etc. — NULL si no es pleno
  monto      DECIMAL(12,2) NOT NULL,
  descripcion VARCHAR(20)
);

-- Resultado de cada giro
CREATE TABLE ruleta_resultados (
  id              BIGINT PRIMARY KEY AUTO_INCREMENT,
  ticket_id       BIGINT,           -- NULL si giró sin apuesta
  numero_ganador  VARCHAR(2) NOT NULL,
  color_ganador   VARCHAR(6) NOT NULL,
  total_apostado  DECIMAL(12,2),
  ganancia_bruta  DECIMAL(12,2),
  ganancia_neta   DECIMAL(12,2),
  balance_antes   DECIMAL(12,2),
  balance_despues DECIMAL(12,2),
  creado_en       TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Consideraciones de seguridad para producción

### ⚠️ Número ganador en el frontend
**Riesgo:** `Math.random()` en el navegador es predecible y manipulable por el usuario.

**Solución para producción:**
1. Frontend llama `POST /api/ruleta/iniciar-giro` al momento de confirmar la apuesta
2. Backend genera el número con `random_int()` de PHP (criptográficamente seguro) y lo guarda en BD
3. Backend devuelve el número al frontend
4. Frontend anima la rueda hasta ese número (ya conoce el resultado antes de girar)

```javascript
// En triggerSpin(), reemplazar líneas 561-562:
// ANTES:
const result = allNums[Math.floor(Math.random() * allNums.length)];

// DESPUÉS:
const data = await apiPost('/api/ruleta/iniciar-giro', { ticket_id: window._ticketId });
const result = data.numero; // El servidor decide el resultado
```

### ⚠️ El servidor es la fuente de verdad del balance
El cálculo de balance en el frontend (`balance = balance - totalBet + winnings`) es solo para mostrar el resultado inmediatamente. El balance real siempre debe venir de la respuesta del servidor en `registrarResultado`.

### ⚠️ Validar todo en el servidor
Aunque el frontend valida `balance < total`, el servidor debe validar independientemente:
- Que el usuario tiene saldo suficiente (no confiar en la validación del JS)
- Que los montos son positivos y no superan `MAX_BET`
- Que el `ticket_id` pertenece al usuario autenticado
- Que no se está registrando un resultado para un ticket ya resuelto
