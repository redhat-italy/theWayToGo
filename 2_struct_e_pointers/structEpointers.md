# Struct e Pointers in GO

## Materiale consigliato

* Leggere le note ed il codice:
  * https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/struct_types/README.md
  * https://github.com/ardanlabs/gotraining/tree/master/topics/go/language/pointers
* **[GoPL]**: Capitolo 4.1 4.2 4.3 4.4
* **[GoIA]**: Capitolo 5.1, 5.2 5.3
* **[safari video]**: Ultimate Go Programming 
  * https://www.safaribooksonline.com/library/view/ultimate-go-programming/9780134757476/ugpg_02_02_02_00.html
  * https://www.safaribooksonline.com/library/view/ultimate-go-programming/9780134757476/ugpg_02_02_03_01.html
  * https://www.safaribooksonline.com/library/view/ultimate-go-programming/9780134757476/ugpg_02_02_03_02.html
  * https://www.safaribooksonline.com/library/view/ultimate-go-programming/9780134757476/ugpg_02_02_03_03.html
  * https://www.safaribooksonline.com/library/view/ultimate-go-programming/9780134757476/ugpg_02_02_03_04.html
  * https://www.safaribooksonline.com/library/view/ultimate-go-programming/9780134757476/ugpg_02_02_03_05.html
* Approfondimenti:
  * https://www.goinggo.net/2017/05/language-mechanics-on-stacks-and-pointers.html
  * https://www.goinggo.net/2013/07/understanding-pointers-and-memory.html
  * https://www.goinggo.net/2017/05/language-mechanics-on-escape-analysis.html
  * https://github.com/golang/go/blob/release-branch.go1.9/src/runtime/stack.go#L64-L82
  * https://github.com/ardanlabs/gotraining/blob/master/topics/go/language/struct_types/README.md
  * https://docs.google.com/document/d/1CxgUBPlx9iJzkz9JWkb6tIpTe5q32QDmz8l0BouG0Cw/edit?usp=sharing
  
## Spunti di discussione
* Struct Padding, spiegare quale é il modo migliore di ottimizzare l’uso della memoria (e perché). 
* Anonymous types e structs, quando si possono assegnare e quando no variabili di tipi struct diversi, e perché
* A che servono i puntatori in go? Quali sono i program boundaries?
* Pass by value/pass by reference
* Stack and heap, dove e come vengono allocate le variabili, come decide (se puó) il programmatore dove allocare?

## Note

### Stack vs Heap
Lo stack in go, e stack vs heap. 
In go c'è uno stack per goroutine (c'é sempre una goroutine, quella del main se non ce ne sono atre esplicite), anziché uno stack per processo. 
Fino a, credo, la versione 1.2 di go, lo stack era gestito in maniera segmentata (split stacks): in pratica una lista doppiamente collegata di stack segment. 
Si partiva da un segmento, se la goroutine necessitava di più stack si allocava un nuovo segmento, quando la goroutine non necessitava più di una certa parte dello stack lo stack veniva ristretto. 
Il segmento iniziale (e quello di crescita) era di 8kb credo nella 1.2

Il problema che fu riscontrato è l'overhead del processo di allocazione e rilascio ("hot splits")

Dalla 1.3 go implementa il contiguous stack. In pratica, per ogni goroutine viene allocato uno stack iniziale di 2kb. 
Se la goroutine necessita di più stack, viene allocato un nuovo stack ed il vecchio viene copiato sul nuovo. 
Lo stack è sempre contiguo e non in segmenti. Per rendere il processo più agile ed efficiente, il nuovo stack viene allocato in potenze di 2. 
In pratica, si parte da 2kb, poi vengono allocati 4kb, poi 8kb etc.

Ad ogni funzione viene assegnato un pezzo dello stack (stack frame), che ha una dimensione nota a compile time (potenzialmente diverso per ogni stack frame). 
In questo modo go sa cosa puó gestire sullo stack e cosa no per ogni funzione

