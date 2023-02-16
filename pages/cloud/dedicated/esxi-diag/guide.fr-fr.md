---
title: 'Maîtriser et sécuriser votre serveur dédié ESXi dès son 1er démarrage'
slug: esxi-diag
excerpt: 'Découvrez ou redécouvrez les différents moyens disponibles vous permettant de sécuriser efficacement votre serveur dédié ESXi'
section: 'Utilisation avancée'
---


## Objectif

Cette documentation aura pour but de vous accompagner pour élaborer au mieux la sécurisation pour votre système ESXi.  
Nous verrons les fonctions embarquées que propose vmware, mais aussi celles proposées par OVHcloud.


> [!warning]
> 
> Récemment, les systèmes ESXi ont été victime d'une faille que de(s) groupe(s) malveillant(s) ont exploités très rapidement à travers les réseaux publiques.  
> Nous vous proposons des moyens rapides et, à moyens/longs termes également, avec cette FAQ complémentaire, disponible [ici](https://docs.ovh.com/fr/dedicated/esxi-faq/).
>


### Rappel des bonnes pratiques de sécurité :

* Mettez à jour régulièrement vos systèmes ESXi.
* Restreignez l’accès aux seules adresses IP de confiance.
* Désactivez les ports ainsi que les services inutilisés.
* Assurez-vous que les accès à vos serveurs ou vos équipements réseaux sont limités, contrôlés et protégés avec des mots de passe robustes.
* Sauvegardez vos données critiques dans des disques externes et des serveurs de backup protégés et isolés d’Internet.

Optionnel:

* Mettez en place des solutions de journalisation nécessaires pour contrôler les évènements survenus sur vos serveurs critiques et vos équipements réseaux.
* Mettez en place des packs de sécurité pour la détection des actions malveillantes, des intrusions (IPS / NIDS) et pour le contrôle de la bande passante de trafic réseau.


## Prérequis

* Être connecté à l'[espace client OVHcloud](https://www.ovh.com/auth/?action=gotomanager&from=https://www.ovh.com/fr/&ovhSubsidiary=fr){.external}.
* Possédez un serveur dédié avec la solution ESXi déployée.
* Avoir souscrit à une offre compatible avec notre fonctionnalité [Network_Firewall](https://docs.ovh.com/fr/dedicated/firewall-network/)


## En pratique

### fail2ban natif

rappel sur sa définition
> [!info]
> Le système ESXi embarque un mécanisme de sécurité lié au compte administrateur.  
> En effet, en cas de plusieurs tentatives érronées les accès liés au compte administrateur seront vérrouilliés temporairement.  
> Ceci afin de protégéger votre système et ainsi d'éviter les tentatives de connexions infructueuses, qui auront comme impact de surcharger votre machine le cas échéant.  

> [!warning]
> Dès lors, si vous souhaitez réinitialiser le décompte du vérrouillage , il vous sera nécessaire de redémarrer votre solution ESXi.  
> 

Il est possible de consulter l'historique des logs d'accès via les fichiers suivants en shell:  

`/var/run/log/vobd.log` logs monitoring
```bash
2023-02-13T16:22:22.897Z: [UserLevelCorrelator] 410535559us: [vob.user.account.locked] Remote access for ESXi local user account 'root' has been locked for 900 seconds after 6 failed login attempts.
2023-02-13T16:22:22.897Z: [GenericCorrelator] 410535559us: [vob.user.account.locked] Remote access for ESXi local user account 'root' has been locked for 900 seconds after 6 failed login attempts.
2023-02-13T16:22:22.897Z: [UserLevelCorrelator] 410535867us: [esx.audit.account.locked] Remote access for ESXi local user account 'root' has been locked for 900 seconds after 6 failed login attempts.
```

`/var/run/log/hostd.log` logs de l'hôte (tâches, accès interface web,etc...):
```bash
2023-02-16T13:35:48.829Z info hostd[2099631] [Originator@6876 sub=Vimsvc.ha-eventmgr opID=esxui-e70a-159a user=root] Event 147 : User root@xxx.xxx.xxx.xxx logged out (login time: Thursday, 16 February, 2023 01:26:42 PM, number of API invocations: 12, user agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36)
2023-02-16T13:36:09.152Z info hostd[2099625] [Originator@6876 sub=Default opID=esxui-eabe-159d] Accepted password for user root from xxx.xxx.xxx.xxx
```

Toutes ces informations sont également disponibles à travers l'interface d'administration web:  
menu `Host` > `Monitor`  

![interface](images/gui_logs_.png)

### Solution Network Firewall

Nous vous proposons d'activer et d'utiliser notre solution de filtrage [Network Firewall](https://docs.ovh.com/fr/dedicated/firewall-network/).  
Cette solution vous permettra de gérer facilement les accès légitimes en complément de ceux que vous aurez mis en place à travers votre système ESXi.  

En effet, cette solution anti-DDOS vous évitera le lock inopiné de votre compte administrateur.  

Il est donc recommandé de filtrer les accès légitimes de cette manière:  
La régle 1 (Priority 0) : autorise les accès externes qui auront besoin d'accèder à votre système ESXi.  
La régle 2 (Priority 1) : bloque tout le reste.  

![Network_Firewall](images/firewall_network_.png)


### Filtrage sous ESXi

> [!primary]
>
> Vous avez également la possibilité de filtrer les accès des utilisateurs de votre système ESXi.  
> Aussi vous pourrez désactiver les services inutiles.  
>

> [!warning]
> La désactivation des services **ssh** et **slp** est fortement conseillée.  
> Ceci est valable également pour les accès en **shell**.  
> Ne prévilégiez que le strict nécessaire pour chacun de vos besoins.  

#### Manipulation via l'interface graphique

*services*

menu `Host` > `Manage` > `services`  
modifiez la `Policy` comme sur l'exemple présenté et choisissez l'option `Start an stop manually` pour une désactivation maîtrisée.  

![services_ssh](images/ssh_disabled_.png)
![services_slp](images/slpd_.png)



*règles de pare-feu*

menu `Networking` > `Firewall rules`  
choisissez `Edit setting`:  

![rules](images/firewall_web_.png)


éditez la règle pour n'ajouter que la ou les adresses IP, ou encore réseau(x), à pouvoir se connecter à votre système ESXi.  

![custom](images/custom_fw_rule.png)


#### Manipulation via le shell

Statut sur l'état du filtrage ou ACL actuel :
```bash
esxcli network firewall ruleset list
esxcli system account list
```

désactivez les services inutiles:

service SLP
```bash
/etc/init.d/slpd stop
esxcli network firewall ruleset set -r CIMSLP -e 0
chkconfig slpd off
```

service SSH
```bash
/etc/init.d/SSH stop
esxcli network firewall ruleset set -r sshServer -e 0
```



## Aller plus loin
Échangez avec notre communauté d'utilisateurs sur <https://community.ovh.com/>.
