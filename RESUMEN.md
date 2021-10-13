# Sistemas Operativos

## Procesos

- Un proceso es un programa en ejecución (código + datos + pila + registro + file descriptor).
- Un multiprocesador es un computador de varias CPUs (puede ejecutar en paralelo).
- Cuando procesos comparten memoria son threads.
- Si no comparten memoria son procesos pesados.

> MMU: Hardware que traduce direcciones virtuales a reales.

### Preemptive vs Non Preemptive

#### Preemptive (con adelantamiento)

- El núcleo puede quitarle la CPU en cualquier momento.

#### Non Preemptive (sin adelantamiento)

- El proceso decide cuando devolver la CPU.

## nSystem

- Procesos livianos: tasks
- Non preemptive.
- No hay verdadero paralelismo.

### Threads en nSystem

- `nTask nEmitTask (int (*f) (...),...)`: Emite o lanza un thread que ejecuta la función f. Cumple el mismo rol que `pthread_create`.
- `int nWaitTask (nTask t)`: Espera que el thread termine y lo entierra. Cumple el mismo rol que `pthread_join`. Retorna el código de retorno de `t`.
- `void nExitTask (int rc)`: Termina la ejecución del thread que lo invoca. El parámetro rc es el código de retorno.

- Ej Factorial:
```C
#define P 16
double fact (int n) {
    double prod;
    producto(1,n,&prod,p);
    return prod;
}

void producto (int i, int j, double *pres, int p) {
    if (p==1 || j-i < 20) {
        int k;
        double prod = 1.0;
        for (k=1; k<=j; k++) {
            prod* = k
            *pres = prod;
        }
    }
    else {
        double prod1,prod2;
        nTask t = nEmitTask(producto,i,(i+j)/2,&prod1,p/2);
        producto((i+j)/2,&prod2,p - p/2);
        nWaitTask(t);
        *pres = prod1 * prod2;
    }
}
```

## Semáforos

- `nSem nMakeSem(int initickets)`: Retorna un nuevo semáforo cono `initickets` depositados inicialmente.
- `void nWaitSem(nSem s)`: Solicita un ticket al semáforo `s`. Si no tiene tickets disponibles, `nWaitSem` espera hasta que otro thread deposite un ticket en `s`.
- `void nSignalSem(nSem s)`: Deposita un ticket en `s`.

> En nSystem los tickets se otorgan por orden de llegada.


## Problema productor/consumidor

### Versión secuencial

```C
for (;;) {
    Item x = produce();
    consume(x);
}
```

### Versión paralela

Productor:

```C
for (;;) {
    Item x = produce();
    put(x);
}
```

Consumidor:
```C
for (;;) {
    Item x = get();
    consume(x);
}
```
 
-----------------

```C
volatile int count;
Item buf[N];
int nextempty = 0, nextpull = 0, count = 0;
nSem empty;// = nMakeSem(N);
nSem full;// = nMakeSem(0);

void put(Item x) {
    nWaitSem(empty);
    buf[nextempty] = x;
    nextempty = (nextempty + 1) % N;
    nSignalSem(full);
}

Item get() {
    nWaitSem(full);
    x = buff[nextfull]
    nextfull = (nextfull + 1) % N;
    nSignalSem(empty);
}
```

## Cena de filósofos

```c
nSem s[5]; // 5 veces nMakeSem(1)

int filosofo(int i) {
    for (;;) {
        int j = min(i, (i+1)%5);
        int k = max(i, (i+1)%5);
        nWaitSem(j);
        nWaitSem(k);
        comer(j,k);
        nSignalSem(j);
        nSignalSem(k);
        pensar();
    }
}
```

> Min y max evitan deadlock.


## Monitores

Mutex y condición.

### Productor/Consumidor

```c
Item buf[N];
int ne = 0, nf = 0;
int c = 0;
nMonitor m;// = nMakeMonitor();
nCondition nofull, noEmpty;

Item get() {
    nEnter(m);
    while (c==0)
        nWait(m); // nWaitCondition(noEmpty)
    Item x = buf[nf];
    nf = (nf + 1) % N;
    c--;
    nNotifyAll(m); // nSignalCondition(notfull)
    nExit(m);
    return x;
}

void put(Item x) {
    nEnter(m);
    while (c==N)
        nWait(m); //nWaitCondition(notfull)
    buf[ne] = x;
    ne = (ne + 1) % N;
    c++;
    nNotifyAll(m); // nSignalCondition(norEmpty)
    nExit(m);
}
```

