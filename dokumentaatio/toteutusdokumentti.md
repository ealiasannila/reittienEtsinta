#Toteutusdokumentti reittien etsintä harjoitustyöstä.

## Ohjelman yleisrakenne:

![alt tag](luokkakaavio.png)

*luokkakaavio. Vain tärkeimmät luokat merkitty*

Ohjelman toiminta jakautuu viiteen osaan:

1. Syötteenä annettujen polygonien lukeminen. 
2. Syötteenä annettujen reittien analysointi ja eri maastotyyppien vauhtien määrittely reittien pohjalta.
3. Verkon generointi polygon aineistosta, ja kaarien painotus maastotyypeille määräytyneiden vauhtien mukaan.
4. Nopeimman reitin etsintä generoidussa verkossa.
5. Etsityn reitin kirjoittaminen tiedostoon jatkokäyttöä varten.

### 1. Polygonien lukeminen
Luokka GeoJson Lukija lukee geojson muodossa tallennetut polygonit ja muodostaa niistä Polygoni luokan olioita. 
Polygoneja voi olla kahta tyyppiä: aluemaisia ja viivmaisia. AluePolygoni tarjoaa Polygoni luokkan metodien lisäksi metodin sen tarkistamiseen, onko piste polygonin sisällä. Tätä tarvitaan seuraavassa vaiheessa.

### 2. Syötteenä annettujen reittien analysointi.
Luokassa Maastokirjasto pidetään kirjaa siitä, minkälaista vauhtia minkäkin tyyppisen polygonin alueella pääsee kulkemaan. Lisäksi pidetään yllä ylimääräistä maastoa, joka kuvaa kaikkea sitä, mikä ei kuulu minkään polygonin vaikutuspiirin. Maastokirjasto tarjoaa metodin jolla kirjastoon voidaan lisätä reitti, ja silloin jokaisen reitin pisteen kohdalla tarkistetaan onko se aluemaisen polygonin sisällä, tai viivmaisen polygonin lähellä. Jos näin on tallennetaan pisteen ja seuraavan pisteen välillä kuljettu vauhti kirjastoon polygonin maaston mukaan. Kun kirjastosta haetaan maaston vauhtia, palauttaa se keskiarvon kaikista maastoon tallennetuista vauhdeista.

### 3. Verkon generointi
Luokka Verkontekijä vastaa verkon generoinnista. Yleisperiaate on se, että solmut muodostuvat syötteenä annettujen polygonien solmuista, ja kaaria yritetään muodostaa jokaisesta solmusta jokaiseen solmuun. Kaarta ei kuitenkaan muodosteta, jos se leikkaa jonkin polygonin reunan. Koska kaarien muodostamisyrityksiä tulisi todella paljon jaetaan polygonit naapurustoihin niiden bounding boxin keskipisteen sijainnin mukaan. Kaaria pyritään muodostamaan vain samassa naapurustossa, sekä naapurustoa ympäröivissä naapurustoissa sijaitsevien polygonien solmujen kanssa. Myöskään kaaren ja polygonin leikkausta ei tarkisteta kuin lähinaapurustoissa. Jos kaari ei leikkaa mitään polygonia lisätään se pisteiden välille. Kaaren painoksi tulee aika-arvio, eli solmujen etäisyys (m)/vauhdilla (m/s).

### 4. Nopeimman reitin etsintä verkossa
Reitin etsintä tapahtuu luokssa Verkko A* algoritmia käyttäen. A* toteutus käyttää minimikekoa ja heuristiikkafunktiona pisteen etäisyyttää maalisolmuun * minimivauhti. Verkko käyttää vieruslistoja. Päädyin tähän ratkaisuun todettuani suorituskykytestauksessa sen olevan huomattavasti tehokkaampaa kuin vierusmatriisin käyttö.

### 5. Etsityn reitin kirjoittaminen tiedostoon.
Verkko tarjoaa metodin lyhyimmän reitin palauttamiseen Reitti oliona. Luokka GeoJsonKirjoittaja tarjoaa metodit reitin kirjoittamiseen geojson tiedostoon, jolloin sen jatkokäyttö esimerkiksi paikkatieto-ohjelmissa on helppoa.


##Aika ja tilavaativuudet:
Käytetään seuraavia merkintöjä:

|V|: verkon solmujen määrä.

|E|: kaarien määrä.

|P|: polygonien määrä.

|PV|: polygonin pisteiden määrä.

|R|: reittipisteiden määrä yhdellä reitillä.

