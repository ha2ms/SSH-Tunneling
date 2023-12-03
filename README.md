# Sécuriser ses remontées de metrics vers Prometheus via le SSH-Tunneling

Lors d'une remontée standard de metrics, l'on configure Prometheus pour pointer sur l'hôte et le port prédéfini de notre machine cible, contentant le node-exporter nous permettant l'accès à ses précieuses metrics.
## 
Cependant ces metrics sont ouvertement accessibles et exploitable par n'importe qui faisant une requête sur le service.
  L'idée est donc de fermer l'accès à ces metrics depuis l’extérieur de la machine, mais comment Prometheus va pouvoir y accéder si les données ne sont accessibles qu'au sein même de la machine ? Un VPN ?
  ##
Plus simple, un Tunnel SSH, pas de certificats à gérer,  pas de configuration complexe, une simple clé privée et publique suffit pour établir nôtre connexion sécurisée pour accéder à l'hôte distant, récupérer les fameuses metrics, et les rendre de nouveaux accessibles au sein même de notre machine Prometheus.

> Le SSH Tunneling va:
> - Ouvrir un service sur un port local de Prometheus pour qu'il puisse s'y connecter (ex: 7080).
> - Dès que Prometheus lancera une requête sur son port 7080, une connexion SSH sera établit avec l'hôte distant contenant les metrics. 
> - Elles seront récupérées depuis l'hôte distant lui même puis redirigées (via le tunnel sécurisé) sur le serveur local de Prometheus (au port 7080).
> 
>![](http://93.90.205.194/docs/ssh-tunneling/ssh-tunneling-draw-number.png)
> ##

## Sur le Serveur Prometheus:
> ## Préparation de notre environnement:

> ### Création d'une nouvelle paire de clés:
> ```bash
> # Création des répertoires nécessaire pour y stocker notre paire de clé 
> mkdir -p /etc/systemd/service_ssh_key/ssh-tunnel
>
> # Création de notre paire de clés
> sudo ssh-keygen -t rsa -b 2048 -f /etc/systemd/service_ssh_key/ssh-> tunnel/id_rsa
>```
> ##


> C'est l'option **-L** de la commande **SSH** qui permet de réaliser ce "forward" en créant un service en local sur un port défini qui sera le "miroir" du service distant:
> ```bash
> ssh -L my_localhost:my_localhost_port:remote_localhost:remote_localhost_port remote_user@remote_host
>
> #ex:
> ssh -L 127.0.0.1:9100:127.0.0.1:9100 remote_user@192.168.1.127
> # Fait un "pont" de communicaton entre le localhost:9100
> # de la machine actuelle et le localhost:9100
> # de la machine distante (qui se situe à 192.168.1.127).
> ```
> La machine sur lequel s’exécute cette commande doit avoir le port sur lequel elle écoute de disponible et non utilisé par un autre processus.
> ##

 > ### Édition du nouveau service:
 > ```bash
 > # Création d'un nouveau service
 > sudo systemctl edit --force --full ssh-tunnel.service
 > ```
> ##
> Ici on configure dans la section [Service] une variable d'environnement contenant le chemin de notre clé privé que l'on réutilisera juste en dessous dans ExecStart:

> ```vim
> [Unit]
> Description=Persistent SSH Tunnel from port localhost:7080 to port 9100 on bigbluebutton.cloudns.eu
> After=network.target
> 
> [Service]
> Restart=on-failure
> RestartSec=5
> Environment="SSH_KEY=/etc/systemd/service_ssh_key/ssh-tunnel/id_rsa"
> ExecStart=/usr/bin/ssh -i $SSH_KEY -NTC -o ServerAliveInterval=60 -o ExitOnForwardFailure=yes -L 127.0.0.1:7080:127.0.0.1:9100 mehdi@bigbluebutton.cloudns.eu
>
> [Install]
> WantedBy=multi-user.target
> ```
> ##

> On active le nouveau service:
> ```bash
> sudo systemctl daemon-reload
> sudo systemctl enable ssh-tunnel.service
> sudo systemctl start ssh-tunnel.service
> 
> # On s'assure que le service tourne correctement
> sudo systemctl status ssh-tunnel.service
> ```
> ##

> On modifie la destination de la target dans notre config **prometheus.yml**:
> ```yaml
> scrape_configs:
>	- job_name: 'node'
>		scrape_interval: 5s
>		scrape_timeout: 5s
>		metrics_path: /metrics
>		scheme: http
>		static_configs:
>			- targets: ['localhost:7080']
> ```
> ##

> #### On restart le service:
> ```bash
> sudo systemctl restart prometheus.service
> ```
> On vérifie enfin que la remonté s'effectue correctement depuis l'interface web de Prometheus:
> ![](http://93.90.205.194/docs/ssh-tunneling/prometheus-7080.png)
> Notre tunnel fonctionne !
> ##

Notre Prometheus passe bien par le tunnel SSH, mais nos metrics restent toujours accessibles depuis l’extérieur, il nous faudra bloquer cet accès pour n'écouter sur le port 9100 seulement depuis la machine elle-même.
 
## Sur la Machine du node-exporter:
> Récupérez et stockez dans le **authorized_keys** (de l'utilisateur utilisé par le **tunnel SSH**) la clé publique généré sur le serveur Prometheus.
> ##

> #### Éditer le service node-exporter:
> ```bash
> sudo systemctl edit --full prometheus-node-exporter.service
> ```
>
>```vim
>[Unit]
> Description=Prometheus exporter for machine metrics
> Documentation=https://github.com/prometheus/node_exporter
>
> [Service]
> Restart=always
> User=prometheus
> EnvironmentFile=/etc/default/prometheus-node-exporter
> ExecStart=/usr/bin/prometheus-node-exporter $ARGS
> ExecReload=/bin/kill -HUP $MAINPID
> TimeoutStopSec=20s
> SendSIGKILL=no
>
> [Install]
> WantedBy=multi-user.target
>```
>##
>## Si vous avez utiliser < apt install > (gestionnaire de paquets) pour le node-exporter
>On peut voir qu'un fichier d'environnement existe dans **</etc/default/prometheus-node-exporter>**, editons le:
>```bash
>sudo nano /etc/default/prometheus-node-exporter
>```
>L'on peut voir vers le début l'argument **ARGS=""**
>Ajoutez entre les guillemets: 
>```vim
>ARGS="--web.listen-address=127.0.0.1:9100"
>```
>Puis dans l'édition du systèmed ajouter **$ARGS** comme argument dans **ExecStart**
>```vim
>ExecStart=/usr/bin/prometheus-node-exporter $ARGS
>```
>##
> ## Si vous avez installé node-exporter manuellement
> Dans l'édition du systemd du node-exporter.
> Ajoutez la variable d'environnement **ARGS** juste au-dessus de **ExecStart**:
> ```vim
> # Création variable environnement
> Environment="ARGS=--web.listen-address=127.0.0.1:9100"
> ExecStart=/usr/bin/node-exporter $ARGS
> # Utilisation de la variable dans ExecStart
> ```
> Puis relançons le démon du systemd:
> ```bash
> sudo systemctl daemon-reload
> sudo systemctl restart node-exporter
> sudo systemctl status node-exporter
> ```
> Maintenant les metrics ne sont accesibles que depuis la machine elle-même ! Et par conséquent seul vôtre tunnel est à même de récupérer les donnés depuis l'extérieur.
> ##
