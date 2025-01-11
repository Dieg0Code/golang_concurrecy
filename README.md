# Apuntes sobre la concurrencia en Golang

## Introducción

La concurrencia es la capacidad de un sistema para ejecutar múltiples tareas de manera simultánea. No es lo mismo que hacer varias cosas al mismo tiempo, sino que se refiere a la capacidad de ejecutar una tarea, si esta se demora, se avanzará con otra tarea mientras tanto y así sucesivamente. En Go, la concurrencia se logra a través de las goroutines y los canales.

## Goroutines

Las goroutines son funciones que se ejecutan de manera concurrente. Se crean anteponiendo la palabra clave `go` a la llamada de la función. Por ejemplo:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    go sayHello()
    time.Sleep(1 * time.Second)
}

func sayHello() {
    fmt.Println("Hello, world!")
}
```

En este caso, la función `sayHello` se ejecutará de manera concurrente con la función `main`. La función `main` espera un segundo antes de terminar, porque sino la ejecución del programa terminaría antes de que la goroutine `sayHello` tenga tiempo de ejecutarse. En el mundo real hacer esto de esperar un segundo no es una buena práctica, esto es solo un ejemplo para mostrar que una goroutine tarda un tiempo en ejecutarse y ese tiempo debe ser considerado.

Para manejar el tiempo que tarda un goroutine en ejecutarse de buena manera se pueden usar los `WaitGroup` de la librería `sync`. Por ejemplo:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func printSomething(s string) {
    fmt.Println(s)
}

func main() {

    words := []string{
        "alpha",
        "beta",
        "gamma",
        "delta",
        "phi",
        "zeta",
        "eta",
        "theta",
        "epsilon",
    }

    for i, word := range words {
        go printSomething(fmt.Sprintf("%d: %s", i, word))
    }

    time.Sleep(1 * time.Second)
}
```

Cuando ejecuto este código se imprimen todas las palabras del slice `words` pero no en el orden en el que están declaradas, las funciones concurrentes no se ejecutan en orden secuencial.

### WaitGroup

Los `WaitGroup` de la librería `sync` son una estructura que permite esperar a que todas las goroutines que se han creado terminen de ejecutarse. En vez de usar `time.Sleep` para esperar a que las goroutines terminen, lo cual es malisima práctica, se puede usar un `WaitGroup`. Por ejemplo:

```go
package main

import (
    "fmt"
    "sync"
)

func printSomething(wg *sync.WaitGroup, s string) {
    defer wg.Done() // Defer se ejecuta siempre justo antes de que la función termine. Cerrar el WaitGroup
    fmt.Println(s)
}

func main() {

    var wg sync.WaitGroup // Se crea un WaitGroup


    words := []string{
        "alpha",
        "beta",
        "gamma",
        "delta",
        "phi",
        "zeta",
        "eta",
        "theta",
        "epsilon",
    }

    wg.Add(len(words)) // Se agrega la cantidad de goroutines que se van a ejecutar

    for i, word := range words {
        go printSomething(&wg, fmt.Sprintf("%d: %s", i, word))
    }

    wg.Wait() // Se espera a que todas las goroutines terminen de ejecutarse
}
```

En donde `wg.Add(len(words))` se encarga de agregar la cantidad de goroutines que se van a ejecutar y `wg.Done()` se encarga de indicar que una goroutine ha terminado de ejecutarse. `wg.Wait()` se encarga de esperar a que todas las goroutines terminen de ejecutarse.

Con esto tenemos el control sobre la ejecución de todas las goroutines y nos aseguramos de que todas terminen antes de que el programa termine.

### Test para los WaitGroups

Podemos caer en el error de calcular mal la cantidad de goroutines que se van a spawnear `wg.Add()` y pasarle un 12 por ejemplo cuando en realidad son solo 9 las necesarias para imprimir todas las palabras del slice `words`. Esto nos lleva al error `all goroutines are asleep - deadlock!`. Para evitar este tipo de errores podemos escribir un test para la función `printSomething` que nos asegure que la cantidad de goroutines que se ejecutan es la correcta. Por ejemplo:

```go
package main

import (
    "fmt"
    "sync"
    "testing"
    "os"
    "io"
    "strings"
)

func Test_printSomething(t *testing.T) {
    stdOut := os.Stdout

    r, w, _ := os.Pipe()
    os.Stdout = w

    var wg sync.WaitGroup
    wg.Add(1)

    go printSomething(&wg, "Hello, world!")

    wg.Wait()

    _ = w.Close()

    result, _ := io.ReadAll(r)
    output := string(result)

    os.Stdout = stdOut

    if !strings.Contains(output, "Hello, world!") {
        t.Errorf("Expected 'Hello, world!' but got %s", output)
    }
}
```