|RK|: kaikkien reittipisteiden määrä.

|N|: naapurustojen määrä.

|NV|: naapurustossa olevien solmujen määrä.

### Reitin analysointi ja maastokirjaston luonti
* Reitti
* Maastokirjasto
* Polygoni
* Aluepolygoni

Seuraavat operaatiot ovat selkeästi aikavaativuudeltaan O(1), sillä nissä ei ole mitään toistoa tai rekursiota.
* Reittiluokan metodit
* Maastokirjaston metodit lukuunottamatta reitin ja reittien lisäystä.
* Polygonin tarjoama pisteenEtaisyysJanasta()

Piste polygonin sisällä:

    pisteSisalla(double pLat, double pLon) {
        if (pLat < this.latmin || pLat > this.latmax || pLon < this.lonmin || pLon > this.lonmax) { 					:vakio
            return false;																								:vakio
        }
        int leikkaukset = 0;																							:vakio

        for (int i = 0; i < this.lat.length; i++) {																		:O(|PV|)
            int loppu = i + 1;																							:vakio
            if (i == this.lat.length - 1) { 																			:vakio
                loppu = 0;																								:vakio
            }

            if ((this.lat[i] <= pLat && this.lat[loppu] > pLat) || (this.lat[i] > pLat && this.lat[loppu] <= pLat)) {	:vakio
                double osuusViivasta = (pLat - this.lat[i]) / (this.lat[loppu] - this.lat[i]);							:vakio
                if (pLon < this.lon[i] + osuusViivasta * (this.lon[loppu] - this.lon[i])) {								:vakio
                    leikkaukset++;																						:vakio
                }

            }
        }
        return leikkaukset % 2 != 0;																					:vakio
    }
    
Pisteen sisällä olemisen tarkistamisen aikavaativuus on siis O(|PV|)


Pisteen etäisyys polygonista

    pisteenEtaisyys(double lat, double lon) {
        double etaisyys = Double.MAX_VALUE;																			:vakio
        for (int i = 0; i < this.lat.length-1; i++) {																:O(|PV|)
            double ehdokas = this.pisteJanasta(lat, lon, this.lat[i], this.lon[i], this.lat[i+1], this.lon[i+1]);	:vakio
            if (etaisyys > ehdokas) {																				:vakio
                etaisyys = ehdokas;																					:vakio
            }   
        }
        return etaisyys;																							:vakio
    }

Reitin lisäys:
    
    public void lisaaReitti(Reitti reitti, Lista<Polygoni> polygonit) {
        for (int i = 0; i < reitti.getAika().length - 1; i++) {									:O(|R|)		
            for (int j = 0; j <= polygonit.koko(); j++) {											:O(|P|)
                if (j == polygonit.koko()) { 															:vakio
                    this.lisaaVauhti(this.vauhtiMaastossa.length - 1, reitti.vauhti(i, i + 1));			:vakio
                    break;
                }

                if (polygonit.ota(j).getClass().equals(AluePolygoni.class)) {							:vakio
                    AluePolygoni aluepoly = (AluePolygoni) polygonit.ota(j);							:vakio
                    if (aluepoly.pisteSisalla(reitti.getLat()[i], reitti.getLon()[i])) {				:O(|PV|)
                        this.lisaaVauhti(polygonit.ota(j).getMaasto(), reitti.vauhti(i, i + 1));		:vakio
                        break;
                    }
                } else { 
                    if (polygonit.ota(j).pisteenEtaisyys(reitti.getLat()[i], reitti.getLon()[i]) < 4) { :O(|PV|)
                        this.lisaaVauhti(polygonit.ota(j).getMaasto(), reitti.vauhti(i, i + 1));		:vakio
                        break;

                    }
                }
            }

        }

    }
    
Reitin lisäyksen aikavaativuus on siis O(|R||P||PV|) = O(|R||V|)

**kaikkien Reittien lisäyksen aikavaativuus on vastaavasti O(|RK||V|)**

### Verkon generointi
osallistuvat luokat:
* verkko
* polygoni
* verkontekija
* maastokirjasto
* lista

Seuraavat operaatiot ovat selkeästi aikavaativuudeltaan O(1), sillä nissä ei ole mitään toistoa tai rekursiota.
* listan operaatiot (kasvata on aikavaativuudeltaan O(n), mutta sitä kutsutaan vain harvoin)
* verkon lisää kaari operaatio
* Polygonin metodit janatLeikkaavat(), janatKohtaavatPaassa() sekä kiertosuunta()
* maastokirjaston haeMaastolla() ja haeVauhti() metodit
* verkontekijan lisaaPolygoni(), naapurustoUlkona() sekä rajat()

