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
