Tämä dokumentti on kirjoitettu vastauksena [Tero Karvisen Palvelinten hallinta -kurssin](https://terokarvinen.com/palvelinten-hallinta/) kotitehtävään _h4 Pizza Fantasia_.

Ympäristönä toimi Oraclen VirtualBoxiin asennettu Debian 13.7.0 -käyttöjärjestelmä. Virtuaalikoneen laitteistoresursseiksi on asetettu kaksi prosessoriydintä, 4 Gt keskusmuistia ja 40 Gt levytilaa. Harjoituksissa hyödynnetään lisäksi UpCloudin virtuaalipalvelinta.

#### x) [Karvinen 2023: Configuration Management of Distributed Systems over Unreliable and Hostile Networks](https://westminsterresearch.westminster.ac.uk/item/w7vvz/configuration-management-of-distributed-systems-over-unreliable-and-hostile-networks)

_täydentyy myöhemmin_
  
### a) Squid-proxyn manuaalinen asennus

Päätin asentaa Squid-daemonin, mikä on yleisesti käytetty välityspalvelin, jonka avulla on mahdollista ohjata, suodattaa ja hallita verkkoliikennettä sekä hyödyntää välimuistia. Asennus suoritettiin manuaalisesti suoraan kohdepalvelimella pakettienhallinnan kautta komennolla `sudo apt update && sudo apt install squid`.

Ensin tarkistettiin, että asennus onnistui ja daemon oli käynnissä komennolla `systemctl status squid`, minkä jälkeen varmistettiin, että daemon kuunteli oletusportissa 3128 komennolla `sudo ss -tulpn | grep squid`:

<img width="1478" height="182" alt="Screenshot 2026-11-18 152538" src="https://github.com/user-attachments/assets/2e2896ef-9cd7-4215-9de0-17b3dc47c8d0" />

Proxyn toimivuutta kokeiltiin paikallisesti `curl`-työkalulla. Testattaessa käytettiin muutamia yleisiä verkkosivuja sekä kevyitä ja selkeitä testiosoitteita, kuten `httpbin.org/ip` ja `httpbin.org/user-agent`, jotta tulosteet pysyisivät lyhyinä ja selkeinä alla olevaa kuvakaappausta varten.

Esimerkki käytetystä komennosta: `curl -x localhost:3128 http://httpbin.org/ip`

- `curl` työkalu verkkosivujen hakemiseen 
- `-x` määrittää käyttämään välityspalvelinta
- `localhost:3128` välityspalvelimen osoite ja portti 
- `http://httpbin.org/ip` käytettävä kohde-url

<img width="1686" height="394" alt="Screenshot 2026-11-18 155241" src="https://github.com/user-attachments/assets/54f49a12-7865-45fc-8831-61072de2fed4" />

Kuvassa alimpana näkyy välityspalvelimen lokitiedostoon (`/var/log/squid/access.log`) tallennettuja tietoja tehdyistä pyynnöistä. Onnistuneiden pyyntöjen perässä näkyy HTTP-tilakoodi 200 (`OK`), mikä todisti, että verkkoliikenne kulki halutulla tavalla välityspalvelimen kautta eikä suoraan internetiin. Listalla näkyvä 301-tilakoodilla (_Moved Permanently_) varustettu rivi johtui kokeilusta kohde-URL:llä `yle.fi`. Oletusarvoisesti `curl` ei seuraa uudelleenohjauksia, joten pyyntö pysähtyi tähän kyseiseen ohjausviestiin. Ohjausten seuraaminen automaattisesti edellyttäisi lisämäärettä `-L` eli komento olisi kokonaisuudessaan `curl -L  -x http://localhost:3128 yle.fi`.

Squidin konfiguraatiotiedosto sijaitsi hakemistossa `/etc/squid/squid.conf`. Vielä tässä vaiheessa siihen ei kajottu.

### b) Automatisoitu asennus Ansiblella

Asennuksen automatisointia varten virtuaalikoneella luotiin Ansibleen uusi rooli `squid` ja siihen liittyvät hakemistot.

<img width="260" height="190" alt="Screenshot 2026-11-18 163623" src="https://github.com/user-attachments/assets/e8781028-c2d8-4e59-845d-43ab1880c60e" />

Ensimmäiseksi haettiin kohdepalvelimelta `squid`-daemonin manuaalisen asennuksen yhteydessä luotu oletuskonfiguraatiotiedosto omalle virtuaalikoneelle uuden roolin `files`-kansioon komennolla `scp testuser@94.237.9.166:/etc/squid/squid.conf roles/squid/files/squid.conf`. 

`tasks`-kansion `main.yml`-tiedostossa määritettiin daemonin asennus sekä hallinnoitavan konfiguraatiotiedoston sijainti sekä paikallisesti että kohdepalvelimella. Tiedostolle asetettiin samalla asianmukaiset omistaja- ja käyttöoikeudet. Konfiguraatiotiedoston kopiointiin liitettiin myös `notify`-rivi, mikä käynnistää tarvittaessa erillisen handlerin daemonin uudelleenkäynnistämiseksi. Lopun `service`-moduulilla varmistetaan, että palvelu on käynnissä ja kytketty päälle järjestelmän käynnistyessä.

```
- apt:
    name: squid
    state: present
    update_cache: yes

- copy:
    src: squid.conf
    dest: /etc/squid/squid.conf
    owner: root
    group: root
    mode: '0644'
  notify: restart squid

- service:
    name: squid
    state: started
    enabled: yes
```

`handlers`-kansiossa olevan `main.yml`-tiedoston sisältämä koodi:

```
- name: restart squid
  ansible.builtin.service:
    name: squid
    state: restarted
```

Vielä ennen Ansible-automaation ajamista kohdassa a) manuaalisesti asennettu daemon poistettiin puhtaalta pöydältä aloittamista varten komennolla `sudo apt purge squid`.

