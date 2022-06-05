# University Exercises
Repository for the second year's exercises of CS course at the university of Bologna.

## SO

### Argomenti trattati negli esercizi


### Headers
```C
#include <stdlib.h>
#include <stdio.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <limits.h>
#include <sys/stat.h>
#include <unistd.h>
#include <string.h>
#include <utime.h>
#include <dirent.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <time.h>

```
### Syscall 

* int eventfd(unsigned int initval, int flags): crea un eventfd object che può essere utilizzato per aspettare/notificare eventi. L'eventfd object contiene un unsigned int a 64 bit utilizzato come counter. Il counter viene inizializzato con il valore nel primo argomento. Come valore di ritorno, viene ritornato un file descriptor che può essere utilizzato per accedere all'eventfd object. Con la flag EFD_SEMAPHORE eventfd assume la semantica di un semaforo. Infatti se il suo counter ha un valore maggiore di 0, allora la funzione read ritorna il valore 1 e il counter viene decrementato di 1; se invece il counter è zero la chiamata di read provoca il blocco del processo finchè il counter non aumenta. La chiamata alla funzione write aggiunge l'intero da 8 byte(il buffer) al counter.

* int pthread_create(pthread_t *restrict thread,
                          const pthread_attr_t *restrict attr,
                          void *(*start_routine)(void *),
                          void *restrict arg) : questa funzione crea un nuovo thread nel processo chiamante. Il nuovo thread iniza l'esecuzione con la l'esecuzione della funzione start_routine(arg). Il thread creato termina se chiama pthread_exit() che specifica l'exit status, il quale è disponibile in un altro thread dello stesso processo che chiama pthread_join(); oppure termina quando ritorna da start_routine() o se viene cancellato con pthread_cancel().

* signalfd(int fd, const sigset_t *mask, int flags): crea un file descripotr che può essere usato per accettare segnali che riguardano il chiamante. E' un'alternativa a sigwaitinfo e ha il vantaggio che il file descripotr può essere monitorato utilizzando poll(), select(), e epoll(). L'argomento mask indica i tipi di segnali che devono essere considerati. Se fd è -1 allora viene creato un nuovo file descriptor , se non è -1 deve indicare un file descriptor. Di seguito la struttura standard per utilizzare signalfd
```C
#include <limits.h>
#include <sys/signalfd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <time.h>
#include <sys/wait.h>
#include <sys/types.h>

#define handle_error(msg) \
           do { perror(msg); exit(EXIT_FAILURE); } while (0)

#define _OPEN_SYS_ITOA_EXT

int main(int argc, char*argv[]) {
sigset_t mask;
int sfd;
struct signalfd_siginfo fdsi;
ssize_t s;
pid_t mainProc = getpid();
// Settiamo la mask su i segnali da ricevere
   sigemptyset(&mask);
   sigaddset(&mask, SIGUSR1);
   sigaddset(&mask, SIGUSR2);

// Tramite sigprocmask cambiamo la signal mask
// standard con quella appena creata
if (sigprocmask(SIG_BLOCK, &mask, NULL) == -1)
	handle_error("sigprocmask");
// Creiamo il file descriptor per accetare i segnali
// indicati dalla mask
sfd = signalfd(-1, &mask, 0);
if (sfd == -1)
	handle_error("signalfd");
// Creiamo il ciclo per ricevere i segnali tramite
// read()
for (;;) {
	s = read(sfd, &fdsi, sizeof(fdsi));
	if (s != sizeof(fdsi))
		handle_error("read");
	if (fdsi.ssi_signo == SIGUSR1) {
		char fileName[10];
		time_t currentTime = time(NULL);
		printf("Got SIGUSR1\n");
		// Creiamo il nome del file
		sprintf(fileName, "%d", fdsi.ssi_pid);
		FILE*file = fopen(fileName,"a");
		char* time_str=ctime(&currentTime);
		time_str[strlen(time_str)-1] = '\0';
		fprintf(file, "USR1 %s\n", time_str);
		fclose(file);
	} else if (fdsi.ssi_signo == SIGUSR2) {
		printf("Got SIGUSR2\n");
		char fileName[10];
		time_t currenTime = time(NULL);
		printf("Got SIGUSR2\n");
		// Creiamo il nome del file
		sprintf(fileName, "%d", fdsi.ssi_pid);
		FILE*file = fopen(fileName,"a");
		char* time_str=ctime(&currenTime);
		time_str[strlen(time_str)-1] = '\0';
		fprintf(file, "USR2 %s\n", time_str);
		fclose(file);
	 } else {
	    printf("Read unexpected signal\n");
    }
}
```


