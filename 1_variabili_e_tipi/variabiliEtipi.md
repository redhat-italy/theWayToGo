# Variabili e tipi in GO

## Materiale consigliato

* Leggere le note ed il codice: https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/variables/README.md
* **[GoPL]**: Capitolo 2.3,2.4,2.5 e 3
* **[GoIA]**: Capitolo 2
* **[safari video]**: Ultimate Go Programming https://www.safaribooksonline.com/library/view/ultimate-go-programming/9780134757476/ugpg_02_02_01_00.html

## Spunti di discussione
* Quando preferire :=, quando var e quando new
* Quali sono le informazioni che un tipo si porta con se?
* Differenze fra casting Java e conversion in Go
* Perché var é l’unico modo per garantire, per tutti i tipi, che una variabile sia inizializzata al suo zero-value?

## Note

### Variabili
Per "creare" una variabile abbiamo 4 modi: var, l'operatore :=, e con i built-in new e make.

* var dichiara una variable e la inizializza allo zero-value di default. Quindi ad esempio per un int è zero, per un bool è false, per un tipo complesso ad esempio una struct inizializza ogni suo membro al suo zero-value di default.
Quindi, ad esempio, scrivere:
```go
var anInt int
```
è equivalente a
```go
anInt := 0
```
Nel secondo caso, il compilatore deduce il tipo della variable dall'assegnazione.
* Il built-in new è uno shortcut per assegnare una variable, inizializzarla e ritornare un puntatore ad essa:
```go
t := new(T)
```
è uno shortcut per:
```go
var temp T
var t = &temp
```
* make è usato invece per creare slice, channel e maps e ritorna il valore, non un puntatore. In pratica, make crea le strutture sullo stack e le ritorna inizializzate per uso immediato.
make e new non sono affatto equivalenti.
Ad esempio:
```go
m := make(map[string]bool)
```
è MOLTO differente da:
```go
p := new(map[string]bool)
```
p in realtà punta ad una nil map, perché new inizializza un puntatore a zero
quindi non si può ad esempio fare:
```go
(*p)["foo"] = true
```
perché andrà in panic.


Giocando un po' con il compilatore e questo file:
```go
package main
import "fmt"

type P struct {
 x, y int
}

func main() {
 var foo int
 var p P

 fmt.Printf("%x\n", foo)
 fmt.Printf("%x\n", p)
}
```
Le Printf ci sono per non far lamentare il compilatore; se si chiede al compilatore di inserire informazioni per disassemblare:
```bash
$ go build -gcflags=“-S -N” test.go
```
e poi facciamo un objdump:
```bash
$ objdump -dS test > test.S
```
e cerchiamo nel disassemblato il main, troviamo questo:
```
   func main() {
      401dd7: 4c 8d 9c 24 d8 fe ff   lea    -0x128(%rsp),%r11
      […altro...]
      401dfd: 48 89 e5                     mov    %rsp,%rbp
      401e00: 41 57                         push   %r15
      401e02: 41 56                         push   %r14
      401e04: 41 55                         push   %r13
      401e06: 41 54                         push   %r12
      401e08: 53                              push   %rbx
      401e09: 48 81 ec f8 00 00 00 sub    $0xf8,%rsp
    var foo int
      401e10: 48 c7 45 c8 00 00 00 movq   $0x0,-0x38(%rbp)
      401e17: 00
    var p P
      401e18: 48 c7 85 e0 fe ff ff     movq   $0x0,-0x120(%rbp)
      401e1f: 00 00 00 00
      401e23: 48 c7 85 e8 fe ff ff     movq   $0x0,-0x118(%rbp)
      401e2a: 00 00 00 00
```
Qui si vede che le variabili foo e p sono messe sullo stack ed inizializzate a zero (le istruzioni movq).

### Uso di new
Sostanzialmente `new` non ha senso di venir usato, come si legge anche su [GoPL]:
> *"The new function is relatively rarely used because the most common unnamed variables are of struct types, for which the struct literal syntax (§4.4.1) is more flexible."*

Rob Pike nel 2010 aveva capito che new non seriva a niente e voleva estendere make per fare quello che oggi fa new. Così da poter utilizzare new per far qualcosa di simile a quello che fanno gli altri linguaggi, siveda [questa discussione](https://groups.google.com/forum/#!topic/golang-nuts/kWXYU95XN04/discussion[1-25]).

### Size dei tipi
la regola dice che size del puntatore = size del word = size dell'int
un int é sempre una word
quindi sul laptop int = int64, sul playground che é amd64p32 é int32