Leikkaako kaari polygonin
    janaLeikkaaPolygonin(double lat1, double lon1, double lat2, double lon2) {
        if ((lat1 < this.latmin && lat2 < this.latmin) || (lat1 > this.latmax && lat2 > this.latmax)					:vakio
                || (lon1 < this.lonmin && lon2 < this.lonmin) || (lon1 > this.lonmax && lon2 > this.lonmax)) {
            return false;																								:vakio
        }
        for (int i = 0; i < this.lat.length; i++) {																		:O(|PV|)
            int loppu = i + 1;																								:vakio
            if (i == this.lat.length - 1) { //viimeisestä pisteesta takaisin ekaan											:vakio
                loppu = 0;																									:vakio
            }
            if (janatLeikkaavat(lat1, lon1, lat2, lon2, this.lat[i], this.lon[i], this.lat[loppu], this.lon[loppu])) {		:vakio
                return true;																								:vakio
            }
        }

        return false;																									:vakio

    }
Aikavaativuus on siis O(|PV|)

Naapurustojen alustus:
    alustaNaapurustot(int n) {
        this.naapurustot = new Lista[n][n];				:vakio

        for (int i = 0; i < n; i++) {					:O(sqrt(|N|))
            for (int j = 0; j < n; j++) {					:O(sqrt(|N|))
                this.naapurustot[i][j] = new Lista(8);			:vakio

            }
        }
    }
Aikavaativuus on siis O(|N|)

    lisaaPolygonit(Lista<Polygoni> polygonit) {
        for (int i = 0; i < polygonit.koko(); i++) {	O(|P|)
            this.lisaaPolygoni(polygonit.ota(i));			:vakio
        }
    }
    
Aikavaativuus on siis O(|P|)

Kaaren asettaminen
    asetaKaari(Verkko verkko, int maasto, int idp, double latp, double lonp, int idk, double latk, double lonk, 
    		   int naapurustoX, int naapurustoY, int kohdenaapurustoX, int kohdenaapurustoY) {
        if (Apumetodit.pisteSama(latp, lonp, latk, lonk)) {									:vakio
            verkko.lisaaKaari(idp, idk, maasto, latp, lonp, latk, lonk, false);				:vakio
            return;
        }

        int[] xRajat = this.rajat(kohdenaapurustoX - naapurustoX);							:vakio
        int[] yRajat = this.rajat(naapurustoY - kohdenaapurustoY);							:vakio

        for (int i = yRajat[0]; i <= yRajat[1]; i++) {										:vakio for looppi käydään läpi 3-5 kertaa
            if (this.naapurustoUlkona(i, naapurustoY)) {										:vakio
                continue;
            }
            for (int j = xRajat[0]; j <= xRajat[1]; j++) {									:vakio for looppi käydään läpi 3-5 kertaa	
                if (this.naapurustoUlkona(j, naapurustoX)) {									:vakio
                    continue;
                }
                Lista<Polygoni> ruutu = this.naapurustot[naapurustoY + i][naapurustoX + j];		:vakio

                for (int k = 0; k < ruutu.koko(); k++) {										:O(|P|)
                    if (ruutu.ota(k).janaLeikkaaPolygonin(latp, lonp, latk, lonk)) {				:O(|PV|)
                        return;
                    }

                }
            }
        }

        verkko.lisaaKaari(idp, idk, maasto, latp, lonp, latk, lonk, false);					:vakio

    }
    
Aikavaativuus on siis O(|NV|)

Aluemaisen Polygonin kaarien asetus
    
    asetaAlueminenKohde(Verkko verkko, Polygoni kohde, int id, int lahtosolmuIndeksi, double lat, double lon, int naapurustoX, int naapurustoY) {
        for (int l = 0; l < kohde.getId().length; l++) {												:O(|PV|)
            if (kohde.getId()[l] != id) {																	:vakio
                if (lahtosolmuIndeksi == l - 1 || lahtosolmuIndeksi == l + 1) { 							:vakio
                    verkko.lisaaKaari(id, kohde.getId()[l], kohde.getMaasto(), lat, lon, 					:vakio
                    				  kohde.getLat()[l], kohde.getLon()[l], true);										
                } else if ((lahtosolmuIndeksi == kohde.getId().length - 1 && l == 0) 						:vakio
                			|| (lahtosolmuIndeksi == 0 && l == kohde.getId().length - 1)) {
                    verkko.lisaaKaari(id, kohde.getId()[l], kohde.getMaasto(), lat, lon, 					:vakio
                    				  kohde.getLat()[l], kohde.getLon()[l], true);
                } else { //kaari alueen läpi
                    asetaKaari(verkko, kohde.getMaasto(), id, lat, lon, kohde.getId()[l], kohde.getLat()[l], :O(|NV|) 
                    		   kohde.getLon()[l], naapurustoX, naapurustoY, naapurustoX, naapurustoY);
                }
            }
        }
    }