* kill(int pid, signal): di solito viene utilizzata per terminare processi, ma se si specifica il signal essa permette di mandare al processo identificato dal pid il signal in input. (Utile per testare programmi che ricevono segnali)

* wait(), waitpid().. permettono di bloccare il processo chiamante finchè non ricevono un segnale da un porcesso figlio. Se un processo figlio termina con exit(stat) waitpid prenderà il valore in exit e lo salverà in status(waitpid(pid, &status, 0)) ed è possibile leggere questo valore grazie alla macro WEXITSTATUS.
```C
for (int i=0; i < numberOfProcess; i++) {
		int status;
		if (processes[i]> 0) {
			waitpid(processes[i], &status, 0);
			if (WIFEXITED(status)) {
				if (WEXITSTATUS(status) == 0) {
					printf("%s %s differ\n", file1Path, file2Path);
					for (int k=i; k < numberOfProcess; k++){
						kill(processes[k], SIGTERM);
					}
					fclose(f1);
					fclose(f2);
					return 0;
				}
			}
		}
	}
```
* getopt(argc, argv, "options(se un'opzione prende un valore aggiungere:)"): prende gli argomenti che iniziano con - e il possibile valore successivo viene salvato in optarg. Alla fine l'argomento successivo all'utlima opzione ha indice optind.
```C
int option;
while ((option= getopt(argc, argv, "j:"))!= -1) { 
		switch(option) {
			case 'j':
				numberOfProcess=atoi(optarg);
				break;
			case ':':
				printf("Option require a value\n");
				break;
			default:
				printf("Invalid option\n");
				exit(EXIT_FAILURE);
		}
	}
```

* utime(filePath, struct utimebuf): permette di cambiare il tempo di ultimo accesso e di ultima modifica di un file. Per accedere ai tempi correnti del filre si può utilizzare stat e prendere il campo st_mtime e st_atime. Es:
```C
while ((entry=readdir(dir)) != NULL) {
			if (entry->d_type & DT_REG) {
				char filePath[PATH_MAX];
				struct utimbuf newTime;
				strcpy(filePath, currentDir);
				strcat(filePath, "/");
				strcat(filePath, entry->d_name);
				// Prendiamo il tempo di modifica corrente del file
				struct stat fileStat;
				stat(filePath, &fileStat);
				// 10 giorni in secondi = 864000
        fileStat.st_atime= fileStat.st_atime - 864000;
        fileStat.st_mtime= fileStat.st_mtime - 864000;
				newTime.actime = fileStat.st_atime;
				newTime.modtime = fileStat.st_mtime;
				utime(filePath, &newTime);
			}
		}
```
### Libraries and API

* inotify: fornisce metodi per monitorare directory e eventi nel filesystem in generale. Quando una directory viene monitorata inotify ritornerà gli eventi riguardanti quella directory.
Di seguito una procedura standard per l'inizializzazione di inotify su una directory(se si vogliono leggere più eventi inserire il while in un altro while (numberOfEvents = read(fd, buffer, EVENT_BUF_LEN)):
```C
#include <sys/inotify.h>
#include <sys/types.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <limits.h>
#include <unistd.h>

#define EVENT_SIZE (sizeof (struct inotify_event))
#define EVENT_BUF_LEN (1024 * ( EVENT_SIZE + 16))

int main(int argc, char*argv[]) {
  ...
  ...
  ...

  /*Viene utilizzato per salvare l'output di read*/
  int numberOfEvents = 0;
  
  /*Variabile per salvare il file descriptor che fa riferimento all'istanza di inotify*/
  int fd;
  
  /*Variabile per salvare il watch descriptor*/
  int wd;

  /*Buffer per leggere gli eventi  che si verificano*/
  char buffer[EVENT_BUF_LEN];

  // Inizializziamo l'istanza di inotify
  fd = inotify_init();
  
  // Controlliamo se l'istanza è stata creata correttamente
  if (fd < 0) perror("inotify_init");

  // Aggiungiamo alla watch list la directory exec da controllare
  wd = inotify_add_watch(fd, dir, IN_CREATE);

  // Chiamiamo read che rimane bloccato finchè non si verificano degli eventi 
  numberOfEvents = read(fd, buffer, EVENT_BUF_LEN);

  // Controlliamo se si sono verificati errori
  if (numberOfEvents < 0) perror("read");

  // A questo punto se non ci sono stati errori numberOfEvents 
  // tiene il conto degli eventi avvenuti
  // e il buffer contiene gli eventi 
  int i=0;
  while (i < numberOfEvents) {
    // Si può accedere ai campi dell'event per ricavare    informazioni utili
    struct inotify_event* event = (struct inotify_event*) &buffer[i];
    ...
    ...
    ...
    i += EVENT_SIZE + event->len;
  }
  inotify_rm_watch(fd, wd);
  close(fd);
  return 0;
}
```

