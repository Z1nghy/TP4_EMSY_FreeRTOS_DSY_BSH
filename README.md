# TP4_EMSY_FreeRTOS_DSY_BSH
## Partie 1 sans OS

voici les configurations à mettre dans putty :
le serial com : a vous de voir le votre
la vitesse de lecture : 9600 baud
Data nits : 
Stop bits : 
Parité : None
Flow control : XON/XOFF

et voici donc ce que nous avons mesuré sur les trois LED's : 

voici ce qui ce passe lorsque nous spammons la lettre H
<img width="1280" height="824" alt="image (1)" src="https://github.com/user-attachments/assets/2b442b5c-8ac4-4cfb-9de4-272bd3c739d4" />
Les signaux correspondent aux activités suivantes :

LED0 (C1 - Jaune) : Interruption UART (réception de caractères).
LED1 (C2 - Vert) : Interruption Timer (mesure de température).
LED2 (C3 - Orange) : Tâche de fond (lecture et affichage).

Sur l’Image, on observe clairement que la tâche de fond (C3) s'arrête de traiter les données alors que les interruptions (C1 et C2) continuent de s'exécuter


voici ce que nous avons mesuré lors de coiper-coller de beaucou de H :
<img width="1280" height="824" alt="image" src="https://github.com/user-attachments/assets/93a71abd-cf26-4e9b-bb2d-9ab40f74ef38" />

Le problème est systématique lors d'un "copier-coller" massif. Cela indique une condition de course ou une corruption de ressources partagées liée à la charge

Mise en évidence et cause :

Le problème majeur est la non-réentrance de la fonction d'écriture dans le FIFO ou l'absence de section critique.

- Concurrence : Une interruption prioritaire, comme celle du Timer, peut couper la parole à une interruption moins prioritaire, par exemple celle de l'UART, pendant que cette dernière est en train de mettre à jour les pointeurs du FIFO (head, tail ou count).

- Corruption : Si le Timer modifie l'index d'écriture alors que l'UART n'a pas encore terminé sa mise à jour, les pointeurs deviennent incohérents.

- Résultat : La tâche de fond (C3) se retrouve alors face à un FIFO dont les compteurs sont erronés — elle peut croire, par exemple, que le FIFO est vide alors qu'il est plein, ou l'inverse. Du coup, elle se bloque ou ignore les données, ce qui explique pourquoi le signal orange reste à l'état bas vers la fin de la capture.

En termes de solutions, je propose soit de masquer sélectivement les interruptions en montant temporairement le niveau de priorité des IRQ susceptibles d'accéder au FIFO, soit de séparer les FIFOs par source en attribuant un FIFO distinct à chaque interruption, éliminant ainsi toute contention sans nécessiter de section critique.

## Partie 1 avec OS

Nous allons aussi utilliser PuTTY avec la configuration suivante:

 - Le Serial com :  Celle que vous avez sur le gestionaire de peripherique
 - la vitesse de lecture : 9600 baud
 - Data bits : 8
 - Stop bits : 1
 - Parity : None
 - Flow Control : XON/XOFF

Voici la configuration de mesure:
  - Chanel 1 sur l'interruption du timer (LED ...)
  - Chanel 2 sur l'interruption de l'UART (LED ...)
  - Chanel 3 sur l'App LCD (LED ...)
  - Chanel 4 sur l'app temp (LED ...)
  - Chanel 1 et 4 sont à 5V/div
  - CHanel 2 et 3 sont é 2V/div
  - Trigger sir le Chanel 2
  - Base de temps à 20ms
  - 
### Saisie de un charactère 
Voci donc notre mesure lorsque un seule charactère est envoyé à la fois:
<img width="1280" height="824" alt="image (2)" src="https://github.com/user-attachments/assets/93b2a2b1-2bde-44ad-9e08-4c8f16a59256" />

Nous pouvons voir clairement lorsqu'il y a un charactère qui est envoyé par le temps bas du chanel 2.

Chanel 1 :
  Ce signal fait des impulsions courtes périodiques toutes les 100 ms. Dans son ISR, il remet le sémaphore binaire avec la    fonction "xSemaphoreGiveFromISR" afin de débloquer et réveiller la tâche d'acquisition de température.
  
Chanel 2 :
  Ce signal passe à l’état bas au moment précis où un caractère ASCII arrive sur le port série depuis PuTTY. L’interruption   UART capte directement ce caractère, l’encapsule dans le format de message prévu (« MSG_TYPE_CHAR » ) et l’envoie dans la   file partagée grâce à la fonction « xQueueSendFromISR »
  
