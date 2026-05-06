---
layout: keystpl
permalink: /api/keys.json
title: "Claves Criptográficas"
---
Utilizo GnuPG para firmar mis comunicaciones, algunos commits de código y verificar mi identidad digital.

## Clave Maestra de Certificación (Master Key)

Esta es la raíz de mi identidad digital. Sólo se utiliza para gestionar subclaves y certificar identidades de terceros.

```text
pub   ed25519/159DD2F574404A83 2026-05-06 [C]
      Key fingerprint = 4C5F 203A 799E D157 F641 4B7E 159D D2F5 7440 4A83
uid   [ultimate] Andres Linares <andreslb151@gmail.com>
```

Puede descargar mi clave pública directamente desde [aquí](/assets/keys/andres_linares_pub.asc)

**Nota importante:** Nunca confíe únicamente en el contenido de este sitio web. Verifique el fingerprint a través de canales independientes como mi post oficial en LinkedIn o contacto directo.


### Caducidad
Mi clave maestra no tiene fecha de caducidad. Para algunos puede no ser lo más conveniente pero esto asegura que mi historial de firmas (como commits de Git de hace años) siga siendo verificable indefinidamente sin depender de rotaciones de claves complejas.

## Verificación Externa

Puede verificar la autenticidad de mi identidad en el siguiente canal secundario:

*   [Post de Verificación en LinkedIn](https://www.linkedin.com/posts/andreslb_fingerprint-de-mi-clave-maestra-4c5f-share-7457667888410394624-L1BG)

_Actualizado el 06/05/2026_