* timerfd: permette di creare timer identificati da un file descriptor attraversp timerfd_create. Poi si utilizza timerfd_settime() per settare i valori del timer. In particolare timerfd_settime() prende in input 4 valori, il primo il file descriptor del timer, il secondo una flag(si può settare a 0), il terzo una struttura che setta i valori del timer(it_value.tv_sec è la parte intera del valore da aspettare in secondi, it_value.tv_nsec è la parte decimale in nanosecondi). Il seguente esempio mostra come settare un timer che prende in input valori in milli secondi e setta il timer per aspettare il tempo in input:
```C
// Creiamo il timer e salviamo il suo file descriptor
int timerFd=timerfd_create(CLOCK_REALTIME, 0);

if (timerFd > 0) {
	// Prendiamo il valore in input e lo convertiamo in intero
	int timeInMs= (int)atoi(argv[1]);
	float parteInteraSec= timeInMs/1000;
	float parteDecimaleSec=(timeInMs % 1000)/1000;
	float nanosec=parteDecimaleSec * 1000000000;
	printf("Tempo in sec: %f\n Tempo in nanosec: %f\n", parteInteraSec, nanosec);
	struct itimerspec spec;
	// spec.it_value specifica la scadenza iniziale del timer 
	// in secondi e in nanosecondi. Settare entrambi i valori con un valore 
	// != 0 fa partire il timer.
	spec.it_value.tv_sec = parteInteraSec;
	spec.it_value.tv_nsec = nanosec;
	//spec.it_interval se settato a 0, il timer espirerà solo
	// una volta, mentre seù settato a un valore != 0 specifica il periodo
	// per timer ripetuti.
	spec.it_interval.tv_sec = 0;
	spec.it_interval.tv_nsec = 0;
	timerfd_settime(timerFd, 0, &spec, NULL);
	printf("Timer started\n");
	uint64_t buf;
	ssize_t size;
	size= read(timerFd, &buf, sizeof(uint64_t));
		if (size != sizeof(uint64_t))
			printf("Error: read\n");
	
	
	printf("Timer expired after %d ms\n", timeInMs);
	
} 
return 0;
```