Chanel 3 :
  On voit que ce signal est beaucoup appelé, c'est pour cela que nous voyons ce signal "toggelé", ce signal est celui de   l'application LCD qui va afficher cur le LDC la temperture lues ainsi que le charactère recu.

Chanel 4 : 
  Ce signal passe à l'état haut pendant une durée fixe d'environ 40 ms. Nous pouvons voir qu'il est un peu en retard par      rapport au Chanel 1 ce qui montre bien que le chanel 1 lui sort de son etat bloqueé, ce qui fais prendre la temperature     le SPI et l'envoi pour l'afficher sur le LCD.
  
### Saisie de plusieure charactère
Voci donc notre mesure lorsque plusieure charactère est envoyé à la fois:

<img width="1280" height="824" alt="image (3)" src="https://github.com/user-attachments/assets/2702dd50-e9f7-4a1a-ba23-0cd901451775" />
 
Chanel 2 :
  On observe un gros bloc vert compact, qui descend à l'état bas pendant environ 35 ms. Cela montre que l'interruption UART   s'exécute de façon quasi continue à cause du débit d'octets arrivant en rafale. L'interruption traite et empile tous les   caractères dans la file d'attente à la suite les uns des autres.


  ## Comparaisons
  ### Envoi unique
Sans OS : 
 - L'interruption UART (C1 - Jaune) s'exécute immédiatement à la réception. Elle est très courte et ne fait qu'empiler le caractère dans le FIFO.
 - Une fois l'interruption terminée, la tâche de fond (C3 - Orange) détecte la donnée dans le FIFO , se réveille et prend en    charge l'affichage LCD. Cet affichage génère un créneau d'exécution unique et continu (C3 reste à l'état bas pendant         toute la durée du traitement).
 - L'interruption Timer (C2 - Vert) s'intercale de manière périodique (toutes les 100 ms) sans interférence logicielle          directe sur l'affichage, si ce n'est par la priorité stricte du matériel.  

Avec FreeRTOS : 
 - Les signaux sont entrelacés et hachés. Il n'y a plus de blocs d'exécution continus.
 - Dès que le Timer (C1 - Jaune) déclenche son interruption, il libère un sémaphore. Le planificateur de FreeRTOS donne         immédiatement la main à la tâche de lecture de température App Temp (C4 - Violet).
 - Le signal de la tâche App LCD (C3 - Orange) montre des commutations rapides : la tâche ne monopolise pas le processeur en    continu. Elle consomme des petits fragments de temps CPU pour traiter la file d'attente (xQueueReceive) et envoyer les       commandes à l'afficheur, laissant les autres tâches et interruptions s'exécuter au besoin

 ### Envoi Copié-Collé

Sans OS:
 - Canal C1 (Jaune - Interruption UART) : Le signal passe son temps à commuter à haute fréquence. L'interruption matérielle     UART s'exécute en continu pour essayer de stocker tous les caractères reçus.

 - Canal C2 (Vert - Interruption Timer) : On remarque une rafale ininterrompue de commutations très rapides (le signal          ressemble à un bloc haché compact). C'est le reflet de la lecture répétée du capteur SPI dans l'interruption.

 - Canal C3 (Orange - Tâche de fond) : Le signal reste totalement plat et figé à l'état haut. La tâche de fond (qui gère        l'affichage LCD) est complètement bloquée et asphyxiée. Comme les interruptions UART (Priorité 4) et Timer (Priorité 3)      s'enchaînent sans s'arrêter, le processeur n'a plus aucun cycle disponible pour revenir à la boucle principale.              L'affichage ne s'actualise plus du tout pendant la rafale.

Avec OS : 

 - Canal C1 (Jaune - Interruption Timer) : Il conserve une forme parfaitement propre et déterministe. L'interruption se contente de signaler un sémaphore, prenant un temps d'exécution minimal et constant.

 - Canal C2 (Vert - Interruption UART) : Contrairement à la version sans OS, les commutations de l'UART s'effectuent au sein    d'une zone délimitée et contrôlée. L'interruption UART décharge le caractère reçu directement dans la file d'attente         xQueueLcd.

 - Canal C3 (Orange - Tâche App LCD) : Le signal commute de manière extrêmement dense et continue (créant une bande orange).    Cela prouve que la tâche d'affichage continue de s'exécuter activement au milieu de la crise. Dès qu'un caractère arrive     dans la queue, le Scheduler (planificateur) redonne la main à l'application LCD pour traiter et afficher la donnée.

 - Canal C4 (Violet - Tâche App Temp) : La tâche de température maintient son cycle d'activation régulier. Elle n'est pas       perturbée par l'afflux massif de caractères sur l'UART car l'OS distribue équitablement le temps CPU



