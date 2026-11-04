#### Raportti on keskeneräinen ja valmistuu myöhässä lopulliseen muotoonsa viimeistään klo 20:15 mennessä. 

Tämä dokumentti on kirjoitettu vastauksena [Tero Karvisen Palvelinten hallinta -kurssin](https://terokarvinen.com/palvelinten-hallinta/) kotitehtävään _h2 Voileipä_.

Ympäristönä toimii Oraclen VirtualBoxiin asennettu Debian 13.7.0 -käyttöjärjestelmä. Virtuaalikoneen laitteistoresursseiksi on asetettu kaksi prosessoriydintä, 4 Gt keskusmuistia ja 40 Gt levytilaa. Harjoituksissa hyödynnetään lisäksi UpCloudin virtuaalipalvelinta.

#### x) [Karvinen 2026: Sudo without password](https://terokarvinen.com/passwordless-sudo/) / [Karvinen 2026: Passwordless Sudo with Ansible](https://terokarvinen.com/passwordless-sudo-with-ansible/)

- konfiguroitaessa hyvä olla samanaikainen root-yhteys kohteeseen vikatilanteita varten
- eduksi luoda asetustiedosto, joka sallii sudoless-ryhmän jäsenten käyttää `sudo`-komentoja ilman salasanakyselyä
- tiedosto `/etc/sudoers.d/sudoless` sisältää rivin `%sudoless ALL = (ALL) NOPASSWD: ALL`
- manuaalinen testaus ennen automaatiota
- oikeuksien määritys yhteensopivin ja turvallisin oktaalimuotoisena (esim. 0644)
  

### a) & b) Salasanaton tunnus ja etähallinta Ansiblella

Käytössä olleelle kohdepalvelimelle luotiin Ansible-hallintatyökalun avulla uusi käyttäjä `testuser`, jolle konfiguroitiin valmiiksi SSH-avaimella kirjautuminen sekä oikeus käyttää sudo-komentoja ilman salasanaa. 

Ensimmäisenä muokattiin Ansiblen inventaariotiedostoa `hosts.ini`, johon kohdekoneeksi määritettiin palvelimen IP-osoite ja yhteyskäyttäjäksi `ansible_user=root`. Tämä jälkimmäinen tuli lisätä siksi, että Ansible yrittää oletusarvoisesti kirjautua kohdepalvelimelle samalla paikallisella tunnuksella, jolla omalla koneella ollaan kirjautuneena. Kyseistä käyttäjätunnusta ei ole olemassa kohdepalvelimella, joten yhteys ei olisi muutoin muodostunut. 

Seuraavaksi luotiin uuden roolin määrittelytiedosto polkuun `roles/testuser/tasks/main.yml`. Tiedostossa määritettiin ehdot, joilla luotiin uusi `testuser`-niminen käyttäjä ja tarvittavat ryhmät, sekä liitettiin julkinen SSH-avain ja määritettiin salasanaton sudo-oikeus luomalla asetus tiedostoon `/etc/sudoers.d/sudoless`. Tiedoston syntaksin pohjana hyödynnettiin Teron _[Passwordless Sudo with Ansible](https://terokarvinen.com/apache-ansible/)_ ohjesivua.

Olemassa olleeseen playbook-tiedostoon (`site.yml`) lisättiin kyseinen `testuser`-rooli sekä `become: true`, jotta Ansible suorittaisi tehtävät pääkäyttäjän oikeuksilla. 

Automaatio ajettiin komennolla `ansible-playbook site.yml`.

Ansiblen yhteenveto näytti, että kaikki meni onnistuneesti läpi (ok=6, changed=4, failed=0).

<img width="1464" height="688" alt="Screenshot 2026-11-04 161558" src="https://github.com/user-attachments/assets/5dc3a602-1f21-4e1a-b1fb-56daf86e6bd5" />

Lopullinen todentaminen tapahtui kirjautumalla luodulla uudella käyttäjällä palvelimelle `ssh testuser@94.237.9.166` ja varmistamalla `whoami` ja `groups` komennoilla, että kyseessä oli varmasti oikea käyttäjä ja ryhmäoikeudet.

<img width="1010" height="398" alt="Screenshot 2026-11-04 162159" src="https://github.com/user-attachments/assets/edb8511b-e833-4eae-b10d-064173e65cea" />

### c) Pakettien asentaminen Ansiblella

Seuraavaksi asennettiin automaation avulla muutama paketti kohdepalvelimelle.

Tämä tapahtui muokkaamalla `testuser`-roolin tehtävätiedostoa `roles/testuser/tasks/main.yml`, minne määritettiin asennettaviksi paketeiksi `apache2`, `tree` ja `htop`. Näiden jälkeen lisättiin määre `state: present`.

<img width="270" height="192" alt="Screenshot 2026-11-04 172817" src="https://github.com/user-attachments/assets/59145c9c-494b-45a6-8cff-15f44216e734" />

Tämän jälkeen automaatio ajettiin jälleen komennolla `ansible-playbook site.yml`.

Palvelimelle kirjautuneena todennettiin asennettujen ohjelmien olemassaolo ja lyhyesti testattiin niiden toimivuutta. Tarkistettiin Apachen tila komennolla `sudo systemctl status apache2`, josta pystyi näkemään, että verkkopalvelin oli aktiivinen.

<img width="1120" height="230" alt="Screenshot 2026-11-04 165355" src="https://github.com/user-attachments/assets/3c96afee-287a-4a88-9e6e-e6b21630ea8e" />

### d) Monirivisen tekstitiedoston lisäys ja oikeudet

Kohdepalvelimelle luotiin useamman rivin mittainen tekstitiedosto lisäämällä `testuser`-roolin `main.yml`-tiedostoon uusi lohko. Tiedoston kohdepoluksi määritettiin käyttäjän kotihakemistossa sijaitseva `/home/testuser/testfile.txt` ja sen sisältö tuotettiin käyttämällä YAML-kielen tukemaa monirivistä putkimerkintää eli `|` (Yaml Validate 2026).

<img width="690" height="300" alt="Screenshot 2026-11-04 175309" src="https://github.com/user-attachments/assets/466a25d6-5c7a-4f38-bc7a-27cf2a8e1133" />

Samassa määritettiin tiedostolle omistajuudet asettamalla omistajaksi ja ryhmäksi `testuser` sekä käyttöoikeudet numeromuodossa `0644`. Lisäysten myötä tiedosto tallennettiin ja ajettiin jälleen `ansible-playbook site.yml`, minkä jälkeen luodun tiedoston olemassaolo, oikeudet ja sisältö todennettiin kohdepalvelimella ajamalla komennot `ls -l` sekä `cat testfile.txt`. 

<img width="776" height="282" alt="Screenshot 2026-11-04 175503" src="https://github.com/user-attachments/assets/e0872c82-cabd-4f6b-b0d0-77f819e349d1" />

Aiemmin mainittu `0644` vastaa käyttöliittymässä esitettynä muotoa `-rw-r--r--`. 

Kirjaimilla symboloidaan oikeuksia seuraavasti:

```
r (read)    = luku
w (write)   = kirjoitus
x (execute) = suoritus
-           = ei oikeuksia
```

Tiedostojen oikeudet esitetään kolmessa lohkossa, jotka määrittelevät järjestyksessä omistajan, omistavan ryhmän sekä muiden käyttäjien oikeudet. Tämän tiedoston kohdalla omistajalla on luku- ja kirjoitusoikeus, kun taas omistavalla ryhmällä ja muilla järjestelmän käyttäjillä on pelkät lukuoikeudet. Tässä yhteydessä tiedostolle ei ole määritetty suoritusoikeutta (`x`), joka tulisi tarpeen vasta mikäli tiedostoa olisi tarkoitus ajaa suoritettavana skriptinä tai ohjelmana. Käyttöoikeuksilla on omat numeroarvonsa, missä `r=4`, `w=2` ja `x=1`, mitkä yhteen laskemalla selittyy konfiguroimisessa käytetty oktaalinumero `0644`. Ensimmäinen arvo, tässä tapauksessa `0` viittaa erikoisoikeuksiin, joita ei yleensä tavanomaisissa tiedostoissa käytetä. (Red Hat 2023.)

### e) Palvelimella pyörivän daemonin tilasta varmistuminen  

Aiemmassa vaiheessa palvelimelle asennetun Apache-verkkopalvelimen tilasta haluttiin varmistua erikseen Ansiblen avulla. Tässä hyödynnettiin Ansiblen dokumentaatiosta löytynyttä `ansible.builtin.systemd` -moduulia, jonka avulla voitiin varmistaa, että kyseinen verkkopalvelin on aktiivisena. Tiedostoon `main.yml` tehty lisäys oli seuraavanlainen:  

```
- name: Ensure Apache2 is running
  ansible.builtin.systemd_service:
    state: started
    name: apache2
```
Playbookin ajamisen yhteydessä saatiin toivotun kaltainen ilmoitus:

<img width="586" height="88" alt="Screenshot 2026-11-04 193703" src="https://github.com/user-attachments/assets/9f6cc73c-f2e4-41f8-b344-3f520e8bfe4f" />


### Lähteet

Karvinen, Tero 2026. Hello Ansible. Luettavissa: https://terokarvinen.com/hello-ansible/. Luettu: 30.10.2026. 

Karvinen, Tero 2026. Palvelinten hallinta -kurssi. Luettavissa: https://terokarvinen.com/palvelinten-hallinta/. Luettu: 30.10.2026. 

Karvinen, Tero 2026. Passwordless Sudo with Ansible. Luettavissa: https://terokarvinen.com/passwordless-sudo-with-ansible/. Luettu: 3.11.2026.

Karvinen, Tero 2026. SSH public key - Login without password. Luettavissa: https://terokarvinen.com/ssh-public-key-login-without-password/. Luettu: 30.10.2026. 

Karvinen, Tero 2026. Sudo without password. Luettavissa: https://terokarvinen.com/passwordless-sudo/. Luettu: 3.11.2026. 

Red Hat 2023. Linux file permissions explained. Luettavissa: https://www.redhat.com/en/blog/linux-file-permissions-explained. Luettu: 4.11.2026.

Yaml Validate 2026. YAML Multiline Strings: When to Use |, >, |-, and >-. https://yamlvalidate.com/blog/yaml-multiline-strings/. Luettu: 4.11.2026.