Se date una occhiata a https://golang.org/src/runtime/stack.go si trova il seguente codice:

```go
 // StackSystem is a number of additional bytes to add
   // to each stack below the usual guard area for OS-specific
   // purposes like signal handling. Used on Windows, Plan 9,
   // and Darwin/ARM because they do not use a separate stack.
   _StackSystem = sys.GoosWindows*512*sys.PtrSize + sys.GoosPlan9*512 + sys.GoosDarwin*sys.GoarchArm*1024
  
   // The minimum size of stack used by Go code
   _StackMin = 2048
  
   // The minimum stack size to allocate.
   // The hackery here rounds FixedStack0 up to a power of 2.
   _FixedStack0 = _StackMin + _StackSystem
   _FixedStack1 = _FixedStack0 - 1
   _FixedStack2 = _FixedStack1 | (_FixedStack1 >> 1)
   _FixedStack3 = _FixedStack2 | (_FixedStack2 >> 2)
   _FixedStack4 = _FixedStack3 | (_FixedStack3 >> 4)
   _FixedStack5 = _FixedStack4 | (_FixedStack4 >> 8)
   _FixedStack6 = _FixedStack5 | (_FixedStack5 >> 16)
   _FixedStack  = _FixedStack6 + 1
```
Per capire la questione stack vs heap, usiamo il seguente codice:
```go
type PS struct {
 A, B int
}

func func1() (*PS) {
 var pp *PS = new(PS)
 return pp
}

func func2() (*PS) {
 var p PS
 return &p
}

func main() {
 p1 := func1()
 p2 := func2()
 fmt.Printf("p1 = %x\np2 = %x\n", p1, p2)
}
```

In go, come si sa, è perfettamente ammissibile ritornare una variable locale ad una funzione. Se andiamo a vedere il codice generato per func1 e func2, troveremo questo:
```go
func func1() (*PS) {
  [...]
var pp *PS = new(PS)
  401e04: be 10 00 00 00        mov    $0x10,%esi
  401e09: bf a0 39 40 00        mov    $0x4039a0,%edi
  401e0e: e8 ed fb ff ff        callq  401a00 <__go_new@plt>
  401e13: 48 89 45 f0           mov    %rax,-0x10(%rbp)
 return pp
  401e17: 48 8b 45 f0           mov    -0x10(%rbp),%rax
  401e1b: 48 89 45 f8           mov    %rax,-0x8(%rbp)
  401e1f: 48 8b 45 f8           mov    -0x8(%rbp),%rax
  401e23: c9                    leaveq 
  401e24: c3                    retq   


func func2() (*PS) {
  [...]
  401e52: be 10 00 00 00        mov    $0x10,%esi
  401e57: bf a0 39 40 00        mov    $0x4039a0,%edi
  401e5c: e8 9f fb ff ff        callq  401a00 <__go_new@plt>
  401e61: 48 89 45 f0           mov    %rax,-0x10(%rbp)
  401e65: 48 8b 45 f0           mov    -0x10(%rbp),%rax
  401e69: 48 89 45 f8           mov    %rax,-0x8(%rbp)
  401e6d: 48 8b 45 f8           mov    -0x8(%rbp),%rax
  401e71: c9                    leaveq 
  401e72: c3                    retq
```

Come si vede, entrambe le funzioni ritornano un puntatore ad un oggetto sull'heap, come testimoniano le chiamate a `__go_new`. Questo perché il compilatore ha effettato una escape analysis e ha capito che le variabili locali devono sopravvivere anche all'uscita della funzione e devono quindi vivere sull'heap.

## Puntatori

In go esiste solo il pass by value: si copia il valore func(value) oppure il puntatore ad un valore func(&value) esattamente come in Java

Usare i puntatori serve solo per fare sharing di valori fra i program boundaries invece che copiarli:
function calls
goroutines

