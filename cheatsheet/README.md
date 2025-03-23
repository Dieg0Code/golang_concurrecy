# 🚀 Cheatsheet de Concurrencia en Go

## 📊 Conceptos Fundamentales

| Concepto         | Definición                             | Analogía                                            |
| ---------------- | -------------------------------------- | --------------------------------------------------- |
| **Concurrencia** | Gestionar múltiples tareas en progreso | Chef cocinando varios platos alternando entre ellos |
| **Paralelismo**  | Ejecutar tareas simultáneamente        | Varios chefs trabajando en diferentes platos        |

## 🧵 Goroutines

```go
// Goroutine básica
go func() {
    // Código que se ejecuta concurrentemente
}()

// Con argumentos
go func(x int) {
    fmt.Println(x)
}(42)
```

## 🔄 Herramientas de Sincronización

### WaitGroup

```go
var wg sync.WaitGroup

wg.Add(n)    // Añade n goroutines para esperar
wg.Done()    // Señaliza que la goroutine ha terminado
wg.Wait()    // Bloquea hasta que todas las goroutines terminen

// Patrón
func worker(wg *sync.WaitGroup) {
    defer wg.Done()  // Siempre se llama cuando la función retorna
    // Trabajo...
}
```

### Mutex

```go
var mu sync.Mutex
var rwmu sync.RWMutex  // Mutex de Lector/Escritor

// Patrón básico
mu.Lock()
// Sección crítica - modificar datos compartidos
mu.Unlock()

// Mejor práctica con defer
mu.Lock()
defer mu.Unlock()
// Sección crítica...

// RWMutex para cargas de trabajo con lectura intensiva
rwmu.RLock()  // Múltiples lectores pueden adquirir el lock
// Leer datos compartidos
rwmu.RUnlock()

rwmu.Lock()   // Acceso exclusivo para escritura
// Escribir datos compartidos
rwmu.Unlock()
```

### Channels

```go
// Creación
ch := make(chan int)        // Sin buffer
ch := make(chan string, 5)  // Con buffer (capacidad 5)

// Operaciones
ch <- valor     // Enviar (bloquea si el canal está lleno)
valor := <-ch   // Recibir (bloquea si el canal está vacío)
valor, ok := <-ch  // Comprobar si el canal está cerrado (ok es false si está cerrado)

// Cerrar
close(ch)  // Señala que no se enviarán más valores

// Iteración
for valor := range ch {  // Itera hasta que el canal se cierre
    // Procesar valor
}

// Restricciones de dirección
func productor(out chan<- int) {}  // Solo envío
func consumidor(in <-chan int) {}  // Solo recepción
```

### Select

```go
select {
case v1 := <-ch1:
    // Manejar valor de ch1
case ch2 <- v2:
    // Manejar envío exitoso en ch2
case <-time.After(1 * time.Second):
    // Timeout después de 1 segundo
case <-done:
    // Manejar cancelación
default:
    // Caso no bloqueante (opcional)
}
```

## 🧩 Patrones Comunes

### Pool de Workers

```go
func workerPool(numWorkers int, jobs <-chan Job, results chan<- Result) {
    var wg sync.WaitGroup
    wg.Add(numWorkers)

    for i := 0; i < numWorkers; i++ {
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }

    wg.Wait()
    close(results)
}
```

### Fan-out/Fan-in

```go
func fanOut(input <-chan int, n int) []<-chan int {
    outputs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        outputs[i] = worker(input)
    }
    return outputs
}

func fanIn(inputs []<-chan int) <-chan int {
    output := make(chan int)
    var wg sync.WaitGroup

    wg.Add(len(inputs))
    for _, ch := range inputs {
        go func(ch <-chan int) {
            defer wg.Done()
            for val := range ch {
                output <- val
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(output)
    }()

    return output
}
```

### Pipeline

```go
func generador(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func cuadrado(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// Uso
nums := generador(1, 2, 3, 4)
cuadrados := cuadrado(nums)
for n := range cuadrados {
    fmt.Println(n)
}
```

### Timeouts

```go
select {
case res := <-ch:
    // Manejar resultado
case <-time.After(2 * time.Second):
    // Manejar timeout
}
```

### Context para Cancelación

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // Asegura que todas las rutas cancelen el contexto

// Con timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Pasar a goroutine
go worker(ctx, ...)

// En la función worker
select {
case <-ctx.Done():
    return ctx.Err()
case result <- resultadoProcesado:
    return nil
}
```

## ⚠️ Problemas Comunes

### Race Conditions

```go
// INCORRECTO - Race condition
counter++

// CORRECTO - Usando mutex
mu.Lock()
counter++
mu.Unlock()

// CORRECTO - Usando channels
counterCh <- 1  // Señal de incremento
```

### Deadlocks

```go
// INCORRECTO - Causará deadlock en la goroutine principal
ch := make(chan int)
ch <- 42  // No hay receptor

// CORRECTO
ch := make(chan int)
go func() { ch <- 42 }()
value := <-ch
```

### Fugas de Goroutines

```go
// INCORRECTO - Goroutine con fuga si ch nunca recibe un valor
go func() {
    val := <-ch  // Puede bloquearse para siempre
    // Procesar val
}()

// CORRECTO - Usando select con timeout o cancelación
go func() {
    select {
    case val := <-ch:
        // Procesar val
    case <-ctx.Done():
        return  // Salir cuando el contexto es cancelado
    case <-time.After(timeout):
        return  // Salir después del timeout
    }
}()
```

## 🔍 Herramientas de Depuración

```bash
# Detector de race conditions
go run -race main.go
go test -race ./...

# Herramienta de traza
go tool trace trace.out

# Perfilado
import _ "net/http/pprof"
go tool pprof http://localhost:6060/debug/pprof/profile
```

## 💡 Mejores Prácticas

1. **Comparte memoria comunicándote**, no comuniques compartiendo memoria
2. **Pasa WaitGroups por referencia**, no por valor
3. **No permitas fugas de goroutines** - siempre proporciona una forma de salida
4. **Minimiza las secciones críticas** - bloquea solo cuando sea necesario
5. **El propietario del canal cierra** el canal, no los receptores
6. **Usa canales con buffer** cuando conozcas la capacidad exacta necesaria
7. **Prefiere context para cancelación** en escenarios complejos
8. **Usa el detector -race** regularmente durante el desarrollo

## 🚨 Gotchas

1. Enviar a un canal cerrado causa pánico
2. Cerrar un canal más de una vez causa pánico
3. Recibir de un canal nil bloquea para siempre
4. Recibir de un canal cerrado devuelve el valor cero
5. Los canales sin buffer sincronizan el emisor y el receptor
6. `for range` en canales se detiene después de que el canal se cierre

Recuerda: _"La concurrencia no es paralelismo. La concurrencia permite el paralelismo."_ - Rob Pike
