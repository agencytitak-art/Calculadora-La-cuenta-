# Handoff — Calculadora "La Cuenta" (negocio de gas LP)

**Para:** Claude Code
**De:** Karla (dueña del proyecto, hija de los dueños del negocio)
**Archivo a intervenir:** `calculadora-la-cuenta.html`
**Documento de reglas de negocio:** `instrucciones-proyecto-gas-lp.md` — **es la fuente de verdad. Ante cualquier duda de negocio, gana ese archivo.**

---

## 1. Qué es esto

Un negocio familiar de distribución de gas LP en México. Una camioneta, dos trabajadores, 52 tanques. Cada noche los papás de Karla hacen "la cuenta": cuánto se vendió, cuánto costó, cuánto sobró.

Hoy eso se hace a mano. Este archivo lo automatiza.

El objetivo del handoff es convertir el HTML de un solo archivo en una **web funcional y desplegable** que los papás puedan abrir en el celular al terminar la ruta.

---

## 2. Estado actual

Un solo archivo HTML, sin build, sin dependencias más allá de Google Fonts. Vanilla JS dentro de un IIFE. Sin framework.

### Lo que ya funciona

| Función | Qué hace |
|---|---|
| Captura de tanques | Steppers con tope según lo que cabe en la camioneta (5/34/12/1) |
| Precio del tanque de 45 kg | Campo que aparece solo cuando se vende uno (precio variable) |
| Cálculo de "la cuenta" | Bruto → sueldos → planta → carburación → comisión |
| Barra de segmentos | Visualiza cómo el bruto se reparte entre costos, caja y comisión |
| Desglose del costo de planta | Una línea por tipo de tanque, con kg y pesos |
| Caja de ahorro | 10% de la comisión diaria; cubre días en rojo; paga imprevistos |
| Gasto imprevisto del día | Pestaña opcional, se guarda con la nota del día |
| Censo de tanques | Picados / vacíos / trabajando, con validaciones |
| Historial y resumen mensual | Días guardados, comisión acumulada, promedio, reparaciones |
| Persistencia | `window.storage` (API de artifacts de Claude) |

### Constantes (líneas iniciales del `<script>`)

```js
const PLANTA      = 14.06;        // pesos por KILOGRAMO. Nunca litros.
const SUELDOS     = 900;          // 2 trabajadores × $450
const CARBURACION = 20 * PLANTA;  // 281.20 — combustible diario de la camioneta
const FIJOS       = 1181.20;      // SUELDOS + CARBURACION
const TOTAL_TANQUES  = 52;
const REPARACION_MES = 1680;      // 4 picados × $420
const PORC_CAJA      = 0.10;      // 10% de la comisión

const TANQUES = [
  {kg:10, precio:225,  stock:5},
  {kg:20, precio:440,  stock:34},
  {kg:30, precio:660,  stock:12},
  {kg:45, precio:null, stock:1}   // precio variable, se captura a mano
];
```

### Fórmulas (no las cambies sin autorización de Karla)

```
kgTotal     = Σ (tanques vendidos × kg del tanque)
bruto       = Σ (tanques vendidos × precio del tanque)
costoPlanta = kgTotal × 14.06
comision    = bruto − 900 − costoPlanta − 281.20

si comision > 0 → abono  = comision × 0.10
si comision < 0 → retiro = min(saldoCaja, −comision)
neta        = comision − abono + retiro

saldoCaja   = Σ abonos − Σ retiros − Σ gastos
```

El **gasto imprevisto no toca la comisión del día**. Sale de la caja.

### Funciones principales

`cuenta()` calcula y devuelve el objeto del día. `calcular()` es el orquestador: llama a `recalcSaldo()`, `cuenta()`, y luego a `dibujarBarra()`, `dibujarDesglose()`, `avisar()` y `pintarCaja()`. Todo evento de UI termina llamando a `calcular()`.

`cargar()` / `persistir()` leen y escriben la llave única `gaslp:datos`.

### Formas de los datos

```js
// llave: 'gaslp:datos'
{
  dias: [{
    fecha: "2026-07-08T02:11:00.000Z",  // ISO, generada con new Date()
    tanques: 18, kg: 390, bruto: 8580,
    comision: 1215.60, abono: 121.56, retiro: 0,
    gasto: {concepto: "Llanta", monto: 1200} | null,
    nota: "Se ponchó saliendo de la planta" | null
  }],
  movimientos: [{
    fecha: "2026-07-08T02:11:00.000Z",
    tipo: "abono" | "retiro" | "gasto",
    concepto: "Abono del día (10%)",
    monto: 121.56
  }],
  inventario: {picados: 4, vacios: 2}
}
```

---

## 3. El bloqueo principal

`window.storage` **solo existe dentro de los artifacts de Claude.** En cuanto este archivo se sirva desde un servidor normal, la persistencia truena en silencio (el `try/catch` la degrada a memoria y el usuario pierde todo al recargar).

**Primera tarea, antes que cualquier otra:** meter un adaptador de almacenamiento detrás de la misma interfaz.

```js
const store = {
  async get(key)        { /* window.storage → localStorage → API */ },
  async set(key, value) { /* idem */ }
};
```

