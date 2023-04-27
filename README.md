# VPS deployment (NodeJS)

> *********************************************Ovo bi trebalo vecinski da funkcionise za bilo koji drugi VPS*********************************************
> 

# Pocetak

- Napravis nalog na linode
- Zakupis VPS masinu sa odredjenim specifikacijama
- Izaberes Ubuntu LTS kao operativni sistem
- Postavis sifru za VPS
- Generises ssh key ili samo svaki put kucas sifru
- Pristupis VPS-u iz temrinala preko ssh
- Treba uraditi update-ovati Linux sistem sa komandama:

```shell
apt-get update
apt-get upgrade
```

# Setup NodeJS i Git

- Instaliras npm i node
- Izlogujes se i ponovo ulogujes i proveris verziju nodea:

```shell
node -v
```

- Setupujes ssh kljuc za git da mozes da povuces repo
    
    ```bash
    ssh-keygen -t rsa
    ```
    
    - lupis sve entere
    - sacuvace se u **`root/.ssh`** folderu
    - private key je **`id_rsa`**
    - public key je **`id_rsa.pub`**
    - **id_rsa.pub** dodas kao trusted ssh key na repo settingsu na githubu
    - pullujes kod
- Uradis **`npm install`** da instaliras node modules
- Preko **SCP**-a kopiras **.env** sa svoje masine na server u folder projekta
- **`node app.js`** i program ce raditi na serveru
- Program u tom trenutku radi na portu 5000 / ili koji god da si uneo (treba kasnije port forwardovati sa reverse proxy-jem sa 80 ili 443 na 5000)

> *Ne bi bilo lose dodati “app.set('trust proxy', 1);” u app.js
i “proxy: true” kod kreiranja bilo kog cookie-a na backendu*
> 

# Process managing sa pm2 package-om

- Treba menage-ovati proces, ne mozes samo pokrenuti **node app.js** jer to ima gomilu mana i nesigurnosti (npr svaki put kad pukne program ili se resetuje masina servera onda moras da se logujes na ssh da ga ozivis rucno)
- Opis paketa sa npm sajta: *“PM2 is a production process manager for Node.js applications with a built-in load balancer. It allows you to keep applications alive forever, to reload them without downtime and to facilitate common system admin tasks.”*
- ChatGPT: ***“using a process manager like pm2 is highly recommended for running Node.js applications in production. Pm2 helps to manage your Node.js processes and ensure they run reliably, without being killed or crashing.”***
- Neke od osnovnih komandi:
    1. **`pm2 start app.js`**: This command starts your Node.js application. You can replace **`app.js`** with the name of your main application file.
    2. **`pm2 list`**: This command lists all the processes that are currently managed by pm2.
    3. **`pm2 stop app`**: This command stops a specific process named **`app`**. Replace **`app`** with the name of the process you want to stop.
    4. **`pm2 restart app`**: This command restarts a specific process named **`app`**. Replace **`app`** with the name of the process you want to restart.
    5. **`pm2 delete app`**: This command deletes a specific process named **`app`**. Replace **`app`** with the name of the process you want to delete.
    6. **`pm2 logs`**: This command displays the logs for all managed processes.
    7. **`pm2 monit`**: This command opens up a monitoring interface that displays detailed information about the managed processes, including CPU usage, memory usage, and more.
    8. **`pm2 save`**: This command saves the current state of all managed processes, so that they will be automatically restarted if the server is rebooted.
    9. **`pm2 startup`**: This command generates and configures a startup script for pm2, so that your managed processes will automatically start when the server is rebooted.
- Instaliraj paket globalno:

```shell
npm install -g pm2
```

- Kreiraj proces sa imenom ***“api”*** i pokreni ga:

```shell
pm2 start app.js -n api
```

- Server sada konstantno radi i ako nodejs program crashuje, pm2 ce probati da ga pokrene 15 puta u toku jedne sekunde dok ne odustane, ovo su generalno situacije koje svakako ne bi smele da se dogode
- Krerati startup skriptu koja ce svaki put da pokrene proces kada se server reboot-uje / crash-uje

```shell
pm2 startup ubuntu
```

# Reverse proxy sa Nginx

- Nginx je najpopularnija tehnologija na svetu za proxy i reverse-proxy
- Da bi reverse proxy radio nije potrebno imati domen, sve se moze setupovati sa ip adresom servera ali da bi kasnije mogli da setupujemo SSL (https) moramo imati domen za bekend
- Instaliraj Nginx:

```shell
apt-get install nginx
```

- Kada se instalira kreirace novi folder u koji se treba prebaciti

```shell
cd ~/etc/nginx
```

- U njemu se nalaze svi nginx fajlovi, nama je najbitniji folder **"`sites-available`"**

```shell
cd sites-available
```

- U tom folderu se nalazi fajl koji se zove **“`default`”**. Njega treba editovati (preko nano-a) da bi se konfigurisao reverse proxy
- U njemu postoji deo fajla:

```shell
server_name _;
```

- Njega treba konfigurisati sa domain name-om bekenda i onda se dobije:

```shell
server_name nekidomen.com www.nekidomen.com;
```

