# Auditoría de seguridad WPA2/WPS en dos redes wifi domésticas

## Qué hace este proyecto

Pruebas de penetración contra dos redes wifi reales, con autorización en ambos casos:
1. Mi propia red en Málaga (router Xiaomi/MiWiFi, dos bandas).
2. La red de mis padres en San Pedro de Alcántara (router distinto, otro operador), con **permiso explícito de mi madre** antes de empezar.

Hecho desde Kali Linux (Live USB) arrancado en mi Asus Vivobook Go 15, con la tarjeta wifi interna (MediaTek MT7921e).

## Qué quería comprobar

Si cada red era vulnerable a los dos ataques más conocidos contra WPA2/WPS:
1. **Captura del handshake WPA2** y crackeo offline por diccionario.
2. **WPS Pixie Dust** (sacar el PIN WPS sin fuerza bruta online, y con él la contraseña).

## Red 1: Málaga (mi router, Xiaomi/MiWiFi, ambas bandas)

### Handshake WPA2
Deauth con `aireplay-ng` (broadcast y dirigido a un cliente concreto) para forzar una reconexión y capturar el handshake con `airodump-ng`.

**Resultado: no se capturó en ninguna de las dos bandas.** Dos intentos distintos, mismo resultado — señal bastante clara de que el router tiene **PMF (802.11w)** activo, que hace que el cliente ignore los paquetes de deauth falsos.

PMKID (alternativa que no depende de deauth) no se pudo probar: la herramienta (`hcxdumptool`) no venía en esta imagen Live reducida. `wifite2` (automatizado) tampoco dio resultado, pero por un bug de compatibilidad conocido entre el driver `mt7921e` y el modo monitor, no por una defensa del router.

### WPS Pixie Dust
`wash` confirmó WPS activo. Lanzado `reaver -K` (Pixie Dust). El router respondía a M1/M2 pero cortaba la negociación con deauth antes de M3 (el mensaje con los datos que necesita `pixiewps` para calcular el PIN).

**Resultado: no completado**, y además el propio intento dejó el WPS bloqueado (`WPS Locked: Yes`). Confirmado con `wash` después: el bloqueo se mantuvo, así que no se pudo ni reintentar con `bully` para comparar herramientas.

### Conclusión red 1
Dos defensas distintas bloquearon cada ataque antes de llegar a ningún resultado explotable: PMF para el handshake, lockout de WPS tras detectar el patrón de ataque para Pixie Dust.

## Red 2: San Pedro (router de mis padres, con permiso)

### Handshake WPA2
Mismo proceso (modo monitor, `airodump-ng` + deauth). **Resultado: handshake capturado a la primera**, confirmado con `aircrack-ng` (`1 handshake` junto al BSSID).

### Crackeo por diccionario
`aircrack-ng` contra el `.cap` con `rockyou.txt`. Cancelado a media ejecución: la contraseña de esta red es la de fábrica (mezcla aleatoria de mayúsculas y minúsculas), y ese tipo de contraseña tiene un espacio de combinaciones demasiado grande para un diccionario — no es un fallo del ataque, es la razón por la que las contraseñas de fábrica son más seguras que las que se inventa la gente.

**Resultado: handshake capturado, pero no crackeado.** Conclusión útil para la memoria: capturar el handshake no es lo mismo que crackearlo.

### WPS Pixie Dust y variantes — varios intentos, resultados mixtos
`wash` mostró WPS activo y **no bloqueado**. Varios intentos con `reaver` a lo largo de la sesión, con comportamientos distintos cada vez:

- **1er intento:** por error apuntado primero a una red vecina — detectado antes de completar nada y corregido para apuntar solo a la red propia (`R-240kb`).
- **2º intento (ya contra la red correcta):** negociación llegó a M1/M2, el router volvió a mandar M1 en vez de avanzar, `reaver` envió NACK y la transacción abortó (código `0x03`) → `Nothing done, nothing to save`, sin PIN.
- **3er intento:** esta vez el router ni siquiera empezó la negociación WPS — se asociaba bien a nivel de red pero ignoraba los paquetes EAP por completo (bucle de `Sending EAPOL START request` → timeout). Comportamiento distinto al del segundo intento: puede ser el firmware del router no respondiendo a un registrador externo, o incompatibilidad con `reaver`.
- **Limitación de hardware detectada:** una prueba de inyección de paquetes (`--test`) contra la propia red dio solo ~70% de éxito con la tarjeta interna del Asus — no al 100%. Con mensajes tan delicados como M2/Auth/Assoc, perder 3 de cada 10 paquetes basta para romper la negociación antes de completarse.
- **4º intento (con flags `-N -L` para evitar cortes por NACK/lockout falso):** esta vez sí hubo comunicación EAP real, pero `reaver` se quedó en bucle repitiendo el mismo PIN (`12345670`) sin avanzar, consistente con ~30% de pérdida de paquetes rompiendo la negociación cada vez. Parado aquí como conclusión razonable.

