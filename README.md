Jenkins


Ce este Jenkins?
Jenkins este un automation server bazat pe Java cu care putem defini continuous integration si continuous delivery (CI/CD) jobs sau pipelines. 
Continuous integration (CI) este o practica DevOps in care membrii unei echipe commit modificari ale codului regulat in GIT, dupa care porneste un build si ruleaza teste automate. Continuous delivery (CD) este tot o practica DevOps prin care modificarile codului sunt deploy-ate in productie.

Cu ce ne va ajuta pe noi Jenkins?
Vom automatize un Deploy Local AI Stack cu Podman.

Acest tutorial este salvat in git in fisierul README.md pe acest URL: https://github.com/bogdanobogeanu/uvt-project/blob/main/README.md
O sa va fie de ajutor sa accesati URL-ul pe masina virtuala si sa copiati comenzile din pagina web in terminal decat sa tastati fiecare comanda manual.

Instalare Jenkins intr-o masina virtuala care ruleaza Rocky Linux 9

Confirmați că system repositories funcționează (dupa pornirea masinii virtuale, la prima executare a comenzii sudo, o sa va ceara parola userului. In cazul meu, userul este student, la voi poate fi diferit):
sudo dnf repolist

Pasul 1: Instalați OpenJDK 21 pe Rocky Linux 9

Listați versiunile disponibile ale JDK pe Rocky Linux 9:
sudo dnf search java-*-openjdk
 
O să instalăm pachetul java-21-openjdk:
sudo dnf -y install java-21-openjdk
 
La finalul instalarii o sa vedem mesajul Complete! Si cateva detalii despre pachetele instalate, java si dependintele necesare:
 
Verificam versiunea de java instalata default pe masina noastra virtuala:
java -version
 
Pasul 2: Adăugați repository-ul Jenkins la Rocky Linux

Echipa Jenkins menține un repository cu pachete RPM pentru Jenkins. Vom adăuga acest repository, apoi vom instala pachete din el.
Utilizați comanda wget pentru a descărca fișierul jenkins.repo și plasați-l în directorul corect:
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
 
De asemenea, importați cheia GPG folosită pentru a semna pachetele Jenkins:
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
 
Verificam daca repository-ul Jenkins este disponibil local:
sudo dnf repolist
 
Putem vedea pe ultima linie, coloana intai, repo id numit “jenkins”.

Step 3: Instalare Jenkins Server pe Rocky Linux 9
Comanda de instalare a lui Jenkins:
sudo dnf -y install jenkins
 
Dupa instalare, o sa modificam portul default pe care ruleaza Jenkins (8080) cu portul 8081 pentru a putea rula fara probleme containerele viitoare:
Primul pas este sa activam Jenkins sa porneasca la start-ul masinii virtuale:
sudo systemctl enable jenkins
 
Apoi putem rula comanda de editare a configuratiei de start a lui Jenkins:
sudo vi /etc/systemd/system/multi-user.target.wants/jenkins.service
 
Cautam dupa variabila Environment sau dupa JENKINS_PORT, editam portul din 8080 
 
in 8081
 
Salvam fisierul si apoi rulam comanda:
sudo systemctl daemon-reload
 

Porniți serviciul Jenkins pe Rocky Linux 9 (dupa ce rulati comanda de start, asteptati in jur de 1 minut ca Jenkins sa porneasca):
sudo systemctl start jenkins
 
Verificam daca Jenkins intr-adevar ruleaza pe masina noastra virtuala:
sudo systemctl status jenkins
 
Putem observa cuvintele de culoare verde: active (running). Asta ne spune ca Jenkins este pornit. Mai putem vedea in partea de logs din poza “Started Jenkins Continuous Integration Server”. Daca din anumite motive nu va porni, o sa vedeti un cuvant de culoarea rosie care va incepe probabil cu cuvantul “fail” sau “failed” si o minima explicatie in partea de loguri.

Pasul 4: Configurați Jenkins pe Rocky Linux 9 din interfața web
După instalare și pornirea serviciului, mergeți la consola browserului web pe URL:
http://localhost:8081
Ar trebui să vedeti o pagină de bun venit și instructiuni despre cum să obțineți parola inițială de administrator:
 
Ca sa vedem parola initiala, trebuie sa efectuam comanda cat pe fisierul /var/lib/jenkins/secrets/initialAdminPassword (Parola initiala e diferita la fiecare instalare, cand instalati voi, o sa aveti o alta parola initiala):
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
 
Copy si paste la parola in pagina Jenkins in casuta Administrator Password si apoi apasati butonul Continue:
 
Dupa ce ati adaugat parola, O sa apara fereastra urmatoare, unde selectati “Select plugins to install”.  

La urmatorul pas, o sa vedem ca Jenkins instaleaza cateva pluginuri automat:
 
Asteptam sa termine de instalat toate pluginurile necesare.

Dupa acest pas, o sa ne configuram userul de administrare si apasam butonul Save and Continue: 
Dupa acest pas, o sa apara pasul de Instance Configuration, nu modificam nimic, doar apasam pe Save and Finish. 
Apasam pe butonul de Start using Jenkins:
 
Am setat Jenkins si putem sa incepem sa il folosim. Urmatoarea poza ne arata prima pagina care apare dupa ce am terminat setup-ul.  


