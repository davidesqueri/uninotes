#uni 
# Nella Pratica
Le informazioni possono essere inviate in due modi:
1. modalità testuale
2. modalità binaria: ovvero trattare il messaggio da inviare come una stream di byte.
La modalità binaria può funzionare se i due processi vivono su due macchine con la stessa endianess e con la stessa dimensione dei tipi predefiniti.
Altrimenti è necessario portare il dato in endianess di rete prima della send().
## TCP
A TCP non importa nulla dei tuoi "messaggi" o "array". Lui vede solo un flusso infinito di byte. Garantisce che i byte arrivino tutti e nell'ordine giusto, ma **non garantisce in quanti blocchi arriveranno**. Ovvero quando il ricevente fa `recv()` non è detto che legga tutto il messaggio che il mittente ha inviati: qualche blocco potrebbe essere ancora a giro per la rete, per questo è necessario delimitare il messaggio inviato: in modalità testuale per esempio posso terminare il messaggio con `\0` – in modalità binaria questo non funziona poiché un byte a caso potrebbe essere erroneamente considerato come un `\0` – oppure comunicare in anticipo al destinatario la dimensione del messaggio che stiamo inviando.

Buona pratica:
```c
size_t sent = 0, totale;
while (sent < totale) {
	ssize_t sent_now = send(fd, buf + sent, len - sent, FLAGS);
	if (sent_now < 0)
		/// errore
	sent += sent_now;
}
```
### Chiusura connessione
Se l'altro processo esegue una `close()`, quando esegui una `ssize_t n = recv()` ottieni `FIN`, ovvero `n = 0` (EOF sul socket).

Se l'altro processo abortisce senza `close()`, quando esegui una `ssize_t n = recv()` ottieni `RST`, ovvero `n = -1` e viene settata la variabile globale `errno` (NON a `EAGAIN` o `EWOULDBLOCK`).

Se esegui una `ssize_t n = recv()` su un socket non bloccante quando non ci sono dati disponibili, `n=-1` e `errno` viene settata a `EAGAIN` o `EWOULDBLOCK`: questo **non** significa che la connessione è chiusa, solo che non ci sono dati pronti.
## UDP
Ragiona a "pacchetti chiusi". Se fai una `send()` di 100 byte, dall'altra parte la `recv()` leggerà esattamente 100 byte in un colpo solo, oppure fallirà. I confini del messaggio sono mantenuti.
# librerie
```c
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
```
# Cooperazione tra processi
- Due processi possono cooperare attraverso:
	- Sincronizzazione (Es. semafori)
	- Comunicazione, cioè scambio di informazioni
		- Memoria condivisa
		- Chiamate a procedura remota
		- Scambio di messaggi
- due processi possono cooperare
	- sulla stessa macchina
	- su macchine diverse (sistema distribuito)
		- come avviene la comunicazione? tramite scambio di messaggi per cui è stata definita l'astrazione dei socket
# Socket
Un socket è un meccanismo per la comunicazione tra processi, sulla stessa macchina o su macchine differenti.

Un socket è identificato da un indirizzo:
- indirizzo host: TCP/IP: indirizzo IP
- indirizzo processo: TCP/IP: numero di porta

Un socket è un'estremità di un canale di comunicazione

È un'astrazione implementata dal sistema operativo, che ci rende disponibile delle primitive (system calls) per:
– Creare un socket
– Assegnargli un indirizzo e una porta
– Connettersi ad un altro socket
– Accettare una connessione
– Inviare e ricevere dati attraverso i socket

## strutture per gli indirizzi
```c
/* Endpoint IPv4in netinet/in.h */
struct sockaddr_in {
	sa_family_t sin_family; /* address family: AF_INET */
	in_port_t sin_port; /* port in network byte order */
	struct in_addr sin_addr; /* internet address */
};
/* Internet address in netinet/in.h */
struct in_addr {
	uint32_t s_addr; /* address in network byte order */
};
```
- Attenzione: esiste anche la struttura struct sockaddr usata da alcune funzioni descritte più avanti ma non la useremo direttamente.
## Formato degli indirizzi
- formato numerico (n): 32 bit usato dal computer
- formato presentazione(p): stringa in notazione decimale puntata
```c
int inet_pton(int af, const char *src, void *dst);
```
- af: (address family) famiglia (AF_INET)
- src: stringa del tipo "ddd.ddd.ddd.ddd"
- dst: puntatore a una struct in_addr

