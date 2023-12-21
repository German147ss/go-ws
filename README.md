## Websocket from scratch

### 1. Run a simple web server HTTP


### 2. Opening handshake
```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
```
### 3. To manipulate the HTTP RESPONSE i use Hijacking HTTP Request

### 4. Server Handshake Response
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### Frame

#### Validate 
Esta función validate es responsable de validar un frame WebSocket (fr). A continuación, se detallan cada una de las validaciones que realiza:

Verificación de máscara:

Condición: !fr.IsMasked
Acción en caso de falla: Establece el código de estado ws.status en 1002 y devuelve un error indicando un error de protocolo: "unmasked client frame". Los frames del cliente deben estar enmascarados según el protocolo WebSocket.

Verificación de frames de control:

Condición: fr.IsControl() && (fr.Length > 125 || fr.IsFragment)
Acción en caso de falla: Establece el código de estado ws.status en 1002 y devuelve un error indicando un error de protocolo: "all control frames MUST have a payload length of 125 bytes or less and MUST NOT be fragmented". Los frames de control deben tener una longitud de carga útil de 125 bytes o menos y no deben estar fragmentados.

Verificación de opcode reservado:

Condición: fr.HasReservedOpcode()
Acción en caso de falla: Establece el código de estado ws.status en 1002 y devuelve un error indicando un error de protocolo: "opcode <código de opcode> is reserved". Se detectó un código de opcode reservado según el protocolo WebSocket.

Verificación de RSV reservado:

Condición: fr.Reserved > 0
Acción en caso de falla: Establece el código de estado ws.status en 1002 y devuelve un error indicando un error de protocolo: "RSV <valor de RSV> is reserved". Se detectó un valor de RSV (Reserved Bits) reservado según el protocolo WebSocket.

Verificación de mensaje de texto UTF-8 válido:

Condición: fr.Opcode == 1 && !fr.IsFragment && !utf8.Valid(fr.Payload)
Acción en caso de falla: Establece el código de estado ws.status en 1007 y devuelve un error indicando un error de código: "invalid UTF-8 text message". Se detectó un mensaje de texto UTF-8 no válido.

Verificación de código de cierre:

Condición: fr.Opcode == 8
Sub-condiciones:
Condición: fr.Length >= 2
Acción en caso de falla: Si el código de cierre es mayor o igual a 5000, o es menor que 3000 y no se encuentra en la lista de códigos de cierre predefinidos, establece el código de estado ws.status en 1002 y devuelve un error indicando un código de error específico.
Condición: fr.Length > 2 && !reason
Acción en caso de falla: Si la longitud del código de cierre es mayor que 2 y el motivo no es un texto UTF-8 válido, establece el código de estado ws.status en 1007 y devuelve un error indicando un código de error específico.
Condición: fr.Length != 0
Acción en caso de falla: Si la longitud del código de cierre no es igual a 0, establece el código de estado ws.status en 1002 y devuelve un error indicando un código de error específico.

Si todas las validaciones pasan:

Acción: Devuelve nil, indicando que no se encontraron problemas durante la validación.# go-websocket-from-scratch
