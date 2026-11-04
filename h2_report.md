#### Raportti on keskeneräinen ja valmistuu myöhässä lopulliseen muotoonsa viimeistään klo 19:30 mennessä. 

Tämä dokumentti on kirjoitettu vastauksena [Tero Karvisen Palvelinten hallinta -kurssin](https://terokarvinen.com/palvelinten-hallinta/) kotitehtävään _h2 Voileipä_.

Ympäristönä toimii Oraclen VirtualBoxiin asennettu Debian 13.7.0 -käyttöjärjestelmä. Virtuaalikoneen laitteistoresursseiksi on asetettu kaksi prosessoriydintä, 4 Gt keskusmuistia ja 40 Gt levytilaa. Harjoituksissa hyödynnetään lisäksi UpCloudin virtuaalipalvelinta.

### x)

### a) & b) Salasanaton tunnus ja etähallinta Ansiblella

Käytössä olleelle etäpalvelimelle luotiin uusi käyttäjä `testuser`, jolle konfiguroitiin valmiiksi SSH-avaimella kirjautuminen sekä oikeus käyttää sudo-komentoja ilman salasanaa. Prosessi automatisoitiin Ansible-hallintatyökalulla.

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

### d)

### e) 

### Lähteet

Karvinen, Tero 2026. Hello Ansible. Luettavissa: https://terokarvinen.com/hello-ansible/. Luettu: 30.10.2026. 

Karvinen, Tero 2026. Palvelinten hallinta -kurssi. Luettavissa: https://terokarvinen.com/palvelinten-hallinta/. Luettu: 30.10.2026. 

Karvinen, Tero 2026. Passwordless Sudo with Ansible. Luettavissa: https://terokarvinen.com/passwordless-sudo-with-ansible/. Luettu: 30.10.2026.

Karvinen, Tero 2026. SSH public key - Login without password. Luettavissa: https://terokarvinen.com/ssh-public-key-login-without-password/. Luettu: 30.10.2026. 

Karvinen, Tero 2026. Sudo without password. Luettavissa: https://terokarvinen.com/passwordless-sudo/. Luettu: 30.10.2026. 



