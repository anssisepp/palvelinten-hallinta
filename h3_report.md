Tämä dokumentti on kirjoitettu vastauksena [Tero Karvisen Palvelinten hallinta -kurssin](https://terokarvinen.com/palvelinten-hallinta/) kotitehtävään _h3 Demoni_.

#### x) [Karvinen 2026: Apache installed with Ansible - quick notes](https://terokarvinen.com/apache-ansible/)

- Apache2:n asennus Ansiblella ja sivujen muokkaus ilman root-oikeuksia
- `package-file-service` roolien rakenteen perustana (asenna, muokkaa, uudelleenkäynnistä) 
- asetusmuutosten tulee aktivoida uudelleenkäynnistys automaattisesti nk. _handlerien_ avulla

#### [Ansible Community Documentation: Handlers: running operations on change](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_handlers.html)
- handlerit suoritetaan kun tehtävä ilmoittaa muutoksesta `notify`-rivin myötä (esim. `notify: restart apache2`)
- handleria kutsutaan omalla nimellä (esim. `name: restart apache2`)
- suoritetaan vasta tehtävien jälkeen
- sama handler suoritetaan vain kerran vaikka moni tehtävä sitä kutsuisi
- useamman handlerin voi yhdistää `listen`-avainsanalla

#### 'ansible-doc service'
- ohjaa ja valvoo etäkoneiden daemoneja riippumatta mitä järjestelmää kohdekone käyttää
- `name` hallittavan palvelun nimi 
- `enabled` määrittää käynnistyykö palvelu käynnistyksen yhteydessä vai ei
- `state` palvelun haluttu tila (`started`, `stopped`, `restarted`, `reloaded`) 
- `enabled` tai `state` vähintään yksi on määritettävä 
  
### a) Apache2-verkkopalvelimen asennus

Ensimmäiseksi tarkistettiin palvelimelle mahdollisesti jo aiemmin asennetut web-palvelimet komennolla `dpkg -l | grep -E "apache2|nginx|caddy"`. Tässä `dpkg -l` listasi kaikki asennetut paketit ja `grep -E` suodatti tulosteesta esiin vain halutut palvelinohjelmistot. Komentorivillä oli nähtävillä, että Apache2 (tai ainakin sen moduuleita) sekä Caddy olivat koneella asennettuina. Komennolla `sudo ss -tulpn | grep 80` nähtiin lisäksi, että portissa 80 liikennettä kuunteli Caddy. `ss -tulpn` listaa kaikki järjestelmän kuuntelevat portit sekä niitä käyttävät prosessit. 

<img width="1776" height="228" alt="Screenshot 2026-11-11 151136" src="https://github.com/user-attachments/assets/d71aa527-db25-4a78-827c-69c58be33a74" />

Puhtaan lähtötilanteen luomiseksi koneelta puhdistettiin vanhat palvelimet ja niihin liittyvät riippuvuudet komennoilla `sudo apt purge apache2 caddy` ja `sudo apt autoremove`. Tämän jälkeen aiemmat komennot uudelleen suorittamalla voitiin todeta, että koneella ei ollut enää web-palvelimia asennettuina tai käynnissä. Tehtävänannossa määrätty Apache2:n asennus suoritettiin päivittämällä ensin pakettilista ja asentamalla ohjelma komennolla `sudo apt install apache2`.

Käyttäjän kotihakemistoon luotiin oma kansio tulevia web-sivuja varten, jonne sijoitettiin testejä varten alustava `index.html`-tiedosto: 

```
mkdir -p ~/publicsite
echo "This is a site for testing purposes." > ~/publicsite/index.html
```
Tämän jälkeen Apache-daemonille tuli antaa suoritusoikeus käyttäjän kotihakemistoon, jotta palvelin voisi lukea siellä sijaitsevia tiedostoja. Muille käyttäjille eli others-ryhmälle annettiin suoritusoikeus komennolla `chmod o+x ~`. 

<img width="572" height="134" alt="Screenshot 2026-11-11 173244" src="https://github.com/user-attachments/assets/95c35667-14a9-4534-99a5-f69ea40cbd9a" />

Hakemistopolussa `/etc/apache2/sites-available/000-default.conf` sijainneen Apache2:n asetustiedoston oletusarvoinen `DocumentRoot /var/www/html`  muutettiin osoittamaan kotihakemistossa sijaitsevaan polkuun `/home/testuser/publicsite`.

Tässä vaiheessa sivustolle pääsyä yrittäessä kohdattiin `403 Forbidden`-vikailmoitus. Syynä oli se, että Apachella ei ollut oletuksena lupaa lukea tiedostoja käyttäjän kotihakemistosta, joten asetustiedostoon piti lisätä erillinen salliva lohko:

```
<Directory /home/testuser/publicsite>
    Require all granted
</Directory>
```

Tämän jälkeen web-palvelimen toimivuus tarkastettiin onnistuneesti `curl`-työkalulla sekä selaimen kautta.

<img width="928" height="440" alt="Screenshot 2026-11-11 160438" src="https://github.com/user-attachments/assets/e6984d58-71b1-4852-8f4c-799abb32c0e9" />

### b) Nginx asennus

Ennen siirtymistä Nginxin asentamiseen apache2-daemon pysäytettiin `sudo systemctl stop apache2`-komennolla ja sen käynnistyminen koneen uudelleenkäynnistyksen yhteydessä estettiin komennolla `sudo systemctl disable apache2`. Tila varmistettiin vielä `sudo systemctl status apache2` -komennolla. 

Nginxin asennus suoritettiin suoraviivaisesti komennolla `sudo apt install nginx`.

Tämän jälkeen muokattiin polussa `/etc/nginx/sites-available/default` sijainnutta Nginxin asetustiedostoa ja vaihdettiin tiedostosta löytynyt rivi `root /var/www/html` osoittamaan omassa kotihakemistossa sijainneeseen polkuun `/home/testuser/publicsite`.

Ohjelma käynnistettiin uudelleen ja varmistuttiin `curl`-työkalun avulla, että tulosteessa näkyi a-kohdassa luodun `index.html`-tiedoston sisältö.

<img width="1016" height="118" alt="Screenshot 2026-11-11 162134" src="https://github.com/user-attachments/assets/e4f0f688-f94e-48df-899f-d049a4c57fea" />

### c) Nginx asennus Ansiblella

##### Kohdan vastausta päivitetty 12.11.2026 klo 17:38. Alkuperäinen suppeampi vastaus tämän alla.

Seuraavaksi automatisoitiin Nginxin asennus suorittamalla Ansiblen avulla sekä ohjelman asennus, että yksinkertaisen konfiguraatiotiedoston siirto kohdepalvelimelle. Aivan aluksi palvelimelta poistettiin aiempi `nginx`-paketin asennus ajamalla paikallisesti komento `sudo apt purge nginx` ja `curl`-työkalulla varmistuttiin, että sivu ei ollut enää saatavilla.

Ansibleen luotiin uusi rooli `nginx`, minkä alle sijoittuivat kansiot `tasks`, `files` ja `handlers`. Tässä käytetiin koodien pohjana [Teron Apachen asennus Ansiblella](https://terokarvinen.com/apache-ansible/) -esimerkkiä.

Tasks-kansion alle `main.yml` tiedostoon sijoitettiin `apt`, `copy` ja `file` -moduulit sisältävä koodi, missä määriteltiin `nginx`-paketin asennus ja tiedoston siirto kohdepalvelimelle:

```
- apt:
    name: nginx
    state: present

- copy:
    dest: "/etc/nginx/sites-available/testisivu.fi.conf"
    src: "testisivu.fi.conf"
    owner: "root"
    group: "root"
    mode: "0644"
  notify: restart nginx

- file:
    src: /etc/nginx/sites-available/testisivu.fi.conf
    dest: /etc/nginx/sites-enabled/testisivu.fi.conf
    owner: root
    group: root
    state: link
  notify: restart nginx

```

Files-kansioon sijoitettiin Nginxin yksinkertainen määrittelytiedosto, jonka Ansible kopioi kohdepalvelimelle. Tiedosto `index.html` oli tässä tapauksessa kohdepalvelimella jo valmiina.

```
server {
    listen 80;
    server_name example.com www.example.com;

    root /home/testuser/publicsite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Handlers-kansioon luotiin `main.yml`-tiedosto, missä määriteltiin Nginxin uudelleenkäynnistys aina kun sitä kutsutaan `copy` tai `file` -moduulien `notify` riveiltä.

```                                           
- name: restart nginx
  systemd:
    name: nginx
    state: restarted
```

Lopuksi ajettiin `ansible-playbook site.yml` ja varmistuttiin tulosteen sekä `curl` avulla, että määritykset onnistuivat ja web-palvelin oli noussut käyntiin.

<img width="798" height="580" alt="Screenshot 2026-11-12 171452" src="https://github.com/user-attachments/assets/1dbb219c-9287-4ea7-9607-e8222148f48a" />


##### Alkuperäinen suppea vastaus:

Seuraavaksi lisättiin virtuaalikoneella Ansiblelle uusi rooli, joka suorittaa kohdepalvelimelle Nginx-webpalvelinsovelluksen asentamisen. Polussa `/ansible/roles/nginx/tasks/main.yml` roolin määrittelytiedostoon lisättiin seuraava `apt`-moodulin lohko:

```
- name:
  apt:
    name: nginx
    state: present
    update_cache: yes
  become: yes
```
Ansiblen pelikirjaan eli `site.yml` tiedostoon puolestaan lisättiin roolien alle kyseinen uusi `nginx`-rooli, jotta se sisältyisi osaksi pelikirjan suoritusta. Kun ajettiin komento `ansible-playbook site.yml`, tulosteen yhteenvedosta nähtiin kohdasta `changed=0`, että uusia muutoksia kohdepalvelimelle ei odotetusti tehty, sillä Nginx oli jo palvelimelle asennettuna. 

<img width="610" height="186" alt="Screenshot 2026-11-11 165003" src="https://github.com/user-attachments/assets/d5ddd740-865f-4032-8994-f00d177af4be" />


### Lähteet

Gite, Vivek 2025. Apache 403 Forbidden Error and Solution. Luettavissa: https://www.cyberciti.biz/faq/apache-403-forbidden-error-and-solution/. Luettu: 11.11.2026.

Karvinen, Tero 2026. Hello Ansible. Luettavissa: https://terokarvinen.com/hello-ansible/. Luettu: 8.11.2026. 

Karvinen, Tero 2016. New Default Website with Apache2 – Show your homepage at top of example.com, no tilde. Luettavissa: https://terokarvinen.com/2016/new-default-website-with-apache2-show-your-homepage-at-top-of-example-com-no-tilde/. Luettu: 11.11.2026.

Karvinen, Tero 2026. Palvelinten hallinta -kurssi. Luettavissa: https://terokarvinen.com/palvelinten-hallinta/. Luettu: 8.11.2026. 

Red Hat 2023. Linux file permissions explained. Luettavissa: https://www.redhat.com/en/blog/linux-file-permissions-explained. Luettu: 7.11.2026.

