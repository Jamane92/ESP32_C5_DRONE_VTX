Le projet de base fonctionne bien avec un téléphone, mais l'utilisation d'un VRX adapté peut augmenter énormément les performances du système. C'est pourquoi j'ai fait le choix d'utiliser un module de récéption wifi avec des antennes, que je vais connecter à mon ordinateur, pour augmenter non seulement la portée mais également la latence. Je vais utiliser une clé WIFI Kali Linux avec puce RTL8812AU, double bande 1200 Mbps, avec un adaptateur USB sans fil USB 3.0, avec double antenne 5 dBi. 

Plus en détail, 

**Amélioration de la Portée (Sensibilité et Signal)**

  	**Monitor (Écoute passive)** : En Wi-Fi standard (mode Access Point ou Station), si le signal devient trop faible, le protocole abandonne et la connexion réseau se coupe brutalement. Sous Linux, la clé Wi-Fi est basculée en mode monitor : elle écoute aveuglément la fréquence 5.8 GHz sans chercher à s'associer au routeur     de l'ESP32. Tant que des ondes arrivent, elle les capte, prolongeant la portée utile de plusieurs centaines de mètres.

  	**Dégradation fluide (Graceful Degradation)** : Au lieu d'avoir une image vidéo qui fige (freeze) d'un seul coup en limite de portée, le récepteur Linux accepte d'afficher des paquets incomplets ou corrompus. L'image va se pixeliser ou se déformer par blocs (un peu comme la neige en analogique), ce qui te laisse le temps 				de     réagir et de faire demi-tour avant le failsafe.

  **Connectique pour antennes à haut gain** : Une clé Wi-Fi USB compatible (ex: chipsets Atheros ou Realtek) permet de visser des antennes directionnelles (Patch ou Hélicoïdale) qui, combinées à l'antenne Lollipop RHCP de ton drone, concentrent la réception sur une zone précise pour aller chercher le signal beaucoup plus loin.

**Optimisation des Performances (Latence et Fluidité)**

  **Suppression des Acquittements (No ACK / Retransmission)** : Le Wi-Fi TCP/UDP standard exige que le récepteur confirme la bonne réception de chaque paquet. S'il y a des interférences (les moteurs, les ESC), le paquet est retransmis, créant des pics de latence énormes. Avec un VRX Linux, le flux est envoyé en Broadcast      (diffusion pure). Si une trame vidéo est perdue, elle est simplement ignorée pour afficher la suivante immédiatement. La latence reste ainsi constante et ultra-faible, ce qui est vital pour le pilotage.

  **Correction d'Erreur (FEC - Forward Error Correction)** : Pour compenser l'absence de retransmission, les systèmes VRX intègrent des algorithmes de redondance. Le système reconstruit mathématiquement les bouts d'images manquants à la volée grâce aux données de parité, lissant le flux vidéo de ton OV5640 même lorsque la         liaison radio est parasitée.