Detecta `typeof window.storage !== 'undefined'` y cae a `localStorage` si no existe. Así el archivo sigue funcionando en el artifact **y** en la web sin bifurcar el código.

---

## 4. Bugs conocidos

1. **Desincronización al borrar un día.** Borrar un día del historial no borra su abono, retiro ni gasto de la caja. El saldo queda inflado.
   *Arreglo:* darle un `id` (por ejemplo `crypto.randomUUID()`) a cada día y guardar `diaId` en cada movimiento generado por él. Al borrar el día, borrar sus movimientos en cascada. Los gastos capturados directo en la sección de la caja no llevan `diaId` y sobreviven.

2. **No se puede capturar un día pasado.** `new Date()` siempre pone hoy. Si se les olvidó registrar el martes, no hay manera de meterlo.
   *Arreglo:* un `<input type="date">` en la tarjeta del día, con hoy como valor por default.

3. **Nada impide guardar dos veces el mismo día.** Se duplica sin avisar.
   *Arreglo:* al guardar, si ya existe un día con esa fecha, preguntar si se reemplaza.

4. **Aritmética de punto flotante en dinero.** Los abonos del 10% arrastran decimales largos. Se ve bien porque `dinero()` formatea a dos decimales, pero las sumas acumuladas del mes derivan.
   *Arreglo:* guardar todo en centavos como enteros y dividir solo al pintar.

5. **Precios quemados en el código.** La planta sube el precio cada tanto y los precios a cliente los fija ella misma. Hoy eso obliga a editar el HTML.
   *Arreglo:* pantalla de ajustes, con los precios persistidos y con historial de vigencias (un día pasado debe recalcularse con el precio que estaba vigente ese día, no con el actual).

6. **El censo de tanques no tiene historia.** Solo guarda picados y vacíos actuales. Las instrucciones piden un censo al inicio de cada mes.
   *Arreglo:* guardar un snapshot mensual fechado.

7. **Google Fonts sin fallback real.** En la ruta, sin señal, la tipografía se cae a las fuentes del sistema. Funciona, pero se ve distinto. Considera hospedar las fuentes.

---

## 5. Hacia dónde llevarlo

En orden de valor para el negocio, no de dificultad técnica.

**Fase 1 — Que no se pierdan los datos**
1. Adaptador de almacenamiento (sección 3).
2. Bugs 1, 2 y 3.
3. Exportar a CSV. Es su respaldo y su forma de llevarlo al contador.

**Fase 2 — Que sirva en la calle**
4. Convertirlo en PWA instalable. **Offline-first no es opcional:** la cuenta se hace en la camioneta o al llegar, muchas veces sin señal. Service worker, cache del shell, escrituras a cola.
5. Botones y tipografía dimensionados para el pulgar. Ya está pensado para móvil, pero pruébalo en un celular real, no en el simulador.
6. Ajustes de precios con vigencias (bug 5).

**Fase 3 — Que se compartan**
7. Backend ligero con auth. Karla desde su casa, los papás desde la camioneta, el mismo dato.
8. Snapshot mensual del censo (bug 6) y reporte de cierre de mes.
9. Gráfica de comisión por día contra el punto de equilibrio (8 tanques de 20 kg).

**No hagas todavía**
- No metas React ni un framework a menos que la Fase 3 lo justifique. Son ~700 líneas de vanilla que funcionan.
- No agregues facturación, inventario de bodega ni nómina. No los pidieron.

---

## 6. Decisiones que ya se tomaron

Estas ya se discutieron con Karla. No las revisites sin preguntar.

- **Todo el negocio opera en kilogramos.** El precio de planta es $14.06 **por kg**. Nunca conviertas a litros ni introduzcas litros en ningún cálculo.
- Los precios a cliente los estipula la planta, no el negocio.
- Los $1,680 de reparación mensual se descuentan del **resumen del mes**, no de la cuenta diaria ni de la caja.
- El retiro de la caja para salir tablas **nunca** puede dejarla en negativo. Solo un gasto imprevisto puede.
- Un día en rojo no abona nada a la caja.
- La carburación se muestra aparte del costo de planta en el desglose, aunque se pague al mismo precio por kg. No es gas vendido.

---

## 7. Preguntas abiertas para Karla

Están al final de `instrucciones-proyecto-gas-lp.md`. Las que afectan el código:

- Precio del tanque de 45 kg: ¿hay fórmula o siempre es a criterio?
- ¿Se pagan los $900 de sueldo en días sin venta?
- ¿Hay tanques en bodega además de los 52 de la camioneta?
- ¿La comisión se reparte con alguien más?

No inventes defaults para estas. Pregunta.

---

## 8. Criterio de terminado (Fase 1)

- [ ] Se recarga la página y los datos siguen ahí, servido desde `file://` y desde un servidor.
- [ ] Se borra un día y el saldo de la caja baja en consecuencia.
- [ ] Se captura un día de la semana pasada con su fecha correcta.
- [ ] Se exporta un CSV que abre limpio en Excel, con acentos.
- [ ] Un mes con 25 días guardados suma exactamente lo mismo que sumar los días a mano, al centavo.
- [ ] Funciona en un iPhone y en un Android de gama baja, en vertical.