Aikavaativuus on siis O(|PV||NV|)


Viivmaisen polygonin kaarien asettaminen
    
    asetaViivamainenKohde(Verkko verkko, int id, int lahtosolmuIndeksi, Polygoni kohde, 
    					  double lat, double lon,int naapurustoX,int naapurustoY) {
        for (int l = 0; l < kohde.getId().length; l++) {												:O(|PV|)
            if (kohde.getId()[l] != id) {																	:vakio
                if (lahtosolmuIndeksi == l - 1 || lahtosolmuIndeksi == l + 1) { 							:vakio
                    verkko.lisaaKaari(id, kohde.getId()[l], kohde.getMaasto(), lat, lon, 					:vakio
                    			      kohde.getLat()[l], kohde.getLon()[l], true);
                } else { //kaari alueen läpi						
                    this.asetaKaari(verkko, -1, id, lat, lon, kohde.getId()[l], kohde.getLat()[l], 			:O(|NV|) 
                    				kohde.getLon()[l], naapurustoX, naapurustoY, naapurustoX, naapurustoY);
                }
            }
        }

    }
Aikavaativuus on siis O(|PV||NV|)

Kahden eri polygonin välisen kaaren asettaminen

    asetaTuntemattomanLapi(Verkko verkko, int id, Polygoni kohde, double lat, double lon, 
						   int naapurustoX, int naapurustoY, int kohdenaapurustoX, int kohdenaapurustoY) {
        for (int l = 0; l < kohde.getId().length; l++) {														:|PV|
            if (kohde.getId()[l] != id) {																			:vakio
                this.asetaKaari(verkko, -1, id, lat, lon, kohde.getId()[l], kohde.getLat()[l], kohde.getLon()[l], 	::O(|NV|) 
                				naapurustoX, naapurustoY, kohdenaapurustoX, kohdenaapurustoY);
            }
        }
    }
    
Aikavaativuus on siis O(|PV||NV|)

Solmusta lähtevien kaarten asettaminen:
    asetaKaaret(Verkko verkko, int lahtosolmuIndeksi, int id, double lat, double lon, 
    			int naapurustoX, int naapurustoY, Polygoni lahto) {
        for (int i = -2; i <= 2; i++) {																	:vakio
            if (this.naapurustoUlkona(i, naapurustoY)) {													:vakio
                continue;
            }
            for (int j = -2; j <= 2; j++) {																	:vakio
                if (this.naapurustoUlkona(j, naapurustoX)) {													:vakio
                    continue;
                }

                for (int k = 0; k < this.naapurustot[naapurustoY + i][naapurustoX + j].koko(); k++) {			:O(|P|) polygonien määrä naapurustossa
                    Polygoni kohde = this.naapurustot[naapurustoY + i][naapurustoX + j].ota(k);						:vakio
                    if (kohde.getId()[0] == lahto.getId()[0]) {														:vakio
                        if (lahto.getClass() == AluePolygoni.class) {												:vakio
                            this.asetaAlueminenKohde(verkko, kohde, id, lahtosolmuIndeksi, 							:O(|PV||NV|)
                            						 lat, lon, naapurustoX, naapurustoY);
                        } else {
                            this.asetaViivamainenKohde(verkko, id, lahtosolmuIndeksi, kohde, 						:O(|PV||NV|)
                            						   lat, lon, naapurustoX, naapurustoY);
                        }
                    } else { 
                        this.asetaTuntemattomanLapi(verkko, id, kohde, lat, lon, naapurustoX, naapurustoY, 			:O(|PV||NV|)
                        							naapurustoX + j, naapurustoY + i);
                    }
                }

            }
        }
    }

