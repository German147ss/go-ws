# Servidor WebSocket en Go

## Introducción

Los WebSockets son una tecnología fundamental para habilitar la comunicación bidireccional en tiempo real entre clientes y servidores. Este servidor WebSocket en Go proporciona una implementación eficiente y fácil de usar para permitir dicha comunicación persistente.

## Uso Básico

### Inicio del Servidor

El servidor se inicia mediante el uso de un manejador HTTP que escucha en un puerto específico (en este caso, el puerto 9001). El código es sencillo y fácil de entender:

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", WsHandle)
	log.Fatal(http.ListenAndServe(":9001", nil))
}
```

En este punto, `WsHandle` es el manejador principal que gestionará las conexiones WebSocket.

### Manejo de Conexiones WebSocket

La función `WsHandle` gestiona las conexiones WebSocket. Realiza el handshake inicial y establece un bucle para manejar los frames WebSocket entrantes y salientes:

```go
func WsHandle(w http.ResponseWriter, r *http.Request) {
    // Crear una nueva conexión WebSocket
	ws, err := New(w, r)
	if err != nil {
		log.Println(err)
		return
	}

    // Realizar el handshake inicial
	err = ws.Handshake()
	if err != nil {
		log.Println(err)
		return
	}

    // Asegurarse de cerrar la conexión al finalizar
	defer ws.Close()

    // Bucle para manejar frames WebSocket
	for {
		frame, err := ws.Recv()
		if err != nil {
			log.Println("Error Decoding", err)
			return
		}

        // Procesar el frame según su opcode
		switch frame.Opcode {
		case 8: // Close
			return
		case 9: // Ping
			frame.Opcode = 10
			fallthrough
		case 0, 1, 2: // Continuation, Text, Binary
			// Procesar y enviar el frame de vuelta
			if err = ws.Send(frame); err != nil {
				log.Println("Error sending", err)
				return
			}
		}
	}
}
```

### Estructura del Frame

El servidor opera principalmente con "frames", unidades de datos en el protocolo WebSocket. La estructura `Frame` encapsula la información clave de un frame:

```go
type Frame struct {
    IsFragment bool
    Opcode     byte
    Reserved   byte
    IsMasked   bool
    Length     uint64
    Payload    []byte
}
```

Estos frames son la base para enviar y recibir mensajes entre el cliente y el servidor.

# Operaciones sobre los Frames

## Estructura del Frame (`Frame`)

La estructura `Frame` encapsula la información necesaria para representar un frame en el protocolo WebSocket, basándose en la definición de la [sección 5 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-5). Cada propiedad desempeña un papel crucial en la interpretación y manipulación de los frames.

```go
type Frame struct {
    IsFragment bool
    Opcode     byte
    Reserved   byte
    IsMasked   bool
    Length     uint64
    Payload    []byte
}
```

### Propiedades del Frame:

- **`IsFragment`**: Un indicador que señala si el frame es parte de un mensaje más grande. Si es `true`, indica que es un fragmento y hay más frames por venir.

- **`Opcode`**: Representa el tipo de operación que el frame está llevando a cabo. Los valores posibles están definidos en la [sección 11.8 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-11.8).

- **`Reserved`**: Tres bits reservados para usos futuros. En la especificación actual, deben ser cero, pero podrían ser utilizados en extensiones futuras del protocolo.

- **`IsMasked`**: Un indicador que señala si el payload del frame está enmascarado. Los frames que viajan desde el cliente al servidor deben estar enmascarados, según la [sección 5.3 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-5.3).

- **`Length`**: La longitud del payload del frame. Puede ser de 7, 7+16 o 7+64 bits, dependiendo del valor de los bits de longitud en el encabezado del frame, según la [sección 5.2 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-5.2).

- **`Payload`**: La carga útil real del frame, que contiene los datos que se están transmitiendo.

## Operaciones sobre Frames

### Obtener un Pong (`Pong`)

```go
func (f Frame) Pong() Frame {
    f.Opcode = 10
    return f
}
```

La operación `Pong` convierte el frame actual en un frame Pong. En el protocolo WebSocket, un Pong se utiliza como respuesta a un Ping y es una confirmación de que la conexión está viva, según la [sección 5.5.3 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-5.5.3).

### Obtener el Texto del Payload (`Text`)

```go
func (f Frame) Text() string {
    return string(f.Payload)
}
```

La operación `Text` interpreta el payload del frame como texto y lo devuelve como una cadena de caracteres. Esto es útil cuando el frame transporta datos de texto, según la [sección 5.6 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-5.6).

### Verificar si es un Frame de Control (`IsControl`)

```go
func (f *Frame) IsControl() bool {
    return f.Opcode&0x08 == 0x08
}
```

La operación `IsControl` verifica si el frame es un frame de control. Los frames de control tienen el bit más significativo del opcode establecido en 1, según la [sección 5.5 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-5.5).

### Verificar si tiene un Opcode Reservado (`HasReservedOpcode`)

```go
func (f *Frame) HasReservedOpcode() bool {
    return f.Opcode > 10 || (f.Opcode >= 3 && f.Opcode <= 7)
}
```

La operación `HasReservedOpcode` verifica si el opcode del frame está en el rango de opcodes reservados según la especificación del protocolo WebSocket, como se define en la [sección 11.8 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-11.8).

### Obtener el Código de Cierre (`CloseCode`)

```go
func (f *Frame) CloseCode() uint16 {
    var code uint16
    binary.Read(bytes.NewReader(f.Payload), binary.BigEndian, &code)
    return code
}
```

La operación `CloseCode` interpreta el payload del frame como un código de cierre. Esto es relevante para frames de cierre de conexión, según la [sección 5.5.1 de la RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455#section-5.5.1).

Estas operaciones están diseñadas para estar en conformidad con la [RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455), que define el protocolo WebSocket. Para obtener detalles adicionales y comprender completamente estas operaciones, se recomienda consultar la especificación original.