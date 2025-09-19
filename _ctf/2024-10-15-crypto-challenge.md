---
layout: ctf
title: "RSA Padding Oracle Attack"
date: 2024-10-15
category: "Crypto"
points: 500
event: "HackTheBox CTF 2024"
description: "Explotación de un oracle de padding en implementación RSA personalizada"
---

# RSA Padding Oracle Attack

Este reto presentaba una implementación personalizada de RSA con padding PKCS#1 v1.5 vulnerable a ataques de oracle de padding.

## Análisis inicial

El servicio proporcionaba las siguientes funcionalidades:
- Cifrado de mensajes con clave pública RSA
- Descifrado que devolvía únicamente si el padding era válido o no
- Acceso a la clave pública (n, e)

```python
# Parámetros proporcionados
n = 0x9a3b2c1d4e5f6789abcdef0123456789...
e = 65537
```

## Identificación de la vulnerabilidad

El oracle de padding permitía distinguir entre:
- Mensajes con padding PKCS#1 v1.5 válido
- Mensajes con padding inválido

Esta diferencia es suficiente para montar un ataque de Bleichenbacher.

## Desarrollo del exploit

### Paso 1: Generación de mensajes conformes

```python
import random
from Crypto.Util.number import *

def generate_conforming_message(oracle, n, e, c):
    """Genera un mensaje que cumple con el formato PKCS#1 v1.5"""
    s = 2
    while True:
        c_new = (c * pow(s, e, n)) % n
        if oracle.is_valid_padding(c_new):
            return s, c_new
        s += 1
```

### Paso 2: Reducción de intervalos

El ataque utiliza la propiedad de que un mensaje conformante debe empezar con `0x00 0x02`.

```python
def narrow_intervals(M, s, n, B):
    """Reduce los intervalos posibles para el mensaje original"""
    M_new = []
    for a, b in M:
        r_min = ceil_div(2 * B + a * s, n)
        r_max = floor_div(3 * B - 1 + b * s, n)
        
        for r in range(r_min, r_max + 1):
            new_a = max(a, ceil_div(2 * B + r * n, s))
            new_b = min(b, floor_div(3 * B - 1 + r * n, s))
            if new_a <= new_b:
                M_new.append((new_a, new_b))
    
    return merge_intervals(M_new)
```

### Paso 3: Implementación completa

```python
def bleichenbacher_attack(oracle, n, e, c):
    """Implementación del ataque de Bleichenbacher"""
    k = (n.bit_length() + 7) // 8
    B = 2 ** (8 * (k - 2))
    
    # Paso 1: Blinding (si es necesario)
    s0, c0 = generate_conforming_message(oracle, n, e, c)
    
    # Intervalo inicial
    M = [(2 * B, 3 * B - 1)]
    i = 1
    
    while True:
        if i == 1:
            # Paso 2a: Buscar el primer s1
            s = ceil_div(n, 3 * B)
            while True:
                c_test = (c0 * pow(s, e, n)) % n
                if oracle.is_valid_padding(c_test):
                    break
                s += 1
        else:
            # Paso 2b o 2c: Buscar siguiente s
            if len(M) > 1:
                # Paso 2b: Múltiples intervalos
                s = s + 1
                while True:
                    c_test = (c0 * pow(s, e, n)) % n
                    if oracle.is_valid_padding(c_test):
                        break
                    s += 1
            else:
                # Paso 2c: Un solo intervalo
                a, b = M[0]
                r = ceil_div(2 * (b * s - 2 * B), n)
                while True:
                    s_min = ceil_div(2 * B + r * n, b)
                    s_max = floor_div(3 * B - 1 + r * n, a)
                    
                    for s in range(s_min, s_max + 1):
                        c_test = (c0 * pow(s, e, n)) % n
                        if oracle.is_valid_padding(c_test):
                            break
                    else:
                        r += 1
                        continue
                    break
        
        # Paso 3: Reducir intervalos
        M = narrow_intervals(M, s, n, B)
        
        # Paso 4: Verificar si terminamos
        if len(M) == 1 and M[0][0] == M[0][1]:
            m = (M[0][0] * inverse(s0, n)) % n
            return long_to_bytes(m)
        
        i += 1

# Funciones auxiliares
def ceil_div(a, b):
    return (a + b - 1) // b

def floor_div(a, b):
    return a // b

def merge_intervals(intervals):
    """Fusiona intervalos solapados"""
    if not intervals:
        return []
    
    intervals.sort()
    merged = [intervals[0]]
    
    for current in intervals[1:]:
        last = merged[-1]
        if current[0] <= last[1] + 1:
            merged[-1] = (last[0], max(last[1], current[1]))
        else:
            merged.append(current)
    
    return merged
```

## Explotación

Con el exploit implementado, fue posible recuperar el mensaje original:

```python
# Conexión al servicio
oracle = PaddingOracle("target.com", 1337)

# Obtener el texto cifrado objetivo
ciphertext = oracle.get_challenge()

# Ejecutar el ataque
plaintext = bleichenbacher_attack(oracle, n, e, ciphertext)
print(f"Flag: {plaintext.decode()}")
```

## Lecciones aprendidas

1. **Implementaciones de padding**: Nunca revelar información sobre la validez del padding
2. **Constancia temporal**: Los errores de padding deben tomar el mismo tiempo que las operaciones exitosas
3. **Uso de estándares**: Utilizar implementaciones probadas como OAEP en lugar de PKCS#1 v1.5

## Referencias

- Bleichenbacher, D. (1998). "Chosen Ciphertext Attacks Against Protocols Based on the RSA Encryption Standard PKCS #1"
- RFC 3447 - PKCS #1: RSA Cryptography Specifications