* poll(): poll permette di avere un insieme di file descriptor e di aspettare gli eventi per ognuno di essi. Può aspettare per un singolo evento oppure per più eventi. Ritorna il numero di eventi che si sono verificati. Di seguito un esempio di utilizzo con timerfd in cui si hanno più timer e si controlla con poll quando scadono.
```C
struct pollfd timerPollFd[argc - 1];
	struct itimerspec specs[argc - 1];
	int timesInMs[argc - 1];
	// Creiamo il timer e salviamo il suo file descriptor per
	// ogni valore in input
	for (int i=0; i < argc - 1; i++) {
	timerPollFd[i].fd=timerfd_create(CLOCK_REALTIME, 0);
	timerPollFd[i].events = POLLIN;
	// E' un campo di ritorno e diventa != 0
	// in base al numero di fd che hanno completato
	// il loro task.

	timerPollFd[i].revents = 0;
	timesInMs[i] =(int)atoi(argv[i+1]);
	float parteInteraSec= timesInMs[i]/1000;
	float parteDecimaleSec=(timesInMs[i] % 1000)/1000;
	float nanosec=parteDecimaleSec* 1000000000;
	specs[i].it_value.tv_sec= parteInteraSec;
	specs[i].it_value.tv_nsec = nanosec;

	//spec.it_interval se settato a 0, il timer espirerà solo
	// una volta, mentre se è settato a un valore != 0 specifica il periodo
	// per timer ripetuti.
	specs[i].it_interval.tv_sec = 0;
	specs[i].it_interval.tv_nsec = 0;
	
	timerfd_settime(timerPollFd[i].fd, 0, &specs[i], NULL);
	printf("Timer %d started\n", timesInMs[i]);

	}
	int i=0;
	int timers = argc -1;
	while (timers > 0) {
		int numOfEvents = poll(&timerPollFd[i],1, -1);
		if(numOfEvents > 0) {
			uint64_t buf;
			ssize_t size;
			size= read(timerPollFd[i].fd, &buf, sizeof(uint64_t));
			if (size != sizeof(uint64_t))
				printf("Error: read\n");
			printf("Timer expired after %d ms\n", timesInMs[i]);
			i++;
			timers--;
		}
	}
	for(int i=0; i<argc; i++) {
		close(timerPollFd[i].fd);
	}
```
### Snippets:
* Snippet di codice per estrarre/spezzare una stringa in base a un carattere inserito attraverso strtok:
```C
token = strtok(lineToRead, " ");
while (token != NULL && i <= 2) {
		token = strtok(NULL, " ");
}
```

* Snippet per aprire un file attraverso fopen:
```C
char filePath[PATH_MAX];
	strcpy(filePath, argv[1]);
	FILE* file= fopen(filePath, "r");

	if (file == NULL) {
		printf("File doesn't found.\n");
        exit(EXIT_FAILURE);
	}
```
* Struttura per testare i segnali utilizzando due processi in parallelo che li mandano:
```C
if(fork() != 0) {
  // codice del processo principale a cui arrivano i segnali
}else {
// Creiamo due processi per testare i signal
	pid_t firstProc;
	pid_t secondProc;
	firstProc = fork();
	if (firstProc == 0) {
		printf("Entered in first process with pid %d\n", getpid());
		kill(mainProc, SIGUSR1);
		sleep(3);
		kill(mainProc, SIGUSR1);
		sleep(4);
		kill(mainProc, SIGUSR2);
		exit(EXIT_SUCCESS);
	}
	secondProc = fork();
	if (secondProc == 0) {
		printf("Entered in second process with pid %d\n", getpid());
		kill(mainProc, SIGUSR2);
		sleep(3);
		kill(mainProc, SIGUSR2);
		sleep(4);
		kill(mainProc, SIGUSR1);
		exit(EXIT_SUCCESS);
	}
}
```
* Snippet per prendere comandi in input:
```C
// Estraiamo il comando in input
char*args[argc];
int i=1;
while (i < argc && argv[i]!= NULL) {
	args[i-1] = argv[i];
	i++;
}
args[i]=NULL;
```

* Funzione per comparare due file settando il punto di partenza e la grandezza del blocco da controllare:
```C
int compareFiles(FILE* file1, FILE* file2, size_t startPoint, size_t blockSize, int processNumber) {
	fseek(file1, startPoint, SEEK_SET);
	fseek(file2, startPoint, SEEK_SET);
	printf("Block size: %ld\n", blockSize);
	char c1=getc(file1);
	char c2=getc(file2);
	size_t i=0;
	while (c1 != EOF && c2 != EOF && i<=blockSize) {
		i++;
		printf("c1: %c id: %d \n", c1, processNumber);
		printf("c2: %c id: %d \n", c2, processNumber);
		if (c1 != c2) {
			return 0;
		}
		c1 = getc(file1);
		c2 = getc(file2);
	}
	return 1;
}

```
### Script

#### Python
* Per esplorare i file all'interno di una singola directory utilizzare os.listdir(dir)
```Python
entries = os.listdir(dir)
	for entry in entries:
		name, ext=os.path.splitext(entry)
		if ('.' in ext):
			filesWithSuffix.setdefault(ext, [])
			filesWithSuffix[ext].append(name+ext)
		else:
			filesWithoutSuffix.append(name)
```
* Per ottenere il path assoluto della directory corrente utilizzare os.getcwd(). 

