Tämä dokumentti on kirjoitettu vastauksena [Tero Karvisen Palvelinten hallinta-kurssin](https://terokarvinen.com/palvelinten-hallinta/) kotitehtävään _h1 Hei Ansiblen maailma_.

### x) 
#### [Karvinen 2026: SSH public key - Login without password](https://terokarvinen.com/ssh-public-key-login-without-password/)
- SSH mahdollistaa turvallisen kirjautumisen palvelimille ilman salasanaa
- toimii usein taustapalveluna
- perustuu avainpareihin (julkinen ja yksityinen avain)
- avainparin luominen komennolla `ssh-keygen`
- julkinen avain kopioidaan palvelimelle komennolla `ssh-copy-id [user@ip-osoite]`
- kirjautuminen tapahtuu käyttäen `ssh [kohde]`

#### [Karvinen 2026: Hello Ansible](https://terokarvinen.com/hello-ansible/)
- Ansible mahdollistaa useiden koneiden keskitetyn hallinnan
- toimii SSH-yhteyden kautta
- määritykset kirjoitetaan koodina (IaC, Infrastructure as Code)
- Ansible vaaditaan vain master-koneella, slave-koneilla oltava SSH ja Python
- hallittavat koneet ja ryhmät lisätään tiedostoon `hosts.ini`
- koneiden roolien ajamisesta  määritetään nk. _playbookissa_ (esim. `site.yml`).
- roolien sisältämät toimet määritetään polussa `roles/[name]/tasks/main.yml`
- Ansiblen omia asetuksia säädetään tiedostolla `ansible.cfg`

### a) Debian 13 -virtuaalikoneen luominen

Tulevien harjoitusten ympäristönä toimii Oraclen VirtualBoxiin asennettu Debian 13.7.0 -käyttöjärjestelmä. Virtuaalikoneen laitteistoresursseiksi asetettiin kaksi prosessoriydintä, 4 Gt keskusmuistia ja 40 Gt levytilaa. Harjoituksissa hyödynnetään localhostia eli paikallista virtuaalikonetta testialustana. 

### b) SSH-daemonin asennus ja testaus

Ensimmäiseksi asennettiin OpenSSH-palvelinohjelmisto järjestelmään komennolla: `sudo apt-get install -y openssh-server`

Tämän jälkeen komentoa `sudo systemctl status ssh` käytettiin varmistamaan, että SSH-palvelu on virtuaalikoneella aktiivinen ja käynnistynyt automaattisesti. Mikäli daemon ei olisi käynnistynyt automaattisesti olisi tullut tarpeen hyödyntää komentoa `sudo systemctl enable --now ssh`, jota ei siis tällä kertaa tarvittu.

<img width="850" height="252" alt="Screenshot 2026-10-28 154911" src="https://github.com/user-attachments/assets/db412b0e-49f8-4679-9854-c8a7b95b0a7d" />
  
Vakuuttuminen etäyhteyden toimivuudesta todettiin kirjautumalla palvelimelle paikallisesti komennolla `ssh localhost`. Kirjautuminen onnistui salasanalla.

### c) SSH-avainkirjautuminen
Kirjautumisen automatisointi tapahtui luomalla ssh-avainpari komennolla `ssh-keygen` ja hyväksymällä oletusasetukset.

Julkisen avaimen siirto palvelimelle eli tässä yhteydessä _localhostille_ tapahtui komennolla `ssh-copy-id localhost`.

Tässä yhteydessä tuli vielä kerran syöttää käyttäjän salasana, minkä jälkeen avain tallentui onnistuneesti oikeaan paikkaan eli kotihakemiston `.ssh/authorized_keys`-tiedostoon.
 
Onnistuneesta ssh-kirjautumisesta ilman salasanakyselyä voitiin varmistua komennolla `ssh localhost`.

<img width="1408" height="462" alt="ssh_copy" src="https://github.com/user-attachments/assets/8d24c1ba-d513-4e3b-9088-2007345e135b" />

### d) Ansible ja Hello World

Tehtävä aloitettiin asentamalla palvelinten hallintaohjelmisto Ansible virtuaalikoneen ympäristöön: `sudo apt-get install ansible`.

Tämän jälkeen käyttäjän kotihakemistoon luotiin hakemisto _ansible_, jonka alle tehtiin tarvittavat tiedostot ja hakemistot. Ensimmäinen näistä oli tekstitiedosto `hosts.ini`, jossa määritetään hallittavat hostit eli koneet. Tiedostoon kirjoitettiin ensimmäiselle riville harjoituksessa käytetty kohdekone `localhost`. Lisäksi tiedostoon lisättiin [Teron ohjeiden avulla](https://terokarvinen.com/hello-ansible/#convenience-stop-whining-about-python-version) määre, mikä käskee Ansiblea käyttämään kohdekoneen paikallista Python-tulkkia, jotta voitiin välttyä tulkin versioon liittyvältä herjalta. Tiedoston sisältö oli siis:

```
localhost

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Lisäksi luotiin konfiguraatiotiedosto `ansible.cfg`, johon määritettiin oletus-inventoryksi kyseinen hosts.ini:

```
[defaults]
inventory = hosts.ini
```

Tämän myötä esimerkiksi komento `ansible all -a 'uptime' -i hosts.ini` lyheni muotoon `ansible all -a 'uptime'`. Ansible käyttää siis oletuksena tätä listaa ellei muuta ole komennossa määritetty.

Ansiblen niin kutsuttu _playbook_-tiedosto, tässä yhteydessä `site.yml`, puolestaan määrittää mitä rooleja ajetaan millekin koneille. Tässä tapauksessa asetettiin kaikille koneille hello-rooli:

```
- hosts: all
  roles: 
    - hello
```

Varsinaisen roolin toimenpiteet määritettiin hakemistossa `/ansible/roles/hello/tasks/`, jonne tehtiin varsinainen tehtävätiedosto `main.yml`. Tehtäväksi tässä yhteydessä annettiin kopioida ja luoda tiedosto polkuun `/tmp/hello-ansible` sisällöllä _Hello World!_:

```
- copy:
    dest: /tmp/hello-ansible
    content: "Hello World!\n"
```

Luotu automaatio ajettiin komennolla `ansible-playbook site.yml`. Tulosteesta voitiin varmistua, että Ansible suoritti onnistuneesti halutun muutoksen kohdekoneelle hyödyntäen SSH-yhteyttä, sillä _PLAY RECAP_ -otsakkeen alta oli nähtävissä `changed=1` eli toimenpide oli muuttanut yhtä tiedostoa kohdekoneella. 

Lopullinen varmistus siitä, että luotu tiedosto oli kulkeutunut oikeaan paikkaan ja oikealla sisällöllä saatiin komennolla `ssh localhost 'cat /tmp/hello-ansible'`.

<img width="916" height="350" alt="Screenshot 2026-10-28 164755" src="https://github.com/user-attachments/assets/b5311318-6c4f-4773-8448-6eaa180dfd1e" />


### Lähteet

Karvinen, Tero 2026. Hello Ansible. Luettavissa: https://terokarvinen.com/hello-ansible/. Luettu: 28.10.2026. 

Karvinen, Tero 2026. Palvelinten hallinta -kurssi. Luettavissa: https://terokarvinen.com/palvelinten-hallinta/. Luettu: 27.10.2026. 

Karvinen, Tero 2026. SSH public key - Login without password. Luettavissa: https://terokarvinen.com/ssh-public-key-login-without-password/. Luettu: 27.10.2026. 
