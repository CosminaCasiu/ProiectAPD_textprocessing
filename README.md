Documentație Tehnică: Procesare Text și Analiza Frecvenței Cuvintelor

1. Descriere Generală
Această aplicație C++ are ca scop procesarea eficientă a unui fișier text de mari dimensiuni pentru a extrage și afișa un clasament al celor mai frecvent utilizate cuvinte, alături de numărul total de cuvinte distincte.
Aplicația se distinge prin implementarea a trei paradigme diferite de procesare, permițând compararea performanței acestora:
1.Secvențială: Procesare standard pe un singur fir de execuție.
2.Multi-threading (Memorie Partajată): Procesare paralelă folosind API-ul nativ din C++ (std::thread).
3.Procesare Distribuită (MPI): Procesare paralelă folosind standardul Message Passing Interface, destinată în mod normal clusterelor de calculatoare.
2. Arhitectura și Structuri de Date
Pentru a asigura un nivel ridicat de performanță, codul utilizează următoarele abordări algoritmice:
Hash Maps (std::unordered_map): Utilizate pentru stocarea și actualizarea frecvenței cuvintelor. Oferă un timp mediu de acces și inserție de O(1), fiind mult mai eficiente comparativ cu arborii binari (std::map, care oferă O(\log n)).
Vectori pre-alocați: Pentru a evita realocările costisitoare de memorie în timpul citirii caracterelor, se folosește cuvant.reserve(32). Astfel, memoria este reținută și refolosită via cuvant.clear().
Citire binară completă (I/O Optimization): Fișierul este citit integral într-un singur șir de caractere (string) printr-o singură operație de bloc (fin.read), minimizând epuizarile de tip I/O.
3. Detalierea Modulurilor (Implementare)
 Funcții 
citesteFisierRapid(nume_fisier): Determină dimensiunea totală a fișierului poziționând cursorul la final (seekg(0, ios::end)), alocă memoria necesară, revine la început și citește toți bytes dintr-un foc.
numaraCuvinte(text, start, end, map_cuvinte): Modulul "muncitor" (worker) principal. Primește o secțiune delimitată de text (start - end). Extrage cuvintele ignorând semnele de punctuație (isalnum), le convertește la litere mici și actualizează dicționarul primit ca parametru. Gestionează automat cazurile în care un cuvânt este tăiat între partiții (ajustând indicii start și end).
afiseazaTop10(map_cuvinte): Convertește dicționarul într-un std::vector de perechi (cuvânt - frecvență) pentru a putea fi sortat descrescător utilizând funcția custom de comparare comparareFrecventa.

Varianta 1: Procesare Secvențială
Funcție: ruleazaSecvential
Mecanism: Un singur proces/thread apelează funcția numaraCuvinte pe întregul șir de caractere (de la indexul 0 la text.length()).
Caz de utilizare: Oferă o valoare de referință (baseline) pentru a măsura eficiența metodelor paralele.
Varianta 2: Procesare Multi-threading (Memorie partajată)
Funcție: ruleazaThreads
Mecanism: 
1. Textul este împărțit în N fragmente egale (în cod, N = 4).
2. Sunt create 4 thread-uri, fiecare primind un fragment de text și un dicționar local (harti_locale[i]).
3. Thread-ul principal așteaptă finalizarea execuției prin thread.join().
4. Thread-ul principal realizează operația de reducere (merging), adunând toate dicționarele locale într-un map_final.

Varianta 3: Procesare Distribuită MPI (Memorie distribuită)
Funcție: ruleazaMPI
Mecanism: Codul rulează ca procese complet izolate în sistemul de operare.
1.Fiecare proces MPI (rank) calculează pe ce porțiune de text trebuie să opereze pe baza size-ului total.
2.Fiecare proces își creează propriul map_local.
Protocolul de Comunicare (Serializare/Deserializare):
Deoarece procesele MPI nu împart aceeași memorie, dicționarele nu pot fi trimise direct ca obiecte C++.
serializeazaBinar: Transformă unordered_map într-un șir compact de bytes (număr de elemente, dimensiune cuvânt, caracterele efective, frecvența).
Procesele worker (rank != 0) trimit prima dată dimensiunea buffer-ului (MPI_Send), apoi datele efective.
Procesul master (rank == 0) folosește un buclă secvențială pentru a primi datele de la toți worker-ii (MPI_Recv), apoi apelează deserializeazaBinar pentru a aduna datele brute în map_final.



1.Eficiența maximă a variantelor cu Threads: Varianta paralelizată cu thread-uri este de aproximativ 3 ori mai rapidă decât cea secvențială. Acest lucru demonstrează scalabilitatea bună a problemei atunci când se folosește memoria partajată. Thread-urile pot scrie și citi din aceeași zonă RAM fără costuri de comunicare inter-proces.
2.Paradoxul MPI pe sisteme Single-Node: Deși este o metodă de paralelizare, varianta MPI a rulat mai lent decât varianta secvențială pe o singură mașină. Acest comportament este așteptat din cauza:
Costului uriaș de Serializare/Deserializare: Convertirea a aproape 500.000 de intrări de dicționar în șiruri de biți consumă enorm de multe cicluri CPU.
Bottleneck la Procesul 0: Procesul principal rank 0 trebuie să preia și să dezarhiveze secvențial structurile de la toți ceilalți workeri.
oOverhead-ul de rețea: Mutarea pachetelor de date între procese separate induce latențe masive comparativ cu accesarea directă a pointerilor în RAM.
5. Mențiuni de Utilizare
Sistemul rulează o buclă infinită controlată de procesul 0, care primește un input (meniu de la 0 la 3). Opțiunea este transmisă tuturor proceselor prin MPI_Bcast.
Aplicația utilizează o barieră de sincronizare MPI_Barrier(MPI_COMM_WORLD) la sfârșitul fiecărui ciclu de meniu pentru a asigura ca niciun proces nu începe o execuție nouă până când afișarea opțiunii curente nu a fost finalizată.