En este test se crea un pipe para redirigir la salida estándar a un buffer y poder leerla. Se crea un `WaitGroup` con una goroutine y se espera a que termine. Luego se lee la salida estándar y se verifica que contenga el string "Hello, world!". Si no contiene ese string el test falla.

Con este test nos aseguramos de que la función `printSomething` se ejecute correctamente y que la cantidad de goroutines que se ejecutan es la correcta.

### Desafío

Modificar el siguiente código para que las llamadas a `updateMessage()` se ejecuten como goroutines, implementar `WaitGroups` para que el programa se ejecute correctamente e imprima tres mensajes diferentes. Luego escribir un test para las tres funciones de este programa: `updateMessage()`, `printMessage()` y `main()`.

```go
package main

import (
	"fmt"
)

var msg string

func updateMessage(s string) {
	msg = s
}

func printMessage() {
	fmt.Println(msg)
}

func main() {

	msg = "Hello, world!"

	updateMessage("Hello, universe!")
	printMessage()

	updateMessage("Hello, cosmos!")
	printMessage()

	updateMessage("Hello, world!")

	printMessage()
}
```

```go
package main

import (
    "fmt"
    "sync"
    "testing"
    "os"
    "io"
    "strings"
)

var msg string

func updateMessage(wg *sync.WaitGroup, s string) {
    defer wg.Done()
    msg = s
}

func printMessage() {
    fmt.Println(msg)
}

func main() {
    var wg sync.WaitGroup

    msg = "Hello, world!"

    wg.Add(3)

    go updateMessage(&wg, "Hello, universe!")
    wg.Wait()
    printMessage()
    go updateMessage(&wg, "Hello, cosmos!")
    wg.Wait()
    printMessage()
    go updateMessage(&wg, "Hello, world!")
    wg.Wait()
    printMessage()
}
```

```go
package main

import (
    "fmt"
    "sync"
    "testing"
    "os"
    "io"
    "strings"
)

func Test_updateMessage(t *testing.T) {
    stdOut := os.Stdout

    r, w, _ := os.Pipe()
    os.Stdout = w

    var wg sync.WaitGroup
    wg.Add(1)

    go updateMessage(&wg, "Hello, world!")

    wg.Wait()

    _ = w.Close()

    result, _ := io.ReadAll(r)
    output := string(result)

    os.Stdout = stdOut

    if !strings.Contains(output, "Hello, world!") {
        t.Errorf("Expected 'Hello, world!' but got %s", output)
    }
}
```

```go
package main

import (
    "fmt"
    "sync"
    "testing"
    "os"
    "io"
    "strings"
)

func Test_printMessage(t *testing.T) {
    stdOut := os.Stdout

    r, w, _ := os.Pipe()
    os.Stdout = w

    printMessage()

    _ = w.Close()

    result, _ := io.ReadAll(r)
    output := string(result)

    os.Stdout = stdOut

    if !strings.Contains(output, "Hello, world!") {
        t.Errorf("Expected 'Hello, world!' but got %s", output)
    }
}
```

```go
package main

import (
    "fmt"
    "sync"
    "testing"
    "os"
    "io"
    "strings"
)

func Test_main(t *testing.T) {
    stdOut := os.Stdout

    r, w, _ := os.Pipe()
    os.Stdout = w

    var wg sync.WaitGroup
    wg.Add(3)

    go main()

    wg.Wait()

    _ = w.Close()

    result, _ := io.ReadAll(r)
    output := string(result)

    os.Stdout = stdOut

    if !strings.Contains(output, "Hello, universe!") {
        t.Errorf("Expected 'Hello, universe!' but got %s", output)
    }

    if !strings.Contains(output, "Hello, cosmos!") {
        t.Errorf("Expected 'Hello, cosmos!' but got %s", output)
    }

    if !strings.Contains(output, "Hello, world!") {
        t.Errorf("Expected 'Hello, world!' but got %s", output)
    }
}
```

## Race conditions, Mutexes y una introducción a los channels

### sync.Mutex

`Mutex` quiere decir `mutual exclusion`. Es una estructura que permite que solo un goroutine acceda a una sección de código a la vez. Se usa para evitar `race conditions`. Las `race conditions` son situaciones en las que dos o más goroutines intentan acceder a una dirección de memoria al mismo tiempo y una de ellas intenta modificarla, esto lleva a errores en donde los resultados de x o y operación no es el que se espera ya que en el camino alguna goroutine modificó la dirección de memoria inesperadamente, es decir el valor de una variable.

