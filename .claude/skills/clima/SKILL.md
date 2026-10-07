---
name: clima
description: Consulta el clima actual y el pronóstico de una ciudad (por defecto Lima, Perú) usando wttr.in y Open-Meteo, sin API keys. Úsala cuando el usuario pregunte por el clima, la temperatura, si va a llover, el pronóstico, o invoque /clima. Acepta opcionalmente otra ciudad como argumento.
argument-hint: "[ciudad]"
allowed-tools: Bash(curl:*)
---

# Clima

Obtén el clima con `curl` desde servicios públicos y gratuitos (no requieren API key) y responde **en español**.

Argumento recibido: `$ARGUMENTS`

## 1. Determinar la ubicación

La ubicación por defecto es **siempre Lima, Perú** (`CIUDAD` = `Lima,Peru`, lat `-12.04318`, lon `-77.02824`, zona horaria `America/Lima`).

- Si `$ARGUMENTS` está vacío o no menciona ninguna ciudad, usa Lima, Perú. No intentes deducir otra ubicación por IP, por el contexto de la conversación ni por consultas anteriores.
- Si `$ARGUMENTS` trae una ciudad, úsala. Reemplaza los espacios por `+` en la URL (ej. `San+Isidro`, `Ciudad+de+Mexico`).
- Si el usuario escribe solo "Lima" (o un distrito de Lima, como Miraflores o San Isidro, sin otro país), asume que es en Perú: usa `Lima,Peru` o `<Distrito>,Lima,Peru`, nunca Lima de otro país.
- Si la ciudad no existe o no se encuentra, dilo y ofrece mostrar el clima de Lima, Perú.

## 2. Consultar wttr.in (fuente principal)

Clima actual en una línea:

```bash
curl -s --max-time 10 "https://wttr.in/CIUDAD?format=%l:+%c+%C,+%t+(sensaci%C3%B3n+%f),+humedad+%h,+viento+%w,+lluvia+%p&lang=es&m"
```

Pronóstico de hoy y los próximos 2 días (JSON compacto):

```bash
curl -s --max-time 10 "https://wttr.in/CIUDAD?format=j1&lang=es" 
```

Del JSON usa: `current_condition[0]` (`temp_C`, `FeelsLikeC`, `humidity`, `windspeedKmph`, `lang_es[0].value`), `nearest_area[0]` (`areaName`, `country`) y `weather[]` (`date`, `mintempC`, `maxtempC`, y en `hourly[]` el máximo de `chanceofrain`). Si el JSON es muy largo, extrae solo esos campos en lugar de leerlo entero.

## 3. Respaldo: Open-Meteo

Si wttr.in falla, tarda o responde algo que no es clima (ej. "Unknown location", HTML, o un mensaje de servicio caído):

1. Geocodifica la ciudad (para Lima, Perú puedes saltarte este paso y usar `latitude=-12.04318`, `longitude=-77.02824`):
   ```bash
   curl -s "https://geocoding-api.open-meteo.com/v1/search?name=CIUDAD&count=1&language=es"
   ```
2. Consulta el clima con la latitud/longitud obtenida:
   ```bash
   curl -s "https://api.open-meteo.com/v1/forecast?latitude=LAT&longitude=LON&current=temperature_2m,apparent_temperature,relative_humidity_2m,wind_speed_10m,weather_code&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max&timezone=auto&forecast_days=3"
   ```
   Para Lima usa `timezone=America/Lima` en lugar de `auto`:
   ```bash
   curl -s "https://api.open-meteo.com/v1/forecast?latitude=-12.04318&longitude=-77.02824&current=temperature_2m,apparent_temperature,relative_humidity_2m,wind_speed_10m,weather_code&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max&timezone=America/Lima&forecast_days=3"
   ```
3. Traduce `weather_code` (WMO): 0 despejado · 1–3 parcialmente nublado/nublado · 45, 48 niebla · 51–57 llovizna · 61–67 lluvia · 71–77 nieve · 80–82 chubascos · 95–99 tormenta.

## 4. Formato de la respuesta

Responde breve y en español, con unidades métricas:

```
📍 <Ciudad, País>
Ahora: <condición>, <temp> °C (sensación <temp> °C)
Humedad <h>% · Viento <v> km/h

Pronóstico:
- Hoy:     <min>–<max> °C, lluvia <p>%
- Mañana:  <min>–<max> °C, lluvia <p>%
- Pasado:  <min>–<max> °C, lluvia <p>%
```

Si el usuario hizo una pregunta concreta (ej. "¿llevo paraguas?"), respóndela directamente en una línea antes del bloque. Indica qué fuente usaste solo si fue el respaldo. Si ambas fuentes fallan, dilo claramente y no inventes datos.