* setdefault permette di creare dizionari con più elementi associati alla stessa chiave:
```Python
filesWithSuffix={}
filesWithSuffix.setdefault(ext, [])
filesWithSuffix[ext].append(name+ext)
```

* per ordinare un dizionario bisogna ordinare con sorted la lista delle chiavi o quella degli elementi:
Es:
```Python
def sortDictionary(dictionary):
    # Estraiamo e ordiniamo dal dizionario i file
		# ordiniamo in ordine alfabetico
    sorted_keys = sorted(dictionary.keys(), key=lambda x: x.lower())

    # Creiamo un dizionario ordinato
    sorted_dict = {}
    for key in sorted_keys:
        sorted_dict[key] = dictionary[key]

    return sorted_dict

```

* Per esplorare un sottoalbero di directory si utilizza os.walk. Esso per ogni direcotory restituisce i file all'interno, il path della directory e i nomi delle directory all'interno.
Es:
```Python
for dirPath, dirNames, files in os.walk(sys.argv[1]):
      # Inseriamo ogni file presente nella directory corrente nel dizionario
      # e associamo a ogni file la directory a cui appartiene
        for file in files:
            lastModTime = os.path.getmtime(os.path.join(dirPath, file))
            subTree.setdefault(lastModTime, [])
            subTree[lastModTime].append(file)
    sort = sorted(subTree.keys())
    print(sort)
    print(
        f'\nLast modified file: {subTree[sort[-1]]}, date of modification: {sort[-1]}')
    print(
        f'\nFile modified earlier: {subTree[sort[0]]}, date of modification: {sort[0]}')
```

* Per ottenere l'utlimo tempo di modifica di un file in python si utilzza os.path.getmtime().
Es:
```Python
lastModTime = os.path.getmtime(os.path.join(dirPath, file))
```

* Per ottenere l'ultimo tempo di accesso a un file o la data di creazione si utilizza os.stat().

* Per dividere il nome di un file in nome e estensione si utilizza os.splitext(nome file)
Es
```Python
fileName, fileExt = os.path.splitext(file)
```
* I for per andare da un valore all'altro utilizzano range(start, end, step)
```Python
for i in range(1, len(sys.argv), 1):
```
* os.scandir() restituisce un os.DirEntry che permette ad accedere a informazioni sui file all'interno di una directory. In particolare è possibile ottenere l'inode e altre informazioni.
Es
```Python
 dirEntries = os.scandir(dirPath)
    for entry in dirEntries:
      pathname.setdefault(entry.inode(), [])
      pathname[entry.inode()].append(os.path.join(dirPath, entry.name))
  
```
* Per controllare gli elementi all'interno di una directory os.path. offre alcuni metodi come os.path.isfile per capire se un path è un file o no.
Es
```Python
 if os.path.isfile(os.path.join(dir, file)):
```
* filecmp.cmp serve per confrontare due file, di default l'opzione shallow è True, quindi confronta la size, la data di modifica ecc ma non il contenuto effettivo. Per confrontare il contenuto bisogna mettere shallow = False.
Es:
```Python
if filecmp.cmp(f, os.path.join(dir, file), shallow=False) == True:
```
* Per eliminare un file si utilizza os.remove(path).

* Per creare un hard link si utilizza os.link(filePointedPath, nameOfLinkFile).

* Per creare un symbolic link si utilizza os.symlink(filePointedPath, nameOfSymlinkFile).

* Funzione per contare le linee di un file.
```Python
def countLines(fileName):
  file = open(fileName, "r")
  counter = 0
  text = file.read()
  textList = text.split('\n')

  for lines in textList:
    if lines:
      counter+=1
  return counter
```
* In python per convertire un tipo in un altro basta fare str(int) o int (str).

* Per leggere le linee di un file si utilizza file.readline(), dopo aver aperto un file tramite open(path, mod).
```Python
openedFile = open(os.path.join(dirPath, file), "r")
firstLine = openedFile.readline()
```

* Per leggere comandi scritti da linea di comando durante l'esecuzione si può utilizzare sys.stdin.readline()

