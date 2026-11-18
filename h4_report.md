### Tehtävä palautunut määräaikaan mennessä raakileena. Päivittyy lopulliseen muotoonsa viimeistään 18.11. klo 20.

Tämä dokumentti on kirjoitettu vastauksena [Tero Karvisen Palvelinten hallinta -kurssin](https://terokarvinen.com/palvelinten-hallinta/) kotitehtävään _h4 Pizza Fantasia_.

#### x) [Karvinen 2023: Configuration Management of Distributed Systems over Unreliable and Hostile Networks](https://westminsterresearch.westminster.ac.uk/item/w7vvz/configuration-management-of-distributed-systems-over-unreliable-and-hostile-networks)

_täydentyy myöhemmin_
  
### a) Squid-proxyn manuaalinen asennus

Päätin asentaa Squid-daemonin, mikä on yleisesti käytetty välityspalvelin, jonka avulla on mahdollista ohjata, suodattaa ja hallita verkkoliikennettä sekä hyödyntää välimuistia.

Asennus suoritettiin manuaalisesti suoraan kohdepalvelimella pakettien hallinnan kautta komennolla `sudo apt update && sudo apt install squid`.

Ensin tarkistettiin, että asennus onnistui ja daemon oli käynnissä komennolla `systemctl status squid`, minkä jälkeen varmistettiin, että Squid-daemon kuunteli oletusportissa 3128 komennolla `sudo ss -tulpn | grep squid`.

Proxyn toimivuutta testattiin suoraan etäpalvelimelta käsin curl-komennolla käyttämällä `-x`-lippua. Testattaessa käytettiin muutamia yleisiä verkkosivuja sekä kevyitä ja selkeitä testiosoitteita, kuten `httpbin.org/ip` ja `httpbin.org/user-agent`, jotta tulosteet pysyisivät lyhyinä ja selkeinä alla olevaa kuvakaappausta varten.

`curl -x localhost:3128 http://httpbin.org/ip`

`curl` käytetty työkalu 
`-x` määrittää käyttämään välityspalvelinta
`localhost:3128` välityspalvelimen osoite ja portti 
`http://httpbin.org/ip` haettava kohde-url

Välityspalvelin tallentaa tietoja tehdyistä pyynnöistä lokitiedostoon hakemistossa `/var/log/squid/access.log`. Lokista havaittiin onnistuneet pyynnöt (esim. `TCP_REFRESH_MODIFIED/200` ja `TCP_TUNNEL/200`), mikä todisti, että verkkoliikenne kulki ohjatusti Squidin läpi eikä suoraan internetiin.

Squidin konfiguraatiotiedosto sijaitsi hakemistossa `/etc/squid/squid.conf`. Vielä tässä vaiheessa siihen ei kajottu.

### b) Automatisoitu asennus Ansiblella

Asennuksen automatisointia varten virtuaalikoneella luotiin Ansibleen uusi rooli `squid` ja siihen liittyvät muut hakemistot.

<img width="260" height="190" alt="Screenshot 2026-11-18 163623" src="https://github.com/user-attachments/assets/e8781028-c2d8-4e59-845d-43ab1880c60e" />

Ensimmäiseksi haettiin kohdepalvelimelta `squid`-daemonin manuaalisen asennuksen yhteydessä luotu oletuskonfiguraatiotiedosto omalle virtuaalikoneelle uudeln roolin `files`-kansioon komennolla `scp anssi@94.237.9.166:/etc/squid/squid.conf roles/squid/files/squid.conf`. 

`tasks`-kansion `main.yml`-tiedostossa määritettiin daemonin asennus sekä siirrettävän konfiguraatiotiedoston sijainti sekä paikallisesti että kohdepalvelimella. Tiedostolle asetettiin samalla asianmukaiset omistaja- ja käyttöoikeudet. Konfiguraatiotiedoston kopiointiin liitettiin myös `notify`-rivi, joka käynnistää tarvittaessa erillisen handlerin daemonin uudelleenkäynnistämiseksi. Lopun `service`-moduulilla varmistetaan, että palvelu on käynnissä ja kytketty päälle järjestelmän käynnistyessä.

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

Ansiblen `site.yml`-tiedostoon lisättiin uudeksi rooliksi `squid`.

Vielä ennen Ansible-automaation playbookin ajamista kohdassa a) manuaalisesti asennettu daemon poistettiin puhtaalta pöydältä aloittamista varten komennolla `sudo apt purge squid`.

### c) 
_päivittyy myöhemmin_

### d)
_päivittyy myöhemmin_

### e) 
_päivittyy myöhemmin_


### Lähteet

Karvinen, Tero 2026. Hello Ansible. Luettavissa: https://terokarvinen.com/hello-ansible/. Luettu: 17.11.2026. 

Karvinen, Tero 2026. Palvelinten hallinta -kurssi. Luettavissa: https://terokarvinen.com/palvelinten-hallinta/. Luettu: 15.11.2026. 

Red Hat 2023. Linux file permissions explained. Luettavissa: https://www.redhat.com/en/blog/linux-file-permissions-explained. Luettu: 15.11.2026.

Ubuntu Server documentation. About Squid proxy servers. Luettavissa: https://ubuntu.com/server/docs/explanation/web-services/about-squid-proxy-servers/. Luettu 18.11.2026.






