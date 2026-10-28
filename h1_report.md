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

Tehtävä aloitettiin asentamalla palvelinten hallintaohjelmisto Ansible virtuaalikoneen ympäristöön `sudo apt-get install ansible`.

Tämän jälkeen käyttäjän kotihakemistoon luotiin hakemisto _ansible_, jonka alle tehtiin tarvittavat tiedostot ja hakemistot. Ensimmäinen näistä oli tekstitiedosto `hosts.ini`, joka sisälsi vain yhden rivin tekstiä eli tässä harjoituksessa käytetyn kohdekoneen `localhost`. Lisäksi luotiin konfiguraatiotiedosto ansible.cfg, johon määritettiin oletus-inventoryksi kyseinen hosts.ini. Tiedosto `site.yml` puolestaan määrittää mitä rooleja ajetaan millekin koneille. Tässä tapauksessa kaikille koneille ajettiin hello-rooli. Hakemistoon `roles/hello/tasks/` luotiin `main.yml` tehtiin varsinainen tehtävätiedosto, joka kopioi ja luo tiedoston polkuun `/tmp/hello-ansible` sisällöllä `"Hello World!"`.

Luotu automaatio ajettiin komennolla `ansible-playbook site.yml`.

Tulosteesta voitiin varmistua, että Ansible suoritti onnistuneesti halutun laisen muutoksen kohdekoneelle hyödyntäen SSH-yhteyttä.

Lopullinen varmistus siitä, että luotu tiedosto oli kulkeutunut oikeaan paikkaan ja oikealla sisällöllä saatiin komennolla `ssh localhost 'cat /tmp/hello-ansible'`.

<img width="916" height="350" alt="Screenshot 2026-10-28 164755" src="https://github.com/user-attachments/assets/b5311318-6c4f-4773-8448-6eaa180dfd1e" />


