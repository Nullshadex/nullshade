---
layout: rf
title: "Decodificación de imágenes NOAA APT"
date: 2024-12-01
frequency: "137 MHz"
type: "Satellite"
description: "Recepción y decodificación de imágenes meteorológicas de satélites NOAA utilizando RTL-SDR"
image: "/assets/images/rf/apt.jpg"
---

# Decodificación de imágenes NOAA APT

Los satélites meteorológicos NOAA transmiten imágenes de la Tierra en tiempo real utilizando el protocolo APT (Automatic Picture Transmission) en las frecuencias de 137 MHz.

## Configuración del hardware

### Equipo utilizado

- **SDR**: RTL-SDR v3 (RTL2832U + R820T2)
- **Antena**: Dipolo de 137 MHz construido artesanalmente
- **Amplificador**: LNA4ALL para mejorar la sensibilidad
- **Software**: SDR# + WXtoImg para decodificación

### Construcción de la antena

Antena dipolo resonante para 137 MHz:
- Longitud total: 109 cm (λ/2)
- Cada brazo: 54.5 cm
- Material: Varilla de cobre de 2mm
- Conector: SO-239 hembra

```bash
# Cálculo de longitud
f = 137.5 MHz (frecuencia central NOAA)
λ = c/f = 300000000/137500000 = 2.18 m
λ/2 = 1.09 m = 109 cm
```

## Satélites NOAA activos

| Satélite | Frecuencia | Horario de paso |
|----------|------------|-----------------|
| NOAA 15  | 137.620 MHz | Órbita polar    |
| NOAA 18  | 137.9125 MHz| Órbita polar    |
| NOAA 19  | 137.100 MHz | Órbita polar    |

## Predicción de pases

Utilizando **Gpredict** para calcular los pases de satélites:

1. Configurar ubicación (coordenadas GPS)
2. Añadir satélites NOAA al tracking
3. Identificar pases con elevación >30°
4. Planificar las capturas

### Configuración Gpredict

```
Ubicación: Madrid, España
Latitud: 40.4168° N
Longitud: 3.7038° W
Altitud: 650m
```

## Proceso de captura

### 1. Configuración SDR#

```
Frecuencia: 137.100 MHz (NOAA 19)
Sample Rate: 1.024 MSPS
Bandwidth: 50 kHz
Gain: 40-45 dB
Audio Output: Virtual Cable
```

### 2. Grabación de audio

Durante el pase del satélite (aproximadamente 10-15 minutos):
- Ajustar frecuencia por efecto Doppler
- Mantener señal centrada
- Grabar audio WAV a 11025 Hz

### 3. Decodificación con WXtoImg

```bash
# Parámetros de decodificación
wxtoimg -e HVC -o noaa19_yyyymmdd_hhmmss.wav imagen.png
```

Opciones:
- `-e HVC`: Mejora de contraste y eliminación de ruido
- `-o`: Combinar ambos canales del APT
- Proyección: Mercator o rectangular

## Resultados obtenidos

### Imagen NOAA 19 - Canal visible

La imagen muestra la península ibérica con excelente definición:
- Resolución: 4 km/pixel
- Cobertura: 2900 km de ancho
- Canales: Visible (0.58-0.68 μm) e infrarrojo (10.3-11.3 μm)

### Análisis de la señal

```
SNR promedio: 25 dB
Duración del pase: 12 minutos
Elevación máxima: 67°
Líneas decodificadas: 1800/2080 (86%)
```

## Mejoras implementadas

### Filtro pasa-banda

Construcción de filtro LC para 137 MHz:
- Atenúa señales FM broadcast (88-108 MHz)
- Mejora relación señal/ruido
- Reduce interferencias de servicios móviles

### Script de automatización

```python
#!/usr/bin/env python3
import subprocess
import datetime
import ephem

def predict_noaa_pass():
    """Predice el próximo pase de NOAA 19"""
    tle_line1 = "1 33591U 09005A   24335.12345678  .00000123  00000-0  45678-4 0  9999"
    tle_line2 = "2 33591  99.1234 123.4567 0012345  67.8901 292.3456 14.12345678123456"
    
    satellite = ephem.readtle("NOAA 19", tle_line1, tle_line2)
    observer = ephem.Observer()
    observer.lat = '40.4168'
    observer.lon = '-3.7038'
    observer.elevation = 650
    
    return satellite.next_pass(observer)

def auto_capture():
    """Automatiza la captura durante el pase"""
    aos, max_elev_time, los = predict_noaa_pass()
    
    # Calcular duración
    duration = (los - aos) * 24 * 3600  # segundos
    
    # Iniciar grabación
    cmd = f"rtl_fm -f 137100000 -s 60000 -g 45 -p 55 - | sox -t raw -r 60000 -e s -b 16 -c 1 - noaa19_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.wav rate 11025"
    
    subprocess.run(cmd, shell=True, timeout=duration)

if __name__ == "__main__":
    auto_capture()
```

## Problemas encontrados

### Interferencias urbanas

En entorno urbano, las principales fuentes de interferencia son:
- Transmisores FM de alta potencia
- Servicios de telecomunicaciones móviles
- Dispositivos ISM (2.4 GHz con armónicos)

**Solución**: Ubicación elevada y filtrado adecuado.

### Efecto Doppler

El desplazamiento Doppler puede alcanzar ±3 kHz:
```
Δf = f₀ × (v/c) × cos(θ)
Δf_max = 137.1 MHz × (7800 m/s / 3×10⁸ m/s) = ±3.6 kHz
```

**Solución**: Seguimiento manual o automático de frecuencia.

## Conclusiones

La recepción de señales NOAA APT es un excelente proyecto para iniciarse en SDR:
- Hardware accesible y económico
- Protocolos bien documentados
- Resultados visuales inmediatos
- Aplicación práctica en meteorología

### Próximos objetivos

1. **METEOR-M**: Satélites rusos con protocolo LRPT (mejor resolución)
2. **GOES**: Satélites geoestacionarios con HRIT/EMWIN
3. **Seguimiento automático**: Control de rotor de antena
4. **Procesado avanzado**: Falso color, realce de características

## Referencias

- NOAA Satellite Operations: https://www.ospo.noaa.gov/
- APT Protocol Specification: NOAA KLM User's Guide
- RTL-SDR Documentation: https://rtl-sdr.com/
- WXtoImg Software: https://wxtoimgrestored.xyz/