---
title: "Manteniendo la consistencia en el código con Go Vet"
date: 2020-03-17T22:04:41+01:00
tags: ["go"]
canonicalUrl: https://mattermost.com/blog/maintaining-consistency-in-codebases-with-go-vet/
draft: true
---

Mantener con exito un projecto open-source grande es uno de los objetivos clave
de Mattermost. Tenemos cientos de contribuidores y queremos crear un proyecto
que sirva de modelo para la comunidad Go. Dicho esto, seguir los principios
idiomaticos de Go es una de las cosas que más cuidamos para mantener la
consistencia de nuestro codigo. Para esta tarea, nosotros utilizamos `go vet` y
con este post me gustaría explicar como hemos extendido la herramienta para
conseguirlo.

La principal restriccion de `go vet` es que existe un numero limitado de
verdades absolutas sobre que está bien y que está mal, así que `go vet`
comprueba cosas muy generales.

Aunque `go vet` tiene un fantastico set de comprobaciones, tener comprobaciones
especificas para to dominio o incluso para tu empresa es algo inevitable cuando
mantienes un proyecto en nuestra escala. En Mattermost tenemos una manera de
hacer ciertas cosas, como logging, aserciones en los tests or añadiendo la
lisencia en la cabecera de nuestro ficheros codigo fuente. Hacemos un gran
esfuerzo por mantener nuestro codigo consistente a la vez que evitamos
reintroducir patrones antiugos.

Creo que la mejor manera de explicar esto es con un ejemplo.

Hace un tiempo, rediseñamos nuestra implementación de logs, y creo que sería un
gran ejemplo para explicar nuestro trabajo manteniendo la consistencia.
Mientras migrabamos al nuevo diseño (esto no paso de un día para otro),
observamos que el patron antiguo seguia apareciendo en algunos nuevos PRs.
Entre otras cosas, porque el patron antiguo era una de las formas obvias de
presentar datos en los logs.

Ahondemos un poquito mas en este asunto. Voy a intentar explicar lo mejor
posible como extendimos `go vet` para añadir nuestros propias comprobaciones.