Aikavaativuus on siis O(|P||PV||NV|) eli O(|NV|^2) 
Verkon luonti


    luoVerkko(Verkko verkko) {
        for (int i = 0; i < this.naapurustot.length; i++) {								:O(sqrt(|N|))
            for (int j = 0; j < this.naapurustot[i].length; j++) {							:O(sqrt(|N|))
                for (int k = 0; k < this.naapurustot[i][j].koko(); k++) {						:O(|P|) -joka polygonille koska edelliset
                    Polygoni polygoni = this.naapurustot[i][j].ota(k);								:vakio
                    for (int l = 0; l < polygoni.getId().length; l++) {								:O(|PV|) - joka solmulle
                        this.asetaKaaret(verkko, l, polygoni.getId()[l], 								:O(|NV|^2)
                                         polygoni.getLat()[l], polygoni.getLon()[l], j, i, polygoni);
                    }
                }
            }
        }
    }

Verkon luonnin aikavaativuus on siis O(|V|(|NV|^2)). |NV|  on lähes vakio koska naapurustojen määrä määritellään Verkontekijä luokassa siten, että jokaiseen naapurustoon tulee enintään noin 300 solmua. niimpä:
**Verkon generoinnin aikavaativuus on O(|V|)**
 
### Reitin etsintä
Reitin etsintään käytetään A* algoritmia minimikeolla ja vieruslistoilla. 
osallistuvat luokat:
* Verkko
* Minimikeko
* Pino


Seuraavat operaatiot ovat selkeästi aikavaativuudeltaan O(1), sillä nissä ei ole mitään toistoa tai rekursiota.
* loysaa(solmu, naapuri, naapurinindeksi)
* arvio(solmu)
* minimikeon konstruktori
* minimikeon vanhempi ja lapsi operaatiot
* minimikeon vaihda(a, b) operaatio
* minimikeon tyhja() operaatio
* pinon lisää ja ota operaatiot

Alustusoperaatio:

    alustus(int lahtosolmu, int maalisolmu) {
        this.maalisolmu = maalisolmu;					:vakio
        this.lahtosolmu = lahtosolmu;					:vakio
        for (int i = 0; i < this.alkuun.length; i++) {  :|V|
            this.alkuun[i] = Double.MAX_VALUE;			  :vakio
            this.polku[i] = -1;							  :vakio
            this.loppuun[i] = this.arvioiEtaisyys(i);	  :vakio

        }
        this.alkuun[lahtosolmu] = 0;					:vakio

    }
Alustuksen aikavaativuus on siis O(|V|)
    
Minimikeko heapify:

	heapify(int i) {														
        int v = this.vasenLapsi(i);											:vakio
        int o = this.oikeaLapsi(i);											:vakio
        int pienempi;														:vakio

        if (o <= this.keonKoko) {											:vakio
            if (this.arvio(this.keko[v]) < arvio(this.keko[o])) {			:vakio
                pienempi = v;												:vakio
            } else {														:vakio
                pienempi = o;												:vakio
            }

            if (arvio(this.keko[i]) > arvio(this.keko[pienempi])) {			:vakio
                this.vaihda(i, pienempi);									:vakio
                this.heapify(pienempi);										:suoritetaan enintään keon korkeuden eli log(|V|) kertaa
            }
        } else if (v == this.keonKoko && arvio(this.keko[i]) > arvio(this.keko[v])) {	:vakio
            this.vaihda(i, v);												:vakio
        }
    }
Heapifyn aikavaativuus on siis O(log|V|)

Minimikeko paivita:

    paivita(int solmu) {
        int i = this.kekoindeksit[solmu];											:vakio
        while (i > 1 && arvio(this.keko[this.vanhempi(i)]) > arvio(this.keko[i])) {	:suoritetaan enintään keon korkeuden eli log(|V|) kertaa
            this.vaihda(i, this.vanhempi(i));										:vakio
            i = this.vanhempi(i);													:vakio
        }
    }
Päivitysoperaation aikavaativuus on siis O(log|V|)


Minimikeko lisaa:

    lisaa(int solmu) {
        this.keonKoko++;														:vakio
        int i = this.keonKoko;													:vakio
        while (i > 1 && arvio(this.keko[this.vanhempi(i)]) > arvio(solmu)) {	suoritetaan enintään keon korkeuden eli log(|V|) kertaa
            this.keko[i] = keko[this.vanhempi(i)];								:vakio
            this.kekoindeksit[this.keko[i]] = i;								:vakio
            i = this.vanhempi(i);												:vakio
        }
        this.keko[i] = solmu;													:vakio
        this.kekoindeksit[solmu] = i;											:vakio

    }    