Ansiblen playbook-tiedostoon `site.yml` lisättiin suoritettavien roolien listaan kyseinen uusi rooli `squid`, minkä jälkeen ajettiin `ansible-playbook site.yml` ja varmistuttiin tulosteesta sekä palvelimen päässä, että asennus oli onnistunut.

<table>
  <tr>
    <td><img src="https://github.com/user-attachments/assets/d90d7211-2add-4800-9d25-f915a7ef72ef" width="100%" /></td>
    <td><img src="https://github.com/user-attachments/assets/74ab5c3f-faad-40f9-885d-6eae7a1e1798" width="100%" /></td>
  </tr>
</table>

### c) Asetustiedoston muuttaminen

Seuraavaksi tarkoituksenani oli pyrkiä todentamaan proxyn toimivuus suorittamalla ulkoinen yhteystesti. Tämä kuitenkin pysähtyi heti herjaan `Could not connect to server`. Syyksi paljastui UpCloudin palomuurisäännöt, jotka estivät kaikki yhteydet porttiin 3128. Näiden sääntöjen muokkaaminen ei ollut UpCloudin maksuttoman kokeilutilauksen puitteissa mahdollista, joten palomuurin kiertämiseksi ja porttien vaihtamisen sijaan päätin luoda ssh-tunnelin paikallisen koneen ja palvelimen välille avaamalla uuteen terminaali-ikkunaan komennon `ssh -N -L 3128:localhost:3128 testuser@94.237.9.166` ja jättäen kyseinen ikkuna avoimeksi taustalle. Määritys `-L 3128:localhost:3128` ohjaa paikallisen koneen portin 3128 suoraan etäpalvelimen porttiin 3128. `-N` puolestaan pitää ikkunan pelkkänä taustatunnelina eikä avaa erillistä uutta komentorivi-istuntoa. (DevCloudLab 2026.)

Tämän jälkeen yhteys toimi. Epäonnistuneen komennon muoto:  
`curl -x http://94.237.9.166:3128 httpbin.org/user-agent`   
Muuttui SSH-tunnelin myötä muotoon:  
`curl -x localhost:3128 httpbin.org/user-agent`.   

Seuraavaksi paikallisella virtuaalikoneella muokattiin `squid.conf`-tiedostoa niin, että estettäisiin sosiaalisen median palvelut. Tiedostoon lisättiin rivit:

