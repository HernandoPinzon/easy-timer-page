# Plan: Fix timer drift with wall-clock deadline

## Problem
`setInterval(fn, 1000)` + `timerRemain--` es el único mecanismo de timing. Navegadores throttlean `setInterval` en tabs de fondo, causando que el timer se atrase silenciosamente.

## Changes (index.html)

### 1. Agregar variable `timerDeadline` (línea 280)
```js
let timerDeadline = 0;
```

### 2. Rewrite `startTimer()` (líneas 381–405)
Reemplazar TODO el cuerpo por:

```js
function startTimer(secs) {
  stopTimer();
  timerDeadline = Date.now() + secs * 1000;
  timerTotal = secs;
  timerRemain = Math.ceil((timerDeadline - Date.now()) / 1000);
  updateDisplay();

  timerInterval = setInterval(() => {
    const prev = timerRemain;
    timerRemain = Math.max(0, Math.ceil((timerDeadline - Date.now()) / 1000));

    if (timerRemain !== prev) {
      updateDisplay();

      if (prev > 10 && timerRemain <= 10) scheduleBeeps(timerRemain);
      else if (prev > 5 && timerRemain <= 5) scheduleUrgent(timerRemain);
      else if (prev > 3 && timerRemain <= 3) scheduleCritical(timerRemain);
    }

    if (timerRemain <= 0) {
      clearInterval(timerInterval);
      timerInterval = null;
      timerRemain = 0;
      updateDisplay();
      document.title = "TIME'S UP · Easy Timer";
      $display.className = 'display alarm';
      $alarmText.classList.add('visible');
      scheduleAlarm();
    }
  }, 100);
}
```

**Qué cambia:**
- `setInterval` pasa de 1000ms a 100ms (solo refresco visual)
- `timerRemain` se calcula desde `timerDeadline - Date.now()` (reloj real)
- Los thresholds de beeps usan `prev > X && timerRemain <= X` para detectar cruce

### 3. Update `stopTimer()` (líneas 407–416)
Agregar reset de `timerDeadline`:

```js
function stopTimer() {
  if (timerInterval) { clearInterval(timerInterval); timerInterval = null; }
  if (audioCtx) { audioCtx.close(); audioCtx = null; }
  timerDeadline = 0;
  timerRemain = 0;
  timerTotal = 0;
  updateDisplay();
  document.title = 'Easy Timer';
  $display.className = 'display';
  $alarmText.classList.remove('visible');
}
```

### 4. Agregar listener `visibilitychange` (después de línea 513, antes del `</script>`)
```js
document.addEventListener('visibilitychange', () => {
  if (!document.hidden && timerInterval && timerDeadline) {
    const prev = timerRemain;
    timerRemain = Math.max(0, Math.ceil((timerDeadline - Date.now()) / 1000));
    if (timerRemain !== prev) updateDisplay();
  }
});
```

Esto actualiza el display inmediatamente al volver a la pestaña, en vez de esperar al próximo tick del interval.

---

## Resumen del comportamiento

| Antes | Después |
|-------|---------|
| `timerRemain--` cada 1s (sin referencia real) | `ceil((deadline - Date.now()) / 1000)` cada 100ms |
| Tabs de fondo: timer se atrasa | Tabs de fondo: el reloj real sigue, el display se actualiza al volver |
| No hay detección de visibilidad | `visibilitychange` refresca display al instante |