### Cena de filósofos

```c
void filosofo(int i) {
    for (;;) {
        pedir(i,(i+1)%5);
        comer();
        devolver(i,(i+1)%5);
        pensar();
    }
}

nMonitor m;
int pals[5];
void pedir(int j, int k) {
    aEnter(m);
    while(pals[j] ||  pals[k])
        nWait(m);
    pals(j) = pals(k) = 1
    nExit(m);
}

void devolver (int j, int k) {
    nEnter(m);
    pals[j] = pals[k] = 0;
    nNotifyAll(m);
    nExit(m);
}
```

### Lectores/Escritores

- Escritores en exclusión mutua con otros escritores y lectores.
- Lectores en paralelo.
- Ej:

```c
char *consultar (char* k) {
    enterRead();
    ...
    exitRead();
}

void definir (char* k, char* v) {
    enterWrite();
    ...
    exitWrite();
}

//// Cambios para hacerlo con orden de llegada (TICKETS)
//// int dist = 0, visor = 0;
int reader = 0;
int writing = FALSE;
nMonitor m; // =nMakeMonitor();

void enterRead() {
    nEnter(m);
    //// int minum = dist++;
    while (writing) { //// minum != visor
        nWait(m);
    }
    readers++; //// visor++;
    //// nNotifyAll(m);
    nExit(m);
}

void exitRead() {
    nEnter(m);
    readers--;
    if (readers == 0)
        nNotifyAll(m);
    nExit(m);
}

void enterWrite() {
    nEnter(m); 
    //// int minum = dist++;
    while (readers > 0 || writing) ///// readers > 0 || minum != visor
        nWait(m);
    writing = TRUE; //// BORRAR
    nExit(m);
}

void exitWrite() {
    nEnter(m);
    writing = FALSE; ///// visor++;
    nNotifyAll(m);
    nExit(m);
}
```

### Lectores / escritores EFICIENTE en cambios de contexto y evitar hambruna

```c
typedef struct {
    nMonitor m;
    FifoQueue q;
    int readers, writing;
} Ctrl;

typedef struct {
    int kindM // Reader or Writer
    nCondition w;
    int ready;
} Request;
```

```c
char *query (Dict *d, char *s) {
    enterRead(d->c); // c es Ctrl*
    ... // consulta
    exitRead(d->c);
    ...
}

void enterRead (Ctrl *c) {
    nEnter(c->m);
    if (c->writing || !EmptyFifoQueue(c->q))
        await(c,READER);
    c->readers++;
    wakeup(c);
    nExit(c->m);
}

void await (Ctrl *c, int k) {
    Request r = {k,nMakeCondition(c->m),FALSE};
    PutObj(c->q, &k);
    while (!r.ready)
        nWaitCondition(r.w);
    nDestroyCondition(r.w);
}

void exitRead (Ctrl *c) {
    nEnter(c->m);
    if (c->readers == 0)
        wakeup(c);
    nExit(c->m);
}

void enterWrite (Ctrl *c) {
    nEnter(c->m);
    if (readers > 0 || c->writing || !EmptyFifoQueue(c->q))
        await(c,WRITER);
    c->writing = TRUE;
    nExit(c->m);
}

void exitWrite (Ctrl *c) {
    nEnter(c->m);
    c->writing = FALSE;
    wakeup(c->m);
}

void wakeup (Ctrl *c) {
    Request *p = GetObj(c->q);
    if (p == NULL)
        return
    if (p->kind == WRITER && c->readers == 0 && !c->writing)
        p->ready = TRUE;
        nSignalCondition(p->w);
    else if (p->kind == READER && !c->writing)
        p->ready = TRUE;
        nSignalCondition(p->w);
    else
        PutObj(c->q,p);
}
```


## Administración de procesos

- `ready_queue`: Cola de scheduling.
- Cambio de contexto: Traspaso de un core de un proceso a otro.
- Cambios de contexto explícito: Proceso cede voluntariamente el core.
- Cambios de contexto implícito: Scheduler quita el core.
- Procesos intensivos en CPU: Ráfagas largas (mucho READY).
- Procesos intensivos en E/S: Ráfagas cortas (mucho WAIT).
- Ráfaga se ejecuta entre WAIT. Es decir no considera estados de READY.
- Tiempo de despacho: Desde que llega a la cola hasta que se despacha la ráfaga.