**Resultado: WPS no completado tampoco en esta red**, pero por un motivo distinto y menos limpio que en Málaga: una mezcla de comportamiento del router (no siempre responde igual al registrador) y limitación de hardware (tarjeta interna sin inyección al 100%, en vez de un adaptador USB dedicado tipo Alfa). No se llegó a probar `bully` en esta red.

### Hallazgo de investigación (no explotado)
Al identificar el router, `reaver` informó inicialmente el vendor como "Broadcom" — pero investigando la MAC y el hostname (`liveboxfibra.home`) el fabricante real es **Sagemcom** (Livebox Fibra, típico de Orange), no Broadcom/Movistar como sugería el SSID "MiFibra". Lección aparte: el SSID no dice de quién es el router de verdad, el vendor de la MAC sí.

Durante la búsqueda apareció una vulnerabilidad real y reciente, **WPSplinter** (hecha pública el 16/10/2026): un fallo de 16 años en código WPS de Broadcom que afecta a routers de varios fabricantes e ISPs, con al menos un caso confirmado en un Sagemcom (BBOX3V+, de otro operador). **No se intentó explotar**: no hay herramienta pública para reproducirlo, es un fallo de corrupción de memoria, y afectaría potencialmente a cualquier router con ese chip, no solo al de esta prueba. Se queda documentado como hallazgo defensivo: si el modelo exacto resulta estar entre los afectados, la recomendación pública es contactar con el operador y considerar poner el router en modo bridge con uno propio detrás.

## Comparación final

| | Red 1 (Málaga, Xiaomi) | Red 2 (San Pedro, Sagemcom/Orange) |
|---|---|---|
| Handshake WPA2 | No capturado (PMF activo) | Capturado a la primera |
| Crackeo diccionario | — | No (contraseña de fábrica, aleatoria) |
| WPS Pixie Dust | No completado (cortado antes de M3, WPS bloqueado después) | No completado (resultados distintos según intento; limitado también por hardware) |
| Motivo principal del fracaso | Defensas del router (PMF + lockout WPS) | Mezcla de comportamiento del router y limitación de la tarjeta wifi interna |

## Qué he aprendido

- PMF y el lockout de WPS no son checkboxes decorativos: en la red 1 bloquearon en la práctica los dos ataques más comunes, incluso bien ejecutados.
- Capturar el handshake no es lo mismo que crackearlo: una contraseña de fábrica aleatoria hace inviable el diccionario aunque el ataque de captura funcione perfectamente.
- Un resultado "no funcionó" necesita diagnóstico, no solo registro: en la red 2 hubo tres motivos de fallo distintos (protocolo cortado, router sin responder, hardware con pérdida de paquetes) y hay que distinguirlos para que la conclusión sea honesta.
- El SSID de una red no identifica al fabricante real del router; el vendor de la MAC sí.
- Para hacer esto en serio (sin los límites de una tarjeta interna) haría falta un adaptador wifi USB dedicado con inyección completa (tipo Alfa AWUS036).

## Qué queda pendiente (fuera de este proyecto, por ahora)

- Probar `bully` contra la red 2 cuando se pueda repetir con mejor hardware.
- Confirmar el PMF de la red 1 de forma directa (bajarla temporalmente a WPA2 puro, comprobar, y revertir).
- Repetir con un adaptador wifi USB externo en vez de la tarjeta interna.

## Aviso legal

Red 1: mi propia red, mi propio router. Red 2: red de mis padres, con permiso explícito y verificado antes de empezar. No se ha atacado ninguna red ni dispositivo de terceros, y la vulnerabilidad WPSplinter encontrada durante la investigación no se ha intentado explotar en ningún momento.