Un buen punto de entrada sería la [comprobación](https://github.com/mattermost/mattermost-govet/blob/master/structuredLogging/structuredLogging.go) 
que añadimos para evitar la costrucción de cadenas usando `fmt.Sprintf` como
parte a las llamadas a nuestra biblioteca de logs. Con esta comprobación
implementada fuemos capaces de detectar todos los casos en el codigo donde se
estaba haciendo "not-structured logging" y reemplazarlo con la forma correcta
de hacer structured logging. Tras esto, añadimos la comprobacion a nuestro
sistema de integración continua para asegurannos de que el patron no se
reintroducia de nuevo en el codigo de manera accidental por nosotros o por
algun contribuidor.

Otro ejemplo intereste es como mejoramos la consistencia de las aserciones en
nuestros tests. Nosotros usamos la biblioteca
[Testify](https://github.com/stretchr/testify) para incluir aserciones más
semnaticas, pero al mismo tiempo, estamos usando `t.Fatalf` en algunos lugares.
`t.Fatalf` hace fallar los tests de una manera menos semantica porque el error
del test no tiene la información necesaria relacionada con la asercion. Así que
creamos una [comprobación para evitar el uso de `t.Fatalf`](https://github.com/mattermost/mattermost-govet/blob/master/tFatal/tFatal.go)
en nuestros tests.

Una vez tuvimos esa comprobación, descubrimos que algunas aserciones no estaban
definidas correctamente. Por ejemplo, estabamos usando `require.Equal(t, 5,
len(x))` que es menos semantico que `require.Len(t, x, 5)`. Así que creamos una
[comprobación de aserciones de longitud semanticas](https://github.com/mattermost/mattermost-govet/blob/master/equalLenAsserts/equalLenAsserts.go),
añadiendo en el mensaje de error una sugerencia para reemplazarlo por la
aserción correcta. Seguimos investigando y descubirmos que en algunos sitios
estabamos comprobando `require.Len(t, x, 0)` que se puede escribir de manera
mas semantica como `require.Empty(t, x)`, así que escribimos la comprobación y
de paso incluimos el caso de `require.Equal(t, 0, len(x))` sugiriendo en ambos
casos el uso de `require.Empty`.

Hemos creado otras comprobaciones con otros objetivos, por ejemplo 
Other checks have been made for other purposes, for example checking the
[comprobar la existencia de la lisencia en la cabecera de nuestros ficheros](https://github.com/mattermost/mattermost-govet/blob/master/license/license.go), o
[comprobar la consistencia en el nombre dado a la variable del receptor del metodo para una misma estructura](https://github.com/mattermost/mattermost-govet/tree/master/inconsistentReceiverName).

Extender `go vet` es una tarea facil, solo necesitas algo de conocimiento sobre
el AST de go porque practicamente todo lo demás es gestionado por `go vet`. Pr
ejemplo, vamos implementar una comprobación para encontrar palabras prohibidas
en las cadenas de nuestro codigo.

Lo primero que vamos a necesitar es un `Analyzer`. Un `Analyzer` es la
estructura reponsable de recibir el AST (y algunas otras cosas), encontrar las
cosas que consideramos errores y notificar a `go vet` sobre estos errores.

Construyamos nuestro `Analyzer`

```go
// Fichero: checkwords.go
package main

import (
	"go/ast"
	"go/token"
	"strings"

	"golang.org/x/tools/go/analysis"
	"golang.org/x/tools/go/analysis/unitchecker"
)

var analyzer = &analysis.Analyzer{
	Name: "checkWords",
	Doc:  "check forbidden words usage in our strings",
	Run:  run,
}

func run(pass *analysis.Pass) (interface{}, error) {
	forbiddenWords := []string{
		"bird",
		"water",
		"candy",
	}

	for _, file := range pass.Files {
		ast.Inspect(file, func(node ast.Node) bool {
			switch x := node.(type) {
			case *ast.BasicLit:
				if x.Kind != token.STRING {
					return false
				}
				words := strings.Fields(x.Value)
				for _, word := range words {
					for _, forbiddenWord := range forbiddenWords {
						if word == forbiddenWord {
							pass.Reportf(x.Pos(), "Forbidden word used, please avoid use the word %s on your strings", word)
						}
					}
				}
				return false
			}
			return true
		})
	}
	return nil, nil
}

func main() {
	unitchecker.Main(
		analyzer,
	)
}
```

Nuestro `Analyzer` inspeciona todo los ficheros buscando nodos `*ast.BasicLit`
con el tipo (`Kind`) `token.STRING`, que son las cadenas literales usadas en
nuestro codigo. Tras esto divide las palabras por espacios y comprueba si
alguna de las palabras prohibidas esta siendo usada (Esta aproximación es
completamente basica y no encuentra muchos casos, pero por simplicidad lo voy a
dejar así). Si encuentra alguna de las palabras prohibidas lo reporta al
usuario con un mensaje de error. Go Vet se encarga de mostrar el nombre del
fichero y la linea donde se ha dado el error.

Una vez tenemos nuestro `Analyzer` solo necesitamos registarlo en nuestra
función main para conectarlo con `go vet` usando la función `unitchecker.Main`
(podemos registrar varios `Analyzer`s al mismo tiempo).

Ahora solo necesitamos compilar el codigo con `go build ./checkwords.go` y
usarlo con `go vet -vettool=./checkwords -checkWords ./file-or-module-path`.

Por ejemplo, podemos crear un fichero `example.go` como este:

```go
package main

import "fmt"

func main() {
	fmt.Println("my candy is forbidden!")
	fmt.Println("but other strings are not")
}
```

y ejecutar nuestra comprobación con `go vet -vettool=./checkwords -checkWords
example.go` y la salida sería la siguiente:

```
# command-line-arguments
./example.go:6:14: Forbidden word used, please avoid use the word candy on your strings
```

Y esto es todo lo que necesitamos. Ahora tenemos una forma automatica de
detectar patrones no deseados en nuestro codigo.

En Mattermost hemos estado usando nuestra version durante algunos meses y
nuestra conclusione es que usar extensiones de `go vet` es una gran oportunidad
para mejorar nuetro codigo, definir nuestros propios patrones y mantener la
consistencia. Como parte de nuestra cultura de open source, puedes encontrar
todo nuestro codigo en el repositorio de [mattermost-govet](https://github.com/mattermost/mattermost-govet).
Si te ves a ti mismo preguntando por los mismos cambios una y otra vez en los
PRs, puede que sea una buena oportunidad para escribir tu propia extension de
`go vet` y detectar ese problema automaticamente.

Una vez las comprobaciones estan creadas las puedes aplicar como quieras, tal
vez manualmente de vez en cuando, tal vez como un git hoo, tal vez de manera
obligatoria en tu sistema de integración continua, tal vez una sola vez como
parte de algún cambio grande en el codigo.