In acest moment al instalarii de Jenkins, consideram ca aveti deja instalat pe masina virtuala, “podman” si acesta este setat sa porneasca la startup.

Pasi necesari pentru a pregati Jenkins pentru a rula pipeline-ul viitor
Login as root:
sudo su -
 
Activam shell-ul userului Jenkins:
usermod -s /bin/bash jenkins

Drepturi pentru userul Jenkins sa execute comenzi fara sa ceara parola:
echo "jenkins ALL=(root) NOPASSWD: ALL " >> /etc/sudoers.d/jenkins



Jenkins automation

Exercițiu Jenkins Pipeline – Deploy Local AI Stack cu Podman

În acest exercițiu veți crea un Jenkins Pipeline care automatizează deploy-ul unui stack AI local, format din două componente:
llama-server – un server bazat pe llama.cpp care încarcă un model de limbaj (LLM) și expune o API compatibilă OpenAI
open-webui – o interfață web prietenoasă prin care puteți interacționa cu modelul, similar cu ChatGPT
Modelul folosit este Qwen2.5 0.5B Instruct (quantizat Q4), un model mic și rapid, potrivit pentru rulat local fără GPU.

Ce face pipeline-ul pas cu pas
Stage 1 – Create /opt/models directory
Creează directorul /opt/models unde va fi stocat fișierul modelului LLM. 
Stage 2 – Create podman network
Creează o rețea virtuală izolată numită llm-net prin care cele două containere vor comunica între ele. Dacă rețeaua există deja, pasul este sărit.
Stage 3 – Download model
Descarcă modelul LLM (fișier .gguf) de pe Hugging Face în directorul /opt/models. Dacă modelul a fost deja descărcat anterior, pasul este sărit pentru a nu repeta un download de câteva sute de MB.
Stage 4 – Start llama-server
Pornește containerul llama-server care încarcă modelul și îl expune ca API REST pe portul 8080. Dacă există deja un container cu același nume (dintr-o rulare anterioară), acesta este oprit și șters înainte de a fi recreat.
Stage 5 – Start open-webui
Pornește containerul open-webui pe portul 3000, configurat să comunice cu llama-server prin rețeaua internă llm-net pe portul 8080. Variabila OPENAI_API_BASE_URL îi spune interfeței unde să trimită cererile către model.
Stage 6 – Verify containers
Verifică că ambele containere rulează efectiv. Dacă unul dintre ele nu a pornit, pipeline-ul eșuează și afișează log-urile containerului pentru debugging.

Start exercitiu

Pentru a crea un nou Jenkins Pipeline, executam pasii urmatori:

Click pe New Item:
Adaugati un nume pentru pipeline, exemplu “AI-deploy”, selectati Pipeline la item type apoi click pe OK
 
O sa apara pagina de configurare a pipeline, care incepe cu partea generala de setari (General)
 
Click pe Pipeline in partea stanga sus:
 
Copiati continutul fisierului Jenkinsfile si ii dati paste in zona numita Script apoi click pe Save (Jenkinsfile se poate copia din https://github.com/bogdanobogeanu/uvt-project/blob/main/Jenkinsfile sau din proiectul MLOPS-Project)
 
Dupa Save, o sa ajungeti la pagina principala a noului pipeline AI-Deploy:
 
Pentru a porni pipeline AI-Deploy, click pe Build Now:
 
Dupa ce s-a dat click pe butonul Build Now, putem da click pe numarul #1. Acest numar reprezinta build-ul cu numarul 1 si asa accesam detaliile build-ului sau executiei pipeline:
 
Exemplu de cum arata executia cu numarul 1 a pipeline:
 
Pentru a vedea logurile executiei pipeline-ului, putem face click pe Console Output:
 
Mai jos avem un exemplu despre cum arata partea de start a logului pipeline-ului:
 
Putem face scroll pana la finalul logului, unde gasim si URL-urile care ne intereseaza:
 
O sa revenim imediat la URL-urile din poza de mai sus.
Mai exista posibilitatea ca executia pipeline-ului sa nu reuseasca, caz in care, tot aici in Console Output puteti vedea detalii despre erori si in functie de erori sau eroare sa cautati o rezolvare. In mod normal, acest pipeline o sa functioneze fara probleme, daca nu sariti vreun pas facut inainte in prezentare.

Acum sa revenim la cele doua URL-uri:

Primul URL pentru llama-server: http://localhost:8080
 
Al doilea URL pentru Open-Webui: http://localhost:3000
 
Accesarea paginilor web functioneaza cam dupa 3 minute dupa ce a terminat pipeline, exceptand primul run de pipeline unde pipeline face download la layers si dureaza oricum cateva minute bune.

Aceasta a fost partea de Automation. O sa las aici cateva detalii ajutatoare.
Pipeline file: https://github.com/bogdanobogeanu/uvt-project/blob/main/Jenkinsfile
                    
Documentation: https://github.com/bogdanobogeanu/uvt-project/blob/main/README.md

Detalii despre masina virtuala:
Operating System: Rocky Linux 9.7
Disk size: 20 GB minim, daca mai vreti sa faceti si altceva pe masina, indicat 30 GB sau mai mult.
CPU: 4 
RAM Memory: 6000 MB.
Posibil sa mearga si cu resurse mai putine, insa la primul run, load-ul creste aproximativ la 5, depinde de performantele masinii virtuale.