```c
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```
- af: famiglia (AF_INET)
- src: puntatore a una struct in_addr
- dst: puntatore a un buffer di caratteri lungo size
- size: deve valere almeno INET_ADDRSTRLEN
## Endianess

### big endian
Per primo il MSB
indirizzo piccolo - byte grande

> ==LA RETE USA BIG ENDIAN==
### little endian
Per primo il LSB
Indirizzo piccolo - byte piccolo

### funzioni di conversione
```c
#include <arpa/inet.h>
uint32_t htonl(uint32_t hostlong); // host to network long
uint16_t htons(uint16_t hostshort); // host to network short
uint32_t ntohl(uint32_t netlong); // network to host long
uint16_t ntohs(uint16_t netshort); // network to host short
```

> unit16_t e uint32_t sono definiti nell'[[Header file]] `<stdint.h>`
## Primitive
### socket()
```c
#incldue <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

int socket(int domain, int type, int protocol);
```
- domain: famiglia di protocolli da utilizzare
	- AF_LOCAL : comunicazione locale
	- AF_INET : protocolli IPv4, TCP e UDP
- type:
	- SOCK_STREAM: TCP
	- SOCK_DGRAM: UDP
- protocol: sempre a $0$ 
- la funzione restituisce:
	- un descrittore di file che rappresenta il socket e servirà a manipolare il socket attraverso altre primitive
	- $-1$ se essore
nota che il socket non è ancora associato ad un indirizzo IP o ad una porta.
### close()
Chiude un socket: non può più essere usato per inviare o ricevere dati.
```c
#include <unistd.h>
int close(int fd);
```
- int fd: descrittore del processo
- restituisce 0 se ha successo, -1 su errore

