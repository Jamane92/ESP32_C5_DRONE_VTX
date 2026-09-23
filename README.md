# ESP32_C5_DRONE_VTX

Ce projet à été réalisé individuellement en parallèle de mes études. C'est un module d'émission vidéo en digital à bas coût. Souhaitant me challenger dans mon parcours d'apprentissage en conception hardware, je me suis lancé le défi de construire un drone moi même, avec en première étape de ce gros projet, de faire un VTX ainsi qu'un Flight Controler (FC) de A à Z. Le projet Flight Controler est disponible sur cette page : https://github.com/Jamane92/Flight_Controler_STM32F405RGT6. 

Qu'est ce qu'un module VTX ? C'est un module d'émission vidéo, qui permet de transmettre la vue à la première personne du drone sur un écran au sol, que ce soit un téléphone ou un ordinateur par exemple. Cela permet de repérer le drone dans l'espace et de le faire voler à des endroits où nous ne pouvons pas voir. Un set complet bas/milieu de gamme coùute aux alentours de 200 à 250€. Ca fait mal. J'ai donc décidé, pour m'apprendre à implémenter une chip ESP32 sur PCB, de faire un PCB custom qui intègre une ESP32-C5-WROOM-1U-N8R8, une OV5640 ainsi qu'une carte SD, pour la somme ≈ 40€. Ce qui est un prix imbattable, tout simplement.

Le but de se projet est simple : d'un point de vue technique, m'apprendre à router des SoC (System On Chip) ainsi que des signaux critiques comme ceux de la caméra OV5640, le tout sur mon premier PCB 4 couches avec des composants SMD. Un grand pas donc depuis mon premier PCB sur mon robot équilibriste (https://github.com/Jamane92/Self_Balancing_Robot). Plus de détail dans les rubriques sur les choix de design (notamment sur des améliorations possibles).

Voilà à quoi ressemble le module : 
<p align="center">
<img width="1309" height="378" alt="vtx" src="https://github.com/user-attachments/assets/fcd6ed43-ef09-44da-ad7d-6fa0e562a62f" />
</p>

Plus en détail, le coeur au milieu, à gauche l'antenne Lollipop et à droite la caméra. Ce PCB à fonctionné du premier coup. Voici un petit exemple de ce dont il est capable (entre autres : réglage de a qualité vidéo, compression, luminosité, saturation, contraste, filtre, enregistrement sur la SD Card) : 
<p align="center">
<img width="1000" src="../media/rec000.mp4" alt="Schématique" />
</p>

Pour la démo en vol, il faudra attendre que le projet du flight controler (https://github.com/Jamane92/Flight_Controler_STM32F405RGT6) soit terminé. 

Pour voir les performances du montage, voir la section dédié.
