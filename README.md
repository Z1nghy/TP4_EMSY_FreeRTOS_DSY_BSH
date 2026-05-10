# TP4_EMSY_FreeRTOS_DSY_BSH

voici les configurations à mettre dans putty :
le serial com : a vous de voir le votre
la vitesse de lecture : 
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











