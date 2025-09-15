---
layout: hardware
title: "Análisis Router ASUS RT-N12"
date: 2025-09-13
category: "Router Analysis"
description: "Análisis de firmware y exploración de vulnerabilidades en router ASUS RT-N12"
image: "/assets/images/asus_router.jpg"
---

# Análisis de un router ASUS (RT-N12E_B1 / RT-N11P)

Este proyecto documenta el proceso de extracción y análisis del firmware de un router doméstico ASUS.  
El objetivo era practicar técnicas de hardware hacking y comprobar cómo se almacenan los parámetros internos en este tipo de dispositivos.

---

## 1. Acceso inicial por UART

El primer contacto se hizo a través del puerto UART del router.  
Con un adaptador **FT232** y un terminal serie se capturó el arranque completo del sistema:

![Conexión UART con FT232](IMG_1.jpg)

Datos observados en consola:
- Bootloader: U-Boot 1.1.3 (2014)
- Kernel: Linux 2.6.36 (MIPS 24Kc)
- Memoria RAM: 32 MB
- Flash SPI: **Winbond W25Q64BV** (8 MB)

Durante el arranque se listaron las particiones MTD:

| Partición | Offset   | Tamaño  | Descripción    |
|-----------|----------|---------|----------------|
| mtd0      | 0x00000  | 192 KB  | Bootloader     |
| mtd1      | 0x30000  | 64 KB   | NVRAM          |
| mtd2      | 0x40000  | 64 KB   | Factory        |
| mtd3      | 0x50000  | ~7 MB   | Kernel+rootfs  |
| mtd4      | 0x12E280 | ~5.6 MB | Rootfs (SquashFS) |

---

## 2. Extracción de la memoria flash

Para garantizar una lectura fiable, se desoldó el chip de memoria SPI y se leyó con un programador externo.

- Chip identificado: **Winbond W25Q64BV**
- Método: desoldado → adaptador SOP8 → programador **XGecu T48**

![Chip Winbond desoldado](IMG_4.jpg)

![Chip soldado al adaptador SOP8](IMG_2.jpg)

![Chip en el programador XGecu](IMG_3.jpg)

Se obtuvieron varias lecturas y se verificó su integridad mediante `sha256sum`.

---

## 3. Análisis del firmware

Con el fichero binario listo, se ejecutó `binwalk`:

```bash
binwalk dump.bin
```

Resultados principales:
- Cadena de versión de U-Boot
- Kernel comprimido con LZMA
- Sistema de ficheros SquashFS v4.0 (XZ)

Extracción del rootfs:

```bash
dd if=dump.bin of=rootfs.img bs=1 skip=1237632
unsquashfs -d squashfs-root rootfs.img
```

Resultado:
- 662 ficheros
- 71 directorios
- 166 symlinks

El rootfs contiene la web de administración y binarios del sistema, pero no incluye las credenciales.
Esas se gestionan en la partición NVRAM.

---

## 4. Análisis de la NVRAM

La NVRAM guarda parámetros como:
- Nombre de la red inalámbrica (SSID)
- Clave de acceso WiFi
- Configuración WAN y LAN
- Opciones de WPS, UPnP y otros servicios

Ejemplo genérico de variables encontradas:

```text
wl0_ssid=MiRedDomestica
wl0_wpa_psk=P4ssw0rdSegura!
wan_proto=dhcp
lan_ipaddr=192.168.1.1
```

---

## 5. Observaciones y reflexiones

- El firmware principal (SquashFS) almacena únicamente binarios y ficheros estáticos.
- Toda la configuración sensible reside en la NVRAM, fácilmente legible una vez se accede a la memoria flash.
- El ataque requiere acceso físico, pero demuestra la fragilidad de dispositivos que no cifran su configuración.
- Una vez extraídos los parámetros, es posible reconstruir completamente el entorno de red del router.

---

## 6. Conclusión

Este ejercicio demuestra cómo con herramientas de bajo coste (programadores, adaptadores, osciloscopio) se pueden recuperar parámetros internos de un router.

Más allá de la práctica técnica, sirve para entender que la seguridad de estos equipos depende tanto del software como de las decisiones de diseño en el hardware.

En resumen: la memoria flash de un router doméstico es una fuente directa de información sensible, y sin medidas de protección adecuadas, su contenido es accesible para cualquiera con el conocimiento y las herramientas adecuadas.