- Obavezno ukljuciti verziju sa i bez www
- Ispod toga u fajlu se nalazi **`location`** blok koji izgleda ovako:

```shell
location / {
	# Neki garbage
}
```

- Obrisati sve sto se nalazi u tom bloku i dodati sledece konfiguracije:

```shell
location / {
	proxy_pass http://localhost:5000;
	proxy_http_version 1.1;
	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection ‘upgrade’;
	proxy_set_header Host $host;
	proxy_cache_bypass $http_upgrade;
}
```

- Da bi rate limiter na node serveru radio moraju se na reverse proxy-ju setovati dodatni headeri kako bi node server znao da preko njih gleda ip adresu klijenta a ne da misli da ga reverse proxy bombarduje DDOS-om

```shell
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

- Na Node serveru treba dodati ovu liniju kako bi server znao da uzima ip adrese iz tih headera umesto da gleda odakle je stvarno zahtev dosao `app.set('trust proxy', true);`
- Sacuvati i izaci iz fajla
- Pokrenuti komandu da proverimo da li je sve kako treba:

```shell
nginx -t
```

- Restartovati Nginx proces da bi prihvatio novu konfiguraciju:

```shell
systemctl restart nginx
```

- To je bila sistemska komanda za pravi reset procesa koja se preporucuje, a postoji i Nginx komanda za reset master workera bez gasenja servera

```shell
nginx -s reload
```

- Sada bi trebalo da se bekendu moze pristupiti na “`domen/api`” umesto “`domen/api:5000`"
- Ako taj url vraca “Apache2 Default page” html stranicu to znaci da je upaljen apache server na VPS masini i treba ga ugasiti i zabraniti da se pali pri svakom narednom reboot-u servera
- Komande za stopiranje i onemogucavanje apache servera
```shell
sudo systemctl stop apache2
sudo systemctl disable apache2
```

# Firewall setup

- Postavljamo firewall da bi dozvolili pristup serveru na samo 3 porta:
    - `80 (HTTP)`
    - `443 (HTTPS)`
    - `22 (SSH)`
- Koristicemo Ubuntu-ov firewall: `ufw`

```shell
ufw enable
ufw allow ssh
ufw allow http
ufw allow https
```

- Sada vise ne mozemo pristupiti direktno sa porta 5000, samo sa porta 80 a onda ce reverse proxy to forwardovati na proces na portu 5000

# Setupovanje domain name-a

- U Linode odemo u Domains
- Create Domain
- Unesemo ime domena koji smo negde zakupili: [nekidomen.com](http://nekidomen.com) (bez www)
- Unesemo email adresu povezanu sa domainom
- U Insert Default Records izaberemo Insert default records from one of my Linodes
- Izaberemo nas Linode (VPS na koji deployujemo)
- Kliknemo Create Domain
- Sada cemo videti imena svih name servera:
    - ns1.linode.com
    - ns2.linode.com
    - ns3.linode.com
    - ns4.linode.com
    - ns5.linode.com
- Odemo na sajt gde smo zakupili domain (GoDaddy ili NameCheap)
- U polju za podesavanje nameservera izaberemo Custom DNS
- Dodamo svih 5 linode name servera
- Ovo podesavanje moze da traje i do 48 sati da se zapravo izvrsi
- Sada smo povezali nas Linode VPS na taj domain name

# Setupovanje SSL-a i HTTPS-a

- Trenutno kada pristupimo url-u pise **“Not secure”**
- Kada bi hteli da rucno ukucamo **“https://url”** server ne bi respondovao jer nema setupovan HTTPS
- Za ovaj korak je **NEOPHODAN** domain name
- Koristicemo letsencrypt i certbot da generisemo besplatne SSL sertifikate
- Instaliramo Certbot i Certbot plugin za Nginx:

```shell
sudo apt-get install certbot
sudo apt-get install python3-certbot-nginx
```

- Hocemo da pokrenemo sledecu komandu da bi certbot prosao kroz config fajlove od Nginx-a i setupovao SSL port

```shell
certbot --nginx
```

- Pitace nas da unesemo email adresu gde ce nam slati obavestenje svaki put kad bude automatski obnavljao SSL sertifikat
- Pitace da li smo procitali Terms of Service i tu napisemo **“`Yes`**”
- Pitace da li zelimo da im dajemo developer informacije, napisemo “`**No**`”
- Povuci ce domain imena koja smo stavili u config fajl od Nginx-a i pitace za koje od njih zelimo da aktiviramo HTTPS, samo unesemo prazno i izabrace sve
- Sada bi trebalo da u browseru mozemo da pristupimo preko https, odnosno da pise da je secured
- Poslednja komanda koja nam je potrebna, da bi testirali renewal SSL sertifikata sa certbotom:

```shell
certbot renew --dry-run
```

- Trebalo bi da svaki put 90 dana pred istek SSL sertifikata certbot kreira novi sertifikat, ova komanda malopre je bila samo da simulira tu obnovu da vidimo da li obnova radi kako treba
- Mozda resetovati Nginx proces opet za svaki slucaj

# Deployment proces zavrsen!!!
