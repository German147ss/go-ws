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

### Operaciones sobre los Frames

Se proporcionan operaciones comunes sobre los frames, como obtener un frame Pong, extraer el texto del payload y verificar si es un frame de control:

```go
func (f Frame) Pong() Frame {
    f.Opcode = 10
    return f
}

func (f Frame) Text() string {
    return string(f.Payload)
}

func (f *Frame) IsControl() bool {
    return f.Opcode&0x08 == 0x08
}

func (f *Frame) HasReservedOpcode() bool {
    return f.Opcode > 10 || (f.Opcode >= 3 && f.Opcode <= 7)
}

func (f *Frame) CloseCode() uint16 {
    var code uint16
    binary.Read(bytes.NewReader(f.Payload), binary.BigEndian, &code)
    return code
}
```

Estas operaciones facilitan el manejo de diferentes tipos de frames y su interpretación.

### Configuración del Servidor

El paquete `Ws` proporciona funcionalidades para el manejo de conexiones WebSocket. La creación de una nueva conexión, el handshake, la lectura y escritura de frames, y la validación se gestionan en esta estructura:

```go
type Ws struct {
    conn   Conn
    bufrw  *bufio.ReadWriter
    header http.Header
    status uint16
}

func New(w http.ResponseWriter, req *http.Request) (*Ws, error) {
    // Crear una nueva conexión WebSocket
	hj, ok := w.(http.Hijacker)
	if !ok {
		return nil, errors.New("webserver doesn't support http hijacking")
	}
	conn, bufrw, err := hj.Hijack()
	if err != nil {
		return nil, err
	}
	return &Ws{conn, bufrw, req.Header, 1000}, nil
}

func (ws *Ws) Handshake() error {
    // Real

izar el handshake inicial
	hash := getAcceptHash(ws.header.Get("Sec-WebSocket-Key"))
	lines := []string{
		"HTTP/1.1 101 Web Socket Protocol Handshake",
		"Server: go/echoserver",
		"Upgrade: WebSocket",
		"Connection: Upgrade",
		"Sec-WebSocket-Accept: " + hash,
		"", // required for extra CRLF
		"", // required for extra CRLF
	}
	return ws.write([]byte(strings.Join(lines, "\r\n")))
}

func (ws *Ws) write(data []byte) error {
    // Escribir datos en la conexión WebSocket
	if _, err := ws.bufrw.Write(data); err != nil {
		return err
	}
	return ws.bufrw.Flush()
}

func (ws *Ws) read(size int) ([]byte, error) {
    // Leer datos de la conexión WebSocket
	data := make([]byte, 0)
	for {
		if len(data) == size {
			break
		}
		// Temporary slice to read chunk
		sz := bufferSize
		remaining := size - len(data)
		if sz > remaining {
			sz = remaining
		}
		temp := make([]byte, sz)

		n, err := ws.bufrw.Read(temp)
		if err != nil && err != io.EOF {
			return data, err
		}

		data = append(data, temp[:n]...)
	}
	return data, nil
}

func (ws *Ws) validate(fr *Frame) error {
    // Validar un frame WebSocket
    // ...
}

func (ws *Ws) Recv() (Frame, error) {
    // Recibir datos de la conexión WebSocket y construir un frame
    // ...
}

func (ws *Ws) Send(fr Frame) error {
    // Enviar un frame a través de la conexión WebSocket
    // ...
}

func (ws *Ws) Close() error {
    // Enviar un frame de cierre y cerrar la conexión TCP
    // ...
```

Estas operaciones son esenciales para gestionar la conexión WebSocket y garantizar la integridad de los datos intercambiados.