```
acl domain_blacklist dstdomain "/etc/squid/domain_blacklist.txt"
http_access deny domain_blacklist
```

Ensimmäisellä rivillä määriteltiin pääsylista (_access control list_), joka lukee estettävät verkkotunnukset määritetyssä polussa sijaitsevasta tiedostosta. Tässä tapauksessa kyseiseen uuteen tekstitiedostoon lisättiin `.facebook.com`, `.instagram.com` ja `.tiktok.com`. Toisella rivillä otettiin kyseinen pääsylista käyttöön kieltämällä kaikki pyynnöt, jotka vastaavat estolistalla olevia verkkotunnuksia. Uuden tiedoston vuoksi `tasks/main.yml` lisättiin kyseisen tiedoston kohdepalvelimelle siirtävä lohko: 

```
- copy:
    src: domain_blacklist.txt
    dest: /etc/squid/domain_blacklist.txt
    owner: root
    group: root
    mode: '0644'
  notify: restart squid
```

Lopuksi konfiguraatio vietiin palvelimelle Ansiblella, ja estolistan toimivuus varmistettiin testikomennolla, joka palautti odotetusti `403 Forbidden` -virheen eli välityspalvelin kieltäytyi käsittelemästä pyyntöä sivustolle facebook.com.

<img width="902" height="62" alt="Screenshot 2026-11-18 195748" src="https://github.com/user-attachments/assets/cb6f4f71-72fc-4d8c-9dbf-f0e91f5e9754" />


### d) Rikkinäiset asetukset

Tämän jälkeen palvelimella suoritettiin paikallisesti tarkoituksella kokonaisuutta hajottavia toimia. Asetustiedosto `squid.conf` poistettiin kokonaan, minkä jälkeen ohjelman uudelleenkäynnistyksen ja tilan tarkistamisen yhteydessä törmättiin odotetun kaltaiseen vikailmoitukseen:

<img width="1088" height="196" alt="Screenshot 2026-11-18 205711" src="https://github.com/user-attachments/assets/e64b75bf-7c6f-4d98-9508-bd116a49c25a" />  

Asia korjaantui nopeasti normaaliksi ajamalla jälleen hallinnoivalla koneella `ansible-playbook site.yml`, jolloin puuttuva tiedosto luotiin uudelleen ja ohjelma käynnistettiin uudelleen:

<table>
  <tr>
    <td><img src="https://github.com/user-attachments/assets/f457781b-275d-45c1-82e9-3e665692c2ac" width="100%" /></td>
    <td><img src="https://github.com/user-attachments/assets/71f73d45-2b81-49fb-a838-24415907a5fd" width="100%" /></td>
  </tr>
</table>


### e) Idempontentti

Määritysten idempotenssi on helppo todeta ajamalla playbook-komento uudelleen. Tässä tapauksessa, koska järjestelmä oli jo valmiiksi halutussa tilassa eikä muutoksia tarvittu, Ansiblen suorituksen yhteenvedon `changed=0` osoittaa, että toistuvat ajot eivät aiheuta tarpeettomia muutoksia.

<img width="970" height="84" alt="Screenshot 2026-11-18 211249" src="https://github.com/user-attachments/assets/4813fb89-1207-4bbc-b058-d0d32fd32886" />


### Lähteet

DevCloudLab. SSH Tunneling Explained: -L, -R, and -D Port Forwarding. Luettavissa: https://devcloudlab.com/blog/ssh-tunneling-explained/. Luettu: 18.11.2026.

Karvinen, Tero 2026. Hello Ansible. Luettavissa: https://terokarvinen.com/hello-ansible/. Luettu: 17.11.2026. 

Karvinen, Tero 2026. Palvelinten hallinta -kurssi. Luettavissa: https://terokarvinen.com/palvelinten-hallinta/. Luettu: 15.11.2026. 

Red Hat Documentation. 16.4. Configuring a Domain Blacklist in Squid. Luettavissa: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/networking_guide/configuring-a-domain-blacklist-in-squid. Luettu 18.11.2026.

Ubuntu Server documentation. About Squid proxy servers. Luettavissa: https://ubuntu.com/server/docs/explanation/web-services/about-squid-proxy-servers/. Luettu 18.11.2026.
