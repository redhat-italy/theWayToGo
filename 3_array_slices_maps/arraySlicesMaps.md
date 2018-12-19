# Array, Slices e Maps in GO

## Materiale consigliato

* **[GoPL]**: Capitolo 4.1 4.2 4.3
* **[GoIA]**: Capitolo 4
* **[safari video]**: Ultimate Go Programming 
  * https://www.safaribooksonline.com/videos/ultimate-go-programming/9780134757476/9780134757476-ugpg_02_03_00_00
  * https://www.safaribooksonline.com/videos/ultimate-go-programming/9780134757476/9780134757476-ugpg_02_03_01_00
  * https://www.safaribooksonline.com/videos/ultimate-go-programming/9780134757476/9780134757476-ugpg_02_03_02_00
  * https://www.safaribooksonline.com/videos/ultimate-go-programming/9780134757476/9780134757476-ugpg_02_03_03_01
  * https://www.safaribooksonline.com/videos/ultimate-go-programming/9780134757476/9780134757476-ugpg_02_03_04_00
* Approfondimenti:
  * https://www.youtube.com/watch?v=Tl7mi9QmLns

## Spunti di discussione
* Data structures e mechanical simpathy
* Discutere le perf di questo snippet di codice: https://github.com/ardanlabs/gotraining/tree/master/topics/go/testing/benchmarks/caching
* Perché in golang non ci sono altre strutture dati (tree, ecc.)
* Le slice finiscono sullo stack o sull'heap?

## Note
Gli array sono un'area contigua di memoria di dimensione fissa e nota a compile time uguale a size elemtno*numero elementi

Le slice formalmente sono composte da 3 parti:
- un puntatore ad un array (detto backing array): punta all area di memoria dove risiedono effettivamente i dati.
- capacity: esprime la lunghezza totale della slice
- length: esprime la parte utilizzata della slice
non e' possibile accedere elementi oltre la length, la capacity puo' essere maggiore o uguale alla length, ma non viceversa.

Generlamente non ha molto senso lavorare con puntatori a slice, un'eccezione puo' essere se si fa uso di append in una funzione; qui' si anno due possibilita': passare un puntatore o ritornare la slice.
La funzione append infatti potrebbe non riallocare o riallocare il backing array, quindi conviene sempre pensare che potrebbe aver modificato la tua slice per questo l'uso idiomatico di append dovrebbe essere:
```go
myslice = append(myslice, anotherslice)
```

E' possibile fare un pezzo di codice che abbia un array nello stack?
```go
package main

import (
	"fmt"
)

func main() {
	var arrayOnStack [1]int
	uselessCall(arrayOnStack)
}

func uselessCall(array [1]int){
	array[0]=1
}
```
se si salva il codice soprastante in un file `example.go` e lo si compila con `go build -gcflags '-m' example.go` si vede che `arrayOnStack` non subisce escape dalla funzione `main()` e quindi viene allocata sullo stack.

Per quanto riguarda gli spunti di discussione:
* Data structures e mechanical simpathy:
per quanto riguarda i dati, nelle moderne architetture di cpu le performance dipendono in gran parte dall'utilizzo delle cache di primo e secondo livello, terzo se proprio... per far questo serve che il proprio codice utilizzi regioni contigue di memoria con un patter di accesso predicibile di modo che la cpu ti metta sempre a disposizione i dati che ti servono in cache.
* Discutere le perf di questo snippet di codice: https://github.com/ardanlabs/gotraining/tree/master/topics/go/testing/benchmarks/caching :
il sunto del discorso qui' e' appunto far vedere in pratica i principi trattati sopra, si crea una grossa matrice sia come array di array che come linked list e la si traversa per riga, per colonna e ovviamente sequenzialmente per la linked list: vince l'attraversamento per riga, secondo la linked list, terzo quello per colonna. la differenza tra linked list e per colonna sta nel fatto che il size della matrice e' fatto in modo che due righe distinte stiano in pagine di memoria del sistema operativo diverse (ovviamente dipende dall'architettura, l'esempio ha come target il go playground), in questo modo si ha un degrado ancora maggiore delle performance in quanto la linked list verosimilmente non stara' contiguissima ma comunque nella stessa pagina di memoria, mentre attraversando per colonna si ha anche la penalita' del miss della pagina di memoria del SO
* Perché in golang non ci sono altre strutture dati (tree, ecc.):
l'idea credo sia quella di spingere, in prima battuta ad usare ove possibile quelle in quanto piu' performanti, in molto casi anche piu' performanti di algoritmi migliori su altre strutture dati meno simpatetiche con l'hardware
* Le slice finiscono sullo stack o sull'heap?:
Le slice e il backing array seguono le normali regole di posizionamento in memoria che semplificando molto e: se la lunghezza e' nota a compile time, se non sono troppo grandi e non sono frutto di escape stanno sullo stack, altrimenti sull heap. Per "grandi" si intende, secondo quanto riportato nel commento a https://golang.org/src/runtime/malloc.go, sopra i 32k.