Los `Mutex` son relativamente fáciles de usar, nos permiten controlar recursos compartidos entre goroutines concurrentes/paralelas. Los `Mutex` permiten bloquear y desbloquear el acceso a un recurso, para que solo una goroutine pueda acceder a él a la vez.

### Channels

Los `channels` son una forma de comunicación entre goroutines. Nos permiten poder comunicar las goroutines entre sí, de esta manera pueden compartir data entre ellas. Es parte de la filosofía de Go el tener cosas que comparten memoria y se comunican entre si, en vez de comunicarse compartiendo memoria. Los `channels` resuelven el problema de producer/consumer, en donde una goroutine produce data y otra la consume.

### Ejemplo de Mutex

```go
package main

import (
    "fmt"
    "sync"
)

var msg string
var wg sync.WaitGroup

func updateMessage(s string, mu *sync.Mutex) {
    defer wg.Done()
    mu.Lock()
    msg = s
    mu.Unlock()
}

func main() {
    msg = "Hello, world!"

    var mu sync.Mutex

    wg.Add(2)
    go updateMessage("Hello, universe!", &mu)
    go updateMessage("Hello, cosmos!", &mu)
    wg.Wait()

    fmt.Println(msg)
}
```

Lo que hacemos es usar un `Mutex` para bloquear el acceso a la variable `msg` y que solo una goroutine pueda acceder a ella a la vez. De esta manera evitamos `race conditions` y nos aseguramos de que el valor de `msg` sea el que esperamos.

#### Testing para encontrar race conditions

```go
package main

import "testing"

func Test_updateMessage(t *testing.T) {
    msg = "Hello, world!"

    wg.Add(2)
    go updateMessage("Hello, universe!")
    go updateMessage("Hello, cosmos!")
    wg.Wait()

    if msg != "Hello, universe!" {
        t.Errorf("Expected 'Hello, universe!' but got %s", msg)
    }
}
```

Lo ejecutamos con:

```bash
go test -race .
```

Si hay `race conditions` el test fallará y nos dirá en qué línea se produjo el error.

Es importante usar el flag `-race` al correr los tests para encontrar `race conditions`, si no usamos esto el test se ejecutará y no encontrará errores aunque haya `race conditions`.

### Un ejemplo mas complejo

```go
package main

import (
    "fmt"
    "sync"
)

var wg sync.WaitGroup

type Income struct {
    Source string
    Amount int
}

func main() {
    // variable for bank balance
    var bankBalance int
    var balanceMutex sync.Mutex

    // print out starting values
    fmt.Printf("Starting bank balance: $%d\n", bankBalance)

    // define weekly revenue
    incomes := []Income {
        {Source: "Main job", Amount: 500},
        {Source: "Side hustle", Amount: 200},
        {Source: "Freelance project", Amount: 50},
        {Source: "Selling stuff online", Amount: 100},
    }
    wg.Add(len(incomes))

    // loop through 52 weeks and print out how much is made; keep a running total
    for i, income := range incomes {
        go func(i int, income Income) {
            defer wg.Done()

            for week := 1; week <= 52, week++ {
                balanceMutex.Lock()
                temp := bankBalance
                temp += income.Amount
                bankBalance = temp
                balanceMutex.Unlock()
                fmt.Printf("On week %d, you earned $%d from %s\n", week, income.Amount, income.Source)
            }
        }(i, income)
    }
    wg.Wait()

    // print out final balance
    fmt.Printf("Final bank balance: $%d\n", bankBalance)
}

```

En este ejemplo es necesario usar un `Mutex` cuando se accede a la variable `bankBalance`. También recordar que es necesario que ejecutemos el código con el flag `-race` para encontrar `race conditions`.

```bash
go run -race .
```

#### Testing para el ejemplo anterior

```go
package main

import (
    "testing"
    "os"
    "io"
)

func Test_main(t *testing.T) {
    stdOut := os.Stdout

    r, w, _ := os.Pipe()

    os.Stdout = w

    main()

    _ = w.Close()

    result, _ := io.ReadAll(r)
    output := string(result)

    os.Stdout = stdOut

    if !strings.Contains(output, "Starting bank balance: $0") {
        t.Errorf("Expected 'Starting bank balance: $0' but got %s", output)
    }
}

```

## Producer-Consumer problem

En computación el problema del productor-consumidor (también conocido como **bounder-buffer problem**) es un clásico problema de sincronización. El problema describe dos procesos, el productor y el consumidor, que comparten un buffer fijo de tamaño limitado. El productor produce datos y los almacena en el buffer, mientras que el consumidor consume los datos almacenados en el buffer. Un Buffer es una estructura de datos que se usa para almacenar datos temporalmente.