### Estrategias de scheduling

1. FCFS: First come - first served.
    - Non preemptive.
    - Simple.
    - No sirve en sistemas interactivos.
    - Minimiza cambios de contexto implícitos.
    - Ráfagas se atienden por orden de llegada.

2. Shortest Job First.
    - Trata de atender primero ráfagas cortas.
    - Non preemptive.
    - Preemptive: quitar la CPU cuando ráfaga excede un límite.

3. Colas de prioridad.
    - Procesos poseen prioridad.
    - Se elige el proceso de mejor prioridad.
    - SJF es un caso (**Prioridad dinámica**).
    - Asignar prioridad fija (**Prioridad estática**).
    - SFJ puede generar hambruna, solución: Aging (*añejar*).
        - Cada cierto tiempo mejora la prioridad de los procesos en espera.
        - Cuando toma la CPU, recuperan su prioridad original.

4. Round Robin.
    - Se da tajadas de tiempo de CPU a cada proceso (*slice*).


### Implementación semáforos

```c
nTask current_task;
Queue readu_queue;
int current_slice;

void Resume() {
    nTask esta = current_task;
    nTask prox = GetTask(ready_queue);
    current_task = prox;
    changeContext(esta,prox);
    // acá no // current_task = esta;
}

typedef struct {
    int c;
    Queue q;
} *nSem;

void nWaitSem (nSem s) {
    START_CRITICAL();
    if (s->c > 0)
        s->c --;
    else {
        current_task -> status = WAIT_SEM;
        PutTask(s -> q, current_task);
        Resume();
    }
    END_CRITICAL();
}

void nSignalSem (nSem s) {
    START_CRITICAL();
    if (EmptyQueue(s -> q))
        s->c++;
    else {
        nTask w = GetTask(s -> q);
        w -> status = READY;
        PushTask(ready_queue,w);
        Resume();
    }
    END_CRITICAL();
}
```

```c
void Vtimerhandler() {
    setAlarm(virtualtimer,currentslice,vTimerHandler);
    PutTask(ready_queue,current_task);
    Resume();
}

```

-----

## Funciones

### Tareas

```c
nTask nEmitTask (int (*f) (...),...); // Emite tarea - recibe función, args

nWaitTask(t); // Espera tarea - recibe tarea
```

### Semáforos
```c
nSem empty = nMakeSem(N); // Crea un semáforo de N tickets

nSignalSem(full); // Agrea un ticket a un semáforo

nWaitSem(empty); // Espera el ticket de un semáforo

nDestroySem(sem); // Destruye semáforo
```

### Monitor / Condiciones

```c
nMonitor m = nMakeMonitor(); // Crea un monitor

nEnter(m); // Entra al monitor

nExit(m); // Libera al monitor

nWait(m); // Espera un monitor

nNotifyAll(m); // Notifica a todos los que esperan un monitor

nDestroyMonitor(mon); // Destruye monitor

/////////////////////////////////////

nCondition nofull = nMakeCondition(monito); // Crea una condición, recibe un monitor

nSignalCondition(notEmpty); // Notifica a una condición

nWaitCondition(noEmpty); // Espera una condición

nDestroyCondition(cond); // Destruye condición
```

### Mensajes

```c
int nSend(task,msg); Envía un mensaje a una tarea

nTask* emisor;
void* nReceive(emisor,timeout); // Retorna un mensaje y guarda quien se lo envió

int nReply(emisor,rc); // Envía respuesta rc a emisor
```

### Queues

#### Fifo

```c
FifoQueue queue = nMakeFifoQueue(); // Crea una fifo queue

GetObj(queue); // Obtiene un objeto de la queue

PutObj(queue, &obj); // Agrega un objeto a la queue

EmptyFifoQueue(queue); // Revisa si la queue está vacía

```

#### Queue

```c
Queue q = MakeQueue(); // Crea una queue

GetTask(queue); // Obtiene una tarea de la queue

EmptyQueue(queue); // Revisa si una queue está vacía

PutTask(queue,task); // Agrega tarea a queue

PushTask(queue,task); // Agrega tarea a queue
```

### Otros

```c
START_CRITICAL(); // inhibe interrupciones

END_CRITICAL(); // permite interrupciones

ResumeNextReadyTask(); // Resume tarea de ready_queue

nTask current_task; // Tarea actual

```

-----