Lisäyksen aikavaativuus on siis O(log|V|). 

Minimikeko otapienin:

    otaPienin() {

        int eka = this.keko[1];						:vakio
        this.keko[1] = this.keko[this.keonKoko];	:vakio
        this.kekoindeksit[this.keko[1]] = 1;		:vakio

        this.keonKoko--;							:vakio
        this.heapify(1);							:O(log|V|)
        return eka;									:vakio
    }
    
pienimmän ottamisen aikavaativuus on siis O(log|V|)


A*:

    aStar() {

        MinimiKeko keko = new MinimiKeko(this.alkuun.length) { arvio() funktio}; 	:vakio

        for (int i = 0; i < this.alkuun.length; i++) {								:O(|V|)
            keko.lisaa(i);																:O(log|V|)
        }

        while (!keko.tyhja()) {														:O(|V|)
            int solmu = keko.otaPienin();												:O(log|V|)
            if (solmu == this.maalisolmu) {												:vakio
                return true;															:vakio
            }
            for (int i = 0; i < this.vl[solmu].koko(); i++) {							:O(|solmusta lähtevien kaarien määrä|) 
                int naapuri = (int) this.vl[solmu].ota(i)[0];								:vakio
                if (loysaa(solmu, naapuri, i)) {											:vakio
                    keko.paivita(naapuri);													:O(log|V|)
                }
            }
        }
        if (this.alkuun[this.maalisolmu] == Double.MAX_VALUE) {						:vakio
            return false;															:vakio
        }

        return true;																:vakio
    }

A* aikavaativuus muodostuu siis O(|V|log|V|) osasta ja O(|E|log|V|) osasta. eli kokonaisaikavaativuus on O((|V|+|E|)log|V|).

Lyhyimmän reitin palautus on aikavaativuudeltaan lineaarinen löytyneeseen reittiin nähden. Eli pahimmassa tapauksessa O(|V|) suuntaamattomassa verkossa.

**Reittien etsinnän pahimman tapauksen aikavaativuus on siis kokonaisuudessaan O((|V|+|E|)log|V|).**


###Yhteenveto aikavaativuuksista:
* **Reittien maastokirjastoon lisääminen: O(|RK||V|)** (voisi varmaankin parantaa käyttämällä naapurustoja)
* **Verkon generointi: O(|V|)** (mutta suuret vakiokertoimet)
* **Reittien etsintä: O((|V|+|E|)log|V|).**


## Puuteet ja parannusehdotukset:

###Puuteet
* mikäli kaksi polygonia on täysin vierekkäin voi polygonin läpi muodostua kaari jonka vauhti vastaa tuntemattoman alueen vauhtia.
* mikäli aineistossa on kovin suuria polygoneja, esimerkiksi pitkiä yhtenäisiä tiepätkiä aiheuttavat ne virheitä verkon muodostuksessa. Tämä johtuu siitä että polygonit jaetaan naapurusotihin niiden bounding boxin keskipisteen perusteella, mutta suuren polygonin vaikutus voi ulottua yli kahden naapuruston päähän.
* ohjelma ei tue viivamaisia esteitä. Esimerkiksi moottoritie tai aita on käytännössä este jota ei voi ylittää. Koska referenssireitti ei kulje myöskään estettä pitkin saa este vauhdikseen minimivauhdin (= hyvin pieni), eikä sitä pitkin silloin ehdoteta reittejä. Koska viivamaisen kohteen läpi kuljettaessa liikutaan vain yhden solmun läpi, ilman että yksikään kaari olisi viivamaisen kohteen vauhdilla, voi ohjelma ehdottaa reittiä joka kulkee esteen poikki, vaikka näin ei todillisuudessa voitaisi toimia.

###Jatkokehitettävää
* Suunnasta riippuvien maastotyyppien toteuttaminen. Reittien etsinnän osalta tämä olisi varmaankin kohtalaisen helppoa, mutta reitin analysoinnin näkökulmasta mahdollisesti haastavaa.
* Usean päällekkäisen muuttujan vaikutus vauhtiin. Yhdessä ylläolevan kanssa mahdollistaisi esimerkiksi korkeusvaihteluiden huomioonottamisen.
* Ohjelmasta voisi mahdollisesti tehdä esimerkiksi QGis pluginin, jolloin reitin analysoinnin ja reittiehdotuksen antamisen, voisi tehdä paikkatieto-ohjelman sisältä graaffisesta käyttöliittymästä.