Para ilustrar este problema podemos pensar en una pizzería que produce pizzas, este sería nuestro **producer**, luego tenemos nuestros clientes, que pueden ser N los cuales hacen pedidos de pizzas, estos serían nuestros **consumers**. La pizzería tiene un horno que puede cocinar una pizza a la vez, por lo que si hay más de una pizza en espera, estas se almacenan en un buffer. En este caso el buffer sería el horno, que puede almacenar N pizzas en espera.

```go
package main

import (
    "fmt"
    "sync"
    "math/rand"
    "time"
    "github.com/fatih/color"
)

const NumberOfPizzas = 10

var pizzasMade, pizzasFailed, totalPizzas int

type Producer struct {
    data chan PizzaOrder
    quit chan chan error // Un canal de canales de error
}

type PizzaOrder struct {
    pizzaNumber int
    message string
    success bool
}

func (p *Producer) Close() error {
    ch := make(chan error)
    p.quit <- ch
    return <-ch
}

func makePizza(pizzaNumber int) *PizzaOrder {
    pizzaNumber++
    if pizzaNumber <= NumberOfPizzas {
        delay := rand.Intn(5) + 1
        fmt.Printf("Received order #%d!\n", pizzaNumber)

        rnd := rand.Intn(12) + 1
        msg := ""
        success := false

        if rnd < 5 {
            pizzasFailed++
        } else {
            pizzasMade++
        }

        totalPizzas++

        fmt.Printf("Making pizza #%d. It will take %d seconds...\n", pizzaNumber, delay)
        // delay for a bit
        time.Sleep(time.Duration(delay) * time.Second)

        if rnd <= 2 {
            msg = fmt.Sprintf("*** We ran out of ingredients for pizza #%d! ***", pizzaNumber)
        } else if rnd <= 4 {
            msg = fmt.Sprintf("*** The cook quit! We can't make pizza #%d! ***", pizzaNumber)
        } else {
            msg = fmt.Sprintf("Pizza #%d is ready!", pizzaNumber)
            success = true
        }

        p := PizzaOrder {
            pizzaNumber: pizzaNumber,
            message: msg,
            success: success,
        }

        return &p
    }

    return &PizzaOrder{
        pizzaNumber: pizzaNumber,
    }
}

func pizzeria(pizzaMaker *Producer) {
    // keep track of which pizza we are making
    var i = 0
    // run forever or until we receive a quit notification

    // try to make pizzas
    for {
        currentPizza := makePizza(i)
        if currentPizza != nil {
            i = currentPizza.pizzaNumber
            select {
                // we tried to make a pizza (we sent something to the data channel)
                case pizzaMaker.data <- *currentPizza:

                case quitChan := <- pizzaMaker.quit:
                    // close channels
                    close(pizzaMaker.data)
                    close(pizzaMaker.quit)

                    return
            }
        }
    }
}

func main() {
    // seed the random number generator
    rand.Seed(time.Now().UnixNano())

    // print out a message
    color.Cyan("The pizzeria is open for business!")
    color.Cyan("-----------------------------")

    // create a producer
    pizzaJob := &Producer{
        data: make(chan PizzaOrder),
        quit: make(chan chan error),
    }

    // run the producer in background
    go pizzeria(pizzaJob)

    // create and run a consumer
    for i := range pizzaJob.data {
        if i.pizzaNumber <= NumberOfPizzas {
            if i.success {
                color.Green(i.message)
                color.Green("Order #%d is complete!\n", i.pizzaNumber)
            } else {
                color.Red(i.message)
                color.Red("Order #%d failed!\n", i.pizzaNumber)
            }
        } else {
            color.Cyan("Done making pizzas for the day!")
            err := pizzaJob.Close()
            if err != nil {
                color.Red("Error closing the producer: %v", err)
            }
        }
    }
    // print out the end message
    color.Cyan("-----------------------------")
    color.Cyan("The pizzeria is closed for the day!")

    color.Cyan("We made %d pizzas today!, but %d failed with %d attempts", pizzasMade, pizzasFailed, totalPizzas)

    switch {
        case pizzasFailed > 9:
            color.Red("We had a terrible day! We failed to make more than 9 pizzas!")
        case pizzasFailed >= 6:
            color.Yellow("We had a rough day! We failed to make more than 6 pizzas!")
        case pizzasFailed >= 4:
            color.Yellow("We had a decent day! We failed to make more than 4 pizzas!")
        case pizzasFailed >= 2:
            color.Green("We had a good day! We failed to make more than 2 pizzas!")
        default:
            color.Green("We had a great day! We failed to make less than 2 pizzas!")
    }
}
```