> chi chiama la close() manda zero all'altro endpoint e la connessione viene chiusa: l'host remoto riceverà 0 dalla recv()
### Lato server
#### bind()
Assegna un socket ad un indirizzo (IP+porta). Di solito il cliente non ha bisogno di eseguire una bind().
```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
- sockfd: descrittore del socket (socket file descriptor)
- addr: puntatore alla struttura di tipo struct sockaddr
	- Visto che usiamo struct sockaddr_in bisogna convertire il puntatore
- addrlen: dimensione di addr
- La funzione restituisce 0 se ha successo, -1 su errore

```c
ret = bind(sd, (struct sockaddr*)&my_addr, sizeof(my_addr));
```
#### listen()
Imposta il socket in modalità passiva, di ascolto. Il socket verrà usato per ricevere richieste di connessione. Si possono mettere in attesa solo i socket SOCK_STREAM.
```c
int listen(int sockfd, int backlog);
```
- sockfd: descrittore del socket
- backlog: dimensione della coda, ovvero quante richieste da client possono rimanere in attesa
- restituisce 0 se ha successo, -1 altrimenti
- *non bloccante* 

La richiesta (da parte di un client) di connessione al socket del server arriva al sistema operativo:
- Il kernel la mette in attesa in una coda, finché il server non chiama accept() per prenderla in carico.
- Se arrivano più richieste contemporanee, vengono accodate.
#### accept()
Accetta una richiesta di connessione pervenuta sul socket (solo su SOCK_STREAM).
```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
- sockfd: descrittore socket
- addr: puntatore ad una struttura (vuota) di tipo struct sockaddr, ==inizializzare una nuova struttura e inserirla nella accept, sarà la sockaddr del client che si connette== 
- addrlen: dimensione di addr
- restituisce il descrittore di un nuovo socket che verrà usato per la comunicazione, -1 se errore
- questa funzione è **bloccante**, il programma si ferma finché non arriva una richiesta
```c
struct sockaddr_in cl_addr;
int len = sizeof(cl_addr);
new_sd = accept(sd, (struct sockaddr*)&cl_addr, &len);
```
### Lato Client
#### Connect()
Connette il socket locale ad un socket remoto.
```c++
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
- sockfd: descrittore del socket locale
- addr: puntatore alla struttura contenente l'indirizzo del server
- addrlen
- restituisce 0 se ha successo, -1 se errore
- questa funzione è **bloccante**: il programma si ferma finché la richiesta di connessione non è stata accettata
```c
ret = connect(sd, (struct sockaddr*) &sv_addr, sizeof(sv_addr));
```
### Scambio dati
#### Send()
invia un messaggio attraverso un socket **connesso**
```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```
- sockfd
- buf: puntatore al buffer contenente il messaggio da inviare
- len: dimensione in byte del messaggio
- flags: per settare opzioni, noi lo lasciamo a **0**
- la funzione restituisce il numero di byte inviati se ha successo, altrimenti -1
- la funzione è **bloccante**: il programma si ferma finché non ha inviato tutto il messaggio
> send() non spedisce pacchetti: copia i dati nel buffer del kernel, che poi li frammenta in pacchetti TCP
#### recv()
Preleva un messaggio da un socket **connesso**.
```c
ssize_t recv(int sockfd, const void *buf, size_t len, int flags);
```
- sockfd
- buf: puntatore al buffer in cui salvare il messaggio
- len (in byte) del messaggio
- flags
	- MSG_WAITALL
- restituisce il numero di byte ricevuti, -1 per errore, 0 se il socket remoto si è chiuso
- funzione **bloccante**: si ferma finché non ha letto qualcosa
### Gestione degli errori
Le primitive viste restituiscono -1 quando concludono in errore.
Inoltre settano una variabile, **errno** (in `errno.h`), che può essere letta per scoprire il motivo dell'errore.
Nel manuale di ogni funzione c'è l'elenco degli errori possibili.
```c
#include <errno.h>
//…
ret = bind(sd, (struct sockaddr*)&my_addr, sizeof(my_addr));
	if (ret == -1) {
		if (errno == EADDRINUSE) {/* Gestisci errore */}
		if (errno == EINVAL) {/* Gestisci errore */}
		//…
}
```

> a volte vogliamo solo sapere l'errore e uscire: `perror("Error: ");` legge *errno* e stampa l'errore su schermo in forma leggibile
#### Errori possibili
| Valore | Nome costante      | Significato                      | Quando capita tipicamente                                              |
| ------ | ------------------ | -------------------------------- | ---------------------------------------------------------------------- |
| 9      | EBADF              | Bad file descriptor              | Il socket non è valido o è stato già chiuso.                           |
| 13     | EACCES             | Permission denied                | Porta privilegiata (<1024) senza privilegi root.                       |
| 98     | EADDRINUSE         | Address already in use           | `bind()` su una porta già occupata.                                    |
| 99     | EADDRNOTAVAIL      | Cannot assign requested address  | IP non valido o non configurato sull'host.                             |
| 111    | ECONNREFUSED       | Connection refused               | Il server non è in ascolto sulla porta indicata.                       |
| 113    | EHOSTUNREACH       | No route to host                 | Nessuna rotta verso l’host remoto.                                     |
| 110    | ETIMEDOUT          | Connection timed out             | Nessuna risposta dal server entro il timeout.                          |
| 32     | EPIPE              | Broken pipe                      | Scrittura (`send`) su un socket chiuso dall’altro lato.                |
| 104    | ECONNRESET         | Connection reset by peer         | L’altro lato ha chiuso in modo “brusco” la connessione.                |
| 105    | ENOBUFS            | No buffer space available        | Buffer di rete saturi, sistema sotto pressione.                        |
| 115    | EINPROGRESS        | Operation now in progress        | In socket non-bloccanti durante `connect()`.                           |
| 11     | EAGAIN/EWOULDBLOCK | Resource temporarily unavailable | Nessun dato pronto in socket non-bloccante (o coda piena in `send()`). |
| 4      | EINTR              | Interrupted system call          | La `recv()`/`send()` è stata interrotta da un segnale.                 |
# esempio 1
```c
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
int main () {
	/* Creazione socket */
	int sd = socket(AF_INET, SOCK_STREAM, 0);
	/* Creazione indirizzo */
	struct sockaddr_in my_addr;
	memset(&my_addr, 0, sizeof(my_addr); // Pulizia
	my_addr.sin_family = AF_INET ;
	my_addr.sin_port = htons(4242);
	inet_pton(AF_INET, "192.168.4.5", &my_addr.sin_addr);
}
```
# Esempio di connessione
![[Pasted image 20251103160053.png]]
# Server Concorrente
## Creazione di Processo Figlio
Come li distinguo? 
```c
p = fork()
```
- il padre vede `p = PID figlio`
- il figlio vede `p = 0` 
## Uso di Fork
Il figlio deve cancellare il socket in ascolto.
Il padre deve cancellare il socket per la comunicazione.
## Server multi-processo
```c
#include …
int main () {
int ret, sd, new_sd, len;
struct sockaddr_in my_addr, cl_addr;
//...
pid_t pid;
sd = socket(AF_INET, SOCK_STREAM, 0);
ret = bind(sd, (struct sockaddr*)&my_addr, sizeof(my_addr));
ret = listen(sd, 10);
len = sizeof(cl_addr);
//…
while (1) {
	new_sd = accept(sd, (struct sockaddr*)&cl_addr,&len);
	pid = fork();
	if (pid == -1){/*Gestione errore*/}
	if (pid == 0) {
	// Sono nel processo figlio. Chiudo socket in ascolto
		close(sd);
		// Servo la richiesta con new_sd
		//…
		close(new_sd);
		exit(0); // Il figlio termina
	}
	// Sono nel processo padre. Chiudo socket connesso al client
	close(new_sd);
}
```
## Uso dei thread
```c
#include <pthread.h>
```
e per compilare
```shell
gcc <opzioni> file.c -pthread
```
Tipi e funzioni da usare:
```c
pthread_t

pthread_mutex_t // semaforo per proteggere le risorse condivise

int pthread_create(pthread_t*thread, const pthread_attr_t*attr, void*(*start_routine)(void*),void*arg) // crea un nuovo thread che parte eseguendo start_routine(void*)

int pthread_join(pthread_t thread, void** retval) // un thread può bloccarsi in attesa della terminazione di un altro thread

void pthread_exit(void* retval) //per terminare un thread e mettere a disposizione un suo risultato (retval)
```
# Socket Bloccanti e non bloccanti
## Socket bloccante
Di default un socket è bloccante, tutte le operazioni su di esso fermano l'esecuszione del processo in attesa del risultato:
- connect
- accept
- send
- recv: si blocca finché non c'è un qualche dato disponibile
	- ritorna 0 finchè tutto il messaggio richiesto non è disponibile, con il flag MSG_WAITALL
## Socket non bloccante
Un socket può essere settato come non bloccante:
```c
socket(AF_INET,SOCK_STREAM|SOCK_NONBLOCK,0);
```
le operazioni non attendono risultati:
- connect: se non può connettersi subito restituisce -1 e setta **errno** a EINPROGRESS
- accept: se non si sono richieste restituisce -1 e setta errno a EWOULDBLOCK
- send: se non riesce ad inviare tutto il messaggio subito (il buffer è pieno), restituisce -1 e setta errno a EWOULDBLOCK
- recv: se non ci sono messaggi restituisce -1 e setta errno a EWOULDBLOCK
# I/O multiplexing
Esistono multiplexing sincrono e asincrono
## Multiplexing sincrono
Problema:
- voglio controllare più socket contemporaneamente
- se faccio operazioni su un socket bloccante, non posso controllarne altri
Soluzioni:
- multiplexing con la primitiva select()
- esamina più socket contemporaneamente, il primo pronto viene usato
	- **sincrono**: il programma rimane comunque in attesa che qualche descrittore sia pronto
## select()
Controlla più socket contemporaneamente
```c
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
int select(int nfds, fd_set *readfds, fd_set *writefds,fd_set *exceptfds, struct timeval *timeout);
```
- nfds: numero del descrittore più alto tra quelli da controllare, +1 (the descriptors from $0$ through $nfds-1$ in the descriptor sets are examined))
- readfds: lista di descrittori da controllare per la lettura
- writefds: lista di descrittori da controllare per la scrittura
- exceptfds: lista di descrittori da controllare per le eccezioni (non ci interessa)
- timeout: intervallo di timeout
- la funzione *restituisce il numero di descrittori pronti*, `-1` su errore
- la funzione è **bloccante**, si blocca finché un descrittore tra quelli controllati diventa pronto, oppure finché il timeout non scade. *Se scade il timeout ritorna 0 e non errore!* 
## Descrittori pronti
select() rileva i socket pronti.
Un *socket è pronto in lettura* se:
- C'è almeno un byte da leggere
- Il socket è stato chiuso (read() restituirà 0)
- È un socket in ascolto e ci sono connessioni effettuate
- C'è un errore (read() restituirà -1)

Un *socket è pronto in scrittura* se:
- C'è spazio nel buffer per scrivere
- C'è un errore (write() restituirà -1)
	- Se il socket è chiuso, errno = EPIPE
## Struttura del timeout
```c
#include <sys/socket.h>
#include <netinet/in.h>
struct timeval {
	long tv_sec; /* seconds */
	long tv_usec; /* microseconds */
};
```
- timeout = NULLL attesa indefinita, fino a quando il processore è pronto
- timeout = { 10 ; 5 }: attesa massima di 10secondi e 5 microsecondi
- timeout = { 0 ; 0 }: attesa nulla, controlla i descrittori ed esce immediatamente (**polling**)
## Insieme di descrittori
Un descrittore è un `int` che va da `0` a `FD_SETSIZE` (di solito 1024) (==questo è il numero massimo di socket che posso gestire con l'IO multiplexing, e dipende dal sistema==).
Un insieme di descrittori si rappresenta con una variabile di tipo `fd_set`:
- si manipola con macro simili a funzioni
```c
/* Rimuovere un descrittore dal set */
void FD_CLR(int fd, fd_set *set);
/* Controllare se un descrittore è nel set */
int FD_ISSET(int fd, fd_set *set);
/* Aggiungere un descrittore al set */
void FD_SET(int fd, fd_set *set);
/* Svuotare il set */
void FD_ZERO(fd_set *set);
```
## Utilizzo di select()
select() modifica i set di descrittori:
- prima di chiamare select() inserisco nei seti di lettura e di scrittura i descrittori che voglio monitorare
- dopo select() trovo nei set di lettura e scrittura i descrittori pronti
```c
int main(int argc, char *argv[]){
	fd_set master; // Set principale
	fd_set read_fds; // Set di lettura
	int fdmax; // Numero max di descrittori
	
	struct sockaddr_in sv_addr; // indirizzo server
	struct sockaddr_in cl_addr; // indirizzo client
	int listener; // socket per ascolto
	int newfd;
	char buf[1024]; // Buffer
	int nbytes;
	int addrlen;
	int i;
	/* Azzero i set */
	FD_ZERO(&master);
	FD_ZERO(&read_fds);
	
	listener = socket(AF_INET, SOCK_STREAM, 0);
	sv_addr.sin_family = AF_INET;
	// Mi metto in ascolto su tutte le interfacce (indirizzi IP)
	sv_addr.sin_addr.s_addr = INADDR_ANY;
	sv_addr.sin_port = htons(20000);
	bind(listener, (struct sockaddr*)& sv_addr, sizeof(sv_addr));
	listen(sd, 10);
	FD_SET(listener, &master); // Aggiungo il listener al set
	fdmax = listener; // Tengo traccia del maggiore
		for(;;) {
			read_fds = master; // Copia
			select(fdmax + 1, &read_fds, NULL, NULL, NULL);
			for(i = 0; i <= fdmax; i++) { // Scorro tutto il set
				if(FD_ISSET(i, &read_fds)) { // Trovato un desc. pronto
					if(i == listener) { // È il listener
						addrlen = sizeof(cl_addr);
						newfd = accept(listener,(struct sockaddr *)&cl_addr, &addrlen)
						FD_SET(newfd, &master); // Aggiungo il nuovo socket
						if(newfd > fdmax){ fdmax = newfd; } // Aggiorno max
						} else { // È un altro socket
						nbytes = recv(i, buf, sizeof(buf));
						//… Uso dati
						close(i); // Chiudo socket
						FD_CLR(i, &master); // Rimuovo il socket dal set
					}
				}
			}
		}
	return 0;
}
```
# Socket UDP
Vedi [[Transport Layer]].
La bind() per il client può anche essere opzionale. È necessaria se si vuole usare una porta specifica (per superare restrizioni di firewall o NAT)
Non c'è connessione, quindi non si fanno listen(), connect() o accept(), si utilizza solo socket(), bind() e sendto() e recvfrom().
## sendto()
Invia un messaggio attraverso un socket all'indirizzo specificato.
```c
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,const struct sockaddr *dest_addr, socklen_t addrlen);
```
- sockfd: descrittore socket da usare per invio
- len: dimensione in byte del messaggio
- flags: per settare opzioni, lo lasciamo a zero
- dest_addr: puntatore alla struttura in cui è salvato l'indirizzo del destinatario
- addrlen: lunghezza di dest_addr
- la funzione *restituisce il numero di byte inviati*, -1 su errore
- la funzione è **bloccante**, si ferma finché non ha scritto tutto il messaggio
## recvfrom()
Riceve un messaggio attraverso un socket.
```c
ssize_t recvfrom(int sockfd, const void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t addrlen);
```
- sockfd: descrittore del socket
- buf: puntatore al buffer per ricevere il messaggio
- len: dimensione in byte del messaggio
- src_addr: puntatore dove salvare l'indirizzo del mittente
- addrlen: lunghezza di src_addr
- la funzione restituisce il numero di byte ricevuti, -1 su errore, 0 se il socket remoto si è chiuso
- funzione **bloccante** 