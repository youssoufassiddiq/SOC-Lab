# TP pratique : déployer Wazuh et détecter les échecs SSH dans VirtualBox

**Lab isolé :** Kali Linux (192.168.100.10), Ubuntu Server avec agent (192.168.100.20), Ubuntu Desktop avec Wazuh (192.168.100.30), réseau interne 192.168.100.0/24. Vérifiez que les trois VM utilisent le même réseau interne VirtualBox. Pour télécharger les paquets, ajoutez temporairement une interface NAT; conservez le réseau interne pour les communications du TP et n’exposez aucun service à Internet.

Ce TP installe les trois composants centraux Wazuh sur une seule VM, enrôle l’agent Ubuntu, puis vérifie la remontée des échecs SSH générés depuis Kali. Réalisez les essais uniquement sur vos machines de laboratoire.

## 1. Vérifier les VM

Sur chaque Ubuntu et Kali :

```bash
ip -br address
ip route
```

Tester depuis Kali :

```bash
ping -c 3 192.168.100.20
ping -c 3 192.168.100.30
```

Autoriser dans les pare-feu du lab les flux nécessaires : TCP 1514 (agent vers manager), TCP 1515 (enrôlement vers manager), TCP 443 (navigateur vers dashboard) et TCP 22 vers Ubuntu Server pour le test SSH. Limiter ces règles au réseau du TP.

## 2. Installer Wazuh tout-en-un

Sur Ubuntu Desktop, mettre le système à jour :

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y curl tar
sudo reboot
```

Après redémarrage, vérifier l’adresse 192.168.100.30. Installer ensuite les composants centraux avec l’assistant tout-en-un officiel :

```bash
cd /root
sudo curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

L’assistant affiche le mot de passe du dashboard. Le retrouver si besoin dans l’archive générée :

```bash
sudo tar -O -xvf /root/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Protéger cette archive et ses identifiants. Vérifier les services :

```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
```

Depuis une VM du lab, ouvrir https://192.168.100.30 puis se connecter en admin. Un avertissement de certificat peut apparaître avec le certificat autosigné du lab.

> **À propos de la commande du support :** `wazuh-install.sh --wazuh-server wazuh-1` installe le manager/Filebeat, mais ne suffit pas à installer l’indexer et le dashboard. Pour un serveur unique, la commande `-a` ci-dessus déploie les trois composants. La procédure composant par composant doit suivre toutes les étapes officielles (configuration, certificats, indexer, manager, dashboard).

## 3. Installer et enrôler l’agent Ubuntu

Sur Ubuntu Server 192.168.100.20, ajouter le dépôt officiel :

```bash
sudo apt update
sudo apt install -y curl gnupg apt-transport-https
sudo install -d -m 0755 /usr/share/keyrings
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor --yes -o /usr/share/keyrings/wazuh.gpg
sudo chmod 0644 /usr/share/keyrings/wazuh.gpg
echo 'deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main' | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

Installer l’agent avec l’adresse du manager; l’enrôlement automatique est recommandé :

```bash
sudo WAZUH_MANAGER='192.168.100.30' apt-get install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent --no-pager
```

Dans le dashboard, aller à **Agents management → Summary** et attendre l’état **Active**. Depuis le manager :

```bash
sudo /var/ossec/bin/agent_control -l
```

Le support décrit aussi l’import manuel d’une clé avec `manage_agents`; cette méthode reste disponible, mais l’enrôlement automatique évite le transfert manuel de la clé. Ne partagez et ne réutilisez jamais une clé d’agent.

## 4. Préparer les journaux SSH

Sur Ubuntu Server :

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

Vérifier que le journal existe et que l’agent le collecte :

```bash
sudo test -f /var/log/auth.log && sudo tail -n 20 /var/log/auth.log
sudo grep -n -A3 -B1 '/var/log/auth.log' /var/ossec/etc/ossec.conf
```

Si aucun bloc ne référence ce fichier, ajouter avant `</ossec_config>` dans `/var/ossec/etc/ossec.conf` :

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

Ne pas créer un doublon. Après modification :

```bash
sudo systemctl restart wazuh-agent
```

## 5. Générer et examiner les événements

Depuis Kali 192.168.100.10, vérifier SSH puis effectuer quelques essais avec un utilisateur de test et un mot de passe incorrect :

```bash
nc -zv 192.168.100.20 22
ssh utilisateur-test@192.168.100.20
```

Arrêter après quelques échecs. Sur Ubuntu Server, confirmer leur présence :

```bash
sudo grep -i 'Failed password\|invalid user' /var/log/auth.log | tail -n 20
```

Dans Wazuh, ouvrir **Threat hunting → Events**, sélectionner la période du test et filtrer sur l’agent ou le texte `Failed password`. Vérifier l’heure, l’utilisateur, l’adresse source 192.168.100.10 et la règle déclenchée. Un échec isolé peut être collecté sans constituer une alerte agrégée de brute force.

Logs utiles :

```bash
# Sur l'agent et sur le manager
sudo tail -n 80 /var/ossec/logs/ossec.log
# Alertes analysées sur le manager
sudo tail -n 30 /var/ossec/logs/alerts/alerts.json
```

La chaîne validée est : journal SSH → agent → manager/analyse → indexation → dashboard. Le test démontre la collecte/détection d’échecs SSH. Il ne prouve pas que Wazuh détecte un scan Nmap par défaut : il faut une source réseau (IDS/NIDS ou logs de pare-feu) et l’intégrer à Wazuh.

## 6. Dépannage rapide

**Agent inactif :** sur l’agent, tester les ports, puis consulter les logs :

```bash
nc -zv 192.168.100.30 1514
nc -zv 192.168.100.30 1515
sudo systemctl status wazuh-agent --no-pager
sudo tail -n 80 /var/ossec/logs/ossec.log
```

Vérifier l’adresse du manager, le réseau interne et les pare-feu. Les noms d’agents doivent être uniques.

**Pas d’événement SSH :** confirmer que `/var/log/auth.log` reçoit l’échec, que le bloc `<localfile>` est présent une seule fois, puis redémarrer l’agent.

**Dashboard inaccessible :** sur Ubuntu Desktop, vérifier `wazuh-dashboard`, `wazuh-indexer`, TCP 443 et l’URL HTTPS.

## 7. Recette

- [ ] Les trois VM communiquent sur le réseau interne.
- [ ] Manager, indexer et dashboard sont actifs.
- [ ] Le dashboard répond sur https://192.168.100.30.
- [ ] L’agent Ubuntu Server est **Active**.
- [ ] Les échecs SSH figurent dans auth.log et dans les événements Wazuh.
- [ ] La source Kali 192.168.100.10 est visible.
- [ ] Le rapport distingue détection SSH et détection Nmap.

### Documentation officielle

- [Quickstart](https://documentation.wazuh.com/current/quickstart.html)
- [Architecture et ports](https://documentation.wazuh.com/current/getting-started/architecture.html)
- [Installation assistée du serveur Wazuh](https://documentation.wazuh.com/current/installation-guide/wazuh-server/installation-assistant.html)
- [Installation d’un agent Linux](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html)
- [Dépannage de l’enrôlement](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/troubleshooting.html)