* Per splittare una stringa in base al carattere basta avere una stringa e fare .split("carattere su cui splittare")
```Python
text.split("\n")
```
* Un file aperto può anche essere letto tramite file.read(byte da leggere) che restituisce come stringa i byte letti. Se non si mette nulla in input il valore è -1 ovvero legge l'intero file.

* Per eseguire comandi da terminale si può usare subprocess.run(command, shell=False), che restituisce un oggetto CompletedProcess al completamento. Su questo oggetto è possibile controllare il valore ritornato dal comando eseguito tramite returnValue.check_returncode().
```Python
returnValue = subprocess.run(commands, shell=False)
returnValue.check_returncode()
```

* Per controllare se si ha accesso a un file in scrittura, lettura ecc si utilizza os.access(path, mod)

* Per controllare se una stringa rappresenta un numero si può utilizzare stringa.isnumeric()

* Per scrivere stringhe con più linee la si racchiude tra ''' '''.
```Python
stringa= '''
  #include<stdlib.h>
  #include<stdio.h>

  int main() {
    char *syscall_name[446];
  '''
```
#### Bash


### NOTE:

* I file collegati da un hard link hanno lo stesso inode, compreso il file originale 

* La syscall fork crea un processo figlio del chiamante e dopo la creazione entrambi i processi continuano a eseguire il codice sotto la chiamata.
  Se il valore ritornato dalla fork è 0 allora il controllo è passato al processo creato, se è -1 c'è stato un errore e se è > 0 è il pid del processo crato(viene ritornato al chiamante).

* Utilizzare lstat o stat per ricavare informazioni sul tipo di file. Il campo st_mode tramite funzioni o con & con constanti può essere utilizzato per trovare il tipo di file. 
ES
```C
// Controlliamo se il file è un eseguibile
struct stat file;
lstat(filePath, &file);
if (S_IXUSR & file.st_mode) {}
```

* dup2 può essere utilizzato per stampare l'output di un comando su un file. Esso prende un file descriptor che può essere creato tramite la funzione open().
ES:
```C
argumentList[k] = NULL;
char* prova[] = {"ls", "-l", NULL};
pid_t child;
if ((child = fork()) == 0) {
  int fdOut = open(filePath, O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
  dup2(fdOut, 1);
  close(fdOut);
  int status= execvp(command, argumentList);
          
  if (status == -1) printf("errore");
}

wait(&child);
```
* Quando si elimina un file con rm se si ha un processo che lo mantiene aperto è possibile leggere e scrivere sul file, in quanto non viene liberato il file descriptor nel caso in cui un processo lo sta utilizzando.

* realpath() funziona se il nome del file da trovare è nella directory in cui è il processo che lo chiama, oppure bisogna mettergli il path per trovare il file.

* Per esplorare i file all'interno di una directory conviene usare readdir dopo aver eseguito opendir(DIR* dir). Poi con una struct dirent è possibile avere informazioni sui file all'interno della directory.
```C
#include <dirent.h>
#include <fcntl.h>


DIR* targetDir = opendir(dir);
struct dirent *entry;
file_t files[100];
if (targetDir == NULL){
	perror("Unable to read directory\n");
	exit(EXIT_FAILURE);
}
while ((entry=readdir(targetDir))) {
	if (entry->d_type & DT_REG) {
		char filePath[PATH_MAX];
		strcpy(filePath, dir);
		strcat(filePath, "/");
		strcat(filePath, entry->d_name);
		}
}

```
* Per compilare con eventuali librerie esterne come quella del prof conviene utilizzare:
```shell
gcc file.c -L/home/your_user/path_to_library/build -l:libexecs.a
```
* Linux non ha itoa quindi si deve usare sprintf:
```C
sprintf(string, "%d", number_to_convert);
```
* Per far partire la lettura di un file da un punto diverso da quello iniziale si può utilizzare fseek(file, offset in byte che viene aggiunto alla partenza, punto di partenza). Insieme ad fseek se si usa ftell si può calcolare la dimensione di un file in byte.
```C
fseek(f1, 0L, SEEK_END);
size_t sizeFile1= ftell(f1);

```
* La funzione basename(path) può essere utilizzata per estrarre il nome del file da un path.
## Web Tecnologies