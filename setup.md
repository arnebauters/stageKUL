# Sensu

Het eerste deel van de documentatie staat Sensu en de configuratie uitgelegd aan de hand van de lokale setup.

In het tweede deel staat de configuratie voor de Sensu setup in de test omgeving.

## Sensu setup op VM met Puppet en Vagrant lokaal

### Vagrant

> - 3 VM's met centos7
> - 1 VM als backend en agent, grafana en InfluxDB
> - 2 VM's met enkel agent

De huidige setup bevat nu 3 backend vm's ipv 1, die in een cluster zijn geconfigureerd.

vagrant is een tool voor het bouwen en onderhouden van virtuele machines op een geautomatiseerde manier.

#### VagrantFile

Met behulp van de VagrantFile beschrijf je type machine en hoe de machine geconfigureerd moet worden. In de VagrantFile configureer ik 2 VM's met beide centos7, de ene vm als backend en agent voor sensu, de andere enkel als sensu-agent. Voorlopig zijn Grafana en InfluxDB handmatig geinstalleerd en geconfigureerd op de sensu-backend.

Ik geef beide machine's een privaat IP-address in range: `192.168.52.0/24`. Bij het opstarten van de machines laat ik ze een script draaien dat op beide VM's puppet installeert waardoor ik lokaal op de machines de puppetcode kan uitvoeren. Dit doe ik door het commando: `puppet apply`

```Ruby
Vagrant.configure("2") do |config|
   config.vm.define "server" do |server|
     server.vm.box = "centos/7"
     server.vm.hostname = "backend.example.com"
     server.vm.network "private_network", ip: "192.168.52.10"
     server.vm.network "forwarded_port", guest: 2380, host: 2380, auto_correct: true
     server.vm.network "forwarded_port", guest: 3000, host: 3000, auto_correct: true
     server.vm.network "forwarded_port", guest: 8080, host: 8080, auto_correct: true
     server.vm.network "forwarded_port", guest: 8081, host: 8081, auto_correct: true
   end

   config.vm.define "agent" do |agent|
     agent.vm.box = "centos/7"
     agent.vm.hostname = "agent.example.com"
     agent.vm.network "private_network", ip: "192.168.52.11"
   end

   config.vm.provision "shell", path: "update-puppet.sh"
   config.vm.provision "shell", inline: "puppet apply /vagrant/manifests/site.pp --modulepath=/vagrant/modules --hiera_config=/vagrant/hiera.yaml"
end

```

### Puppet

Puppet is een tool om software en operating systems te automatiseren. Ik gebruik het dus om sensu te automatiseren op elke machine.

#### Puppet structuur

```tree
.
├── hieradata
│   ├── common.yaml
│   │  
│   └── nodes
│        ├── agent.example.com.yaml
│        │  
│        └── backend.example.com.yaml
├── hiera.yaml
│
├── Makefile
│
├── modules
│
├── manifests
│    └── site.pp
├── modules
│    ├── datacat
│    │
│    ├── ntp
│    │
│    ├── sensu
│    │
│    └── stdlib
│
├──── Puppetfile
│  
├──── update-puppet.sh
│  
└──── VagrantFile


```

#### Puppet files

In Puppetfile steken alle modules die we gebruiken die komen uit de [puppetforge](https://forge.puppet.com/). Dit zijn modules die ik niet zelf heb geschreven maar beschikbaar zijn op deze publieke repository.

In `manifests/site.pp` laad je alle classes in die je nodig hebt. In de site.pp ga je in de hiera.yaml kijken welke nodes welke classes nodig hebben. De hiera.yaml gaat dan in de hieradata/nodes kijken welk classes agent.example.com en backend.example.com nodig hebben. Voorlopig gebruik ik enkel de sensu classes. De [sensu module](https://forge.puppet.com/sensu/sensu) komt uit de forge repository, deze klasse stata in modules/sensu.

`site.pp` waarin die in hiera.yaml gaat kijken:

> ```yaml
> #Puppet main manifest
> #
> # Include hiera classes
> #
> #
> node default {
> (lookup('classes', Array[String], 'unique') - lookup('exclude_classes',  Array[String], 'unique')).include
> }
> ```

 `hiera.yaml` die in hieradata naar de fqdn gaat zoeken

> ```yaml
> ---
> version: 5
> defaults:
> datadir: './hieradata/'
> data_hash: yaml_data
> hierarchy:
>  - name: "file backend"
> paths:
>  - nodes/%{facts.fqdn}.yaml
>  - common.yaml
>  - name: "common stuff"
>    glob: 'common/*.yaml'
> ```

`hieradata/nodes/backend.example.com` file waar de klasses worden aangeroepen voor de backend en bapaalde parameters worden ingesteld. Ook de cpu-check met bijhorende assets worden hier geconfigureerd.

>
> ```yaml
> classes:
>  - sensu::agent
>  - sensu::backend
>
> sensu::api_host: backend.example.com
> sensu::api_port: 8080
> sensu::username: admin
> sensu::password: ***
> sensu::old_password: ***
> sensu::use_ssl: false
> sensu::resources::bonsai_assets:
>    sensu-plugins/sensu-plugins-cpu-checks:
>      ensure: present
>      version: latest
>
>    sensu/sensu-ruby-runtime:
>      ensure: present
>      version: latest
>
>    sensu/sensu-slack-handler:
>      ensure: present
>      version: latest
>
> sensu::resources::filters:
>    hourly:
>      ensure: present
>      action: allow
>      expressions: event.check.occurrences == 1 || event.check.occurrences % (3600 / event.check.interval) == 0
>
> sensu::resources::checks:
>    check_cpu:
>      ensure: present
>      command: check-cpu.rb -w 75 -c 90
>      interval: 30
>      subscriptions:
>        - linux
>      publish: true
>      handlers: slack
>      runtime_assets:
>        - sensu-plugins/sensu-plugins-cpu-checks
>        - sensu/sensu-ruby-runtime
>
> sensu::resources::handlers:
>    slack:
>      ensure: present
>      type: pipe
>      env_vars:
>        - SLACK_WEBHOOK_URL=https://hooks.slack.com/services/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
>      command: sensu-slack-handler --channel '#sensu'
>      runtime_assets:
>        - sensu/sensu-slack-handler
>      filters:
>        - is_incident
>       - hourly
> ```

Voorlopig heb ik één check, nl. `check_cpu`. Deze gaat elke 30s de cpu gegevens van alle sensu-agents met subscription linux opvragen. De slack-handler gaat dan dit bericht doorsturen naar [Slack](https://slack.com/) enkel als er een warning of critical status is. Dit gebeurt maar 1 keer per uur als de status hetzelfde blijft dankzij de ingebouwde filter `is_incident` en de filter `hourly`.

Bericht van sensu in Slack via de Slack WebHook naar channel #sensu.

![Slack-screenshot](pictures/slack.png)

### grafana

Om dashboards met gegevens van sensu in grafana te creeren hebben we een database nodig om de gegevens van de checks in sensu op te slaan. Ik gebruik hiervoor [InfluxDB](https://influxdb.com). Om InfluxDB te installeren: [handleiding](https://docs.influxdata.com/influxdb/v1.7/introduction/installation/).
Eerst maak ik een handler die de sensu data gaat opslaan in de db.

```yaml
influx-db:
     ensure: present
     type: pipe
     env_vars:
       - INFLUXDB_ADDR=http:127.0.0.1:8086
       - INFLUXDB_USER=sensu
       - INFLUXDB_PASS=***
     runtime_assets: influxdb-handler
     command: sensu-influxdb-handler -d sensu
```

Daarna koppelen we de handler aan de metrics-cpu en metrics-disk check met de `metric_handlers` en slaan we de gegevens op als graphite_plaintext met de parameter `metric_format`.

```yaml
metrics-disk:
     ensure: present
     command: metrics-disk-usage.rb
     interval: 30
     subscriptions:
       - linux
     publish: true
     handlers: slack
     runtime_assets:
       - sensu-plugins/sensu-plugins-disk-checks
       - sensu/sensu-ruby-runtime
     metric_format: graphite_plaintext
     metric_handlers: influx-db
```

InfluxDB luistert standaard op poort localhost:8086. Om database en user te creeren:

```bash
influx -execute "CREATE DATABASE sensu;"
influx -execute "CREATE USER sensu WITH PASSWORD '***'"
influx -execute "GRANT ALL ON sensu TO sensu"
```

In Grafana om aan de interface te geraken heb ik niet localhost:3000 gebruikt maar op IP-address `192.168.52.10:4000`. 
Om InfluxDB als datasource in Grafana te gebruiken, maak je een file `influxdb.yaml` in `/etc/grafana/provisioning/datasources`

```yaml
apiVersion: 1

deleteDatasources:
- name: InfluxDB
  orgId: 1

datasources:
- name: InfluxDB
  type: influxdb
  access: proxy
  orgId: 1
  database: sensu
  user: grafana
  password: grafana
  url: http://localhost:8086
```

In Grafana zelf kan je nu verschillende dashboardjes make m.b.v. Query syntax. BV:
![Grafana Query Syntax](pictures/query_grafana.png)

2 dashboards met totale cpu van het systeem agent1 en disk% van agent1.

![Grafana Dashboard](pictures/grafana_dashboards.png)

### Sensu Cluster

Nadat alles lukte in Sensu ben ik begonnen met het opzetten van een etcd cluster voor de sensu-backend met 3 VM's. nl: `backend.example.com`, `backend2.example.com` en `backend3.example.com`. Om de cluster op te zetten moeten op alle 3 de VM's de sensu-backend draaien. Ook moeten een aantal etcd parameters aan elke backend meegegeven worden.

- Configuratie voor de sensu-backend:

```yaml
---
classes:
   - sensu::backend
   - sensu::agent

sensu::api_host: backend.example.com
sensu::api_port: 8080
sensu::username: admin
sensu::password: 
sensu::old_password: 
sensu::use_ssl: false

```

- Configuratie voor de etcd-cluster:

```yaml
sensu::backend::config_hash:
  etcd-advertise-client-urls: "http://%{facts.networking.interfaces.lo.ip}:2379"
  etcd-listen-client-urls: "http://%{facts.networking.interfaces.lo.ip}:2379"
  etcd-listen-peer-urls: 'http://[::]:2380'
  etcd-initial-cluster: 'icts-t-sensu-1=http://***:2380,icts-t-sensu-2=http://***:2380,icts-t-sensu-3=http://***:2380'
  etcd-initial-advertise-peer-urls: "http://%{facts.networking.interfaces.lo.ip}:2380"
  etcd-initial-cluster-state: 'new'
  etcd-initial-cluster-token: ''
  etcd-name: "%{::hostname}"

```

De puppet agents runnen facter om een lijst van facts te krijgen over het systeem. `facts.networking.interfaces.eth1.ip` gaat dus het ip address van interface eth1 resolven.

- Als laatste moeten we aan de sensu-agents de ip addressen van deze 3 backends nog meegeven.

```puppetcode
sensu::agent::backends:
  - 'ws://192.168.52.10:8081'
  - 'ws://192.168.52.21:8081'
  - 'ws://192.168.52.22:8081'
```

Om de yaml bestanden te valideren:

`ruby -e "require 'yaml'; YAML.load_file('/vagrant/hieradata/nodes/backend.example.com.yaml')`

### Sensu agents

Voorlopig heb ik in mijn setup 2 sensu-agents. Op de agents installeer ik via puppet een apache webserver. Hiervoor maak ik eerst via puppet de directory structuur /var/www/html/fqdn aan. Hier komen de html files van de vhosts in. In deze directory maak ik dan een index.html file ùet als titel de fully qualified domain name.

```yaml

class profile::vhosts(
){
  $fqdn = "${facts['fqdn']}"
 file { [ '/var/www',
          '/var/www/html',
          "/var/www/html/$fqdn", ]:
   ensure       => directory,
 }

 file {"/var/www/html/$fqdn/index.html":
   ensure       => file,
   content      => "<h1>$fqdn</h1>",
 }
}  
```

Om apache via puppet op de agent te krijgen moeten we eerst de apache module van de forge halen en installeren. In de Puppetfile steken we de apache module:

```puppetcode
forge "https://forgeapi.puppetlabs.com"

# modules from the puppetforge

mod 'puppetlabs-ntp'
mod 'puppetlabs-stdlib'

mod 'sensu-sensu', '4.7.0'
mod 'puppetlabs-apache', '5.4.0'
# modules from github

```

Daarna installeren met volgend commando: `make install-modules`

Nu moeten we enkel de apache class includen in de agent node en de vhost definiëren.

```yaml
---
classes:
  - sensu::agent
  - apache
  - profile::vhosts

sensu::agent::backends:
  - 'ws://192.168.52.10:8081'
  - 'ws://192.168.52.11:8081'
  - 'ws://192.168.52.12:8081'
sensu::agent::subscriptions:
  - linux
  - proxy
  - apache-servers
sensu::use_ssl: false

apache::vhost:
   servername: agent1.example.com
   port: 80
   default_vhost: true
   ssl: false
   docroot: /var/www/html/agent1.example.com

```

![screenshot vhost apache](pictures/apache.png)

Nu apache draait op beide agents kunnen we beginnen met deze service te monitoren. Hiervoor ga ik 2 checks maken. Een check die de metrics opvraagt van de vhost en één check die kijkt of ik status 200 terugkrijg.

Voor deze checks hebben we de asset sensu-plugins-http nodig. Deze definieren we terug op de backend-node met puppet:

```yaml
sensu-plugins/sensu-plugins-http:
     ensure: present
     version: latest
```

Dan creeren we twee proxy entities die de 2 website's gaan monitoren:

```yaml
sensu::resources::entities:
   agent1-site:
     namespace: default
     labels:
       url: "http://192.168.52.13/agent1.example.com/"
       proxy_type: "website"
     ensure: present
     entity_class: proxy


   agent2-site:
     namespace: default
     labels:
       proxy_type: "website"
       url: "http://192.168.52.14/agent2.example.com/"
     ensure: present
     entity_class: proxy

```

Dan maken we de twee checks zelf aan. De eerste gaat kijken of de apache service draait op de sensu-agent. De tweede gaat naar bovenstaande url's de pagina's opvragen en kijken wellke status ze teruggeven. Als deze status 400 teruggeven krijgen we in Slack een bericht hiervan.

```yaml
   check_apache:
     ensure: present
     command: metrics-curl.rb -u "http://localhost:80"
     interval: 60
     subscriptions: apache-servers
     publish: true
     runtime_assets:
       - sensu-plugins/sensu-plugins-http
       - sensu/sensu-ruby-runtime

   check-http:
     ensure: present
     command: check-http.rb -u {{ .labels.url }}
     interval: 60
     proxy_requests:
       entity_attributes:
         - entity.entity_class == proxy
         - entity.labels.proxy_type == website
     subscriptions: proxy
     publish: true
     handlers: slack
     round_robin: true
     runtime_assets:
       - sensu-plugins/sensu-plugins-http
       - sensu/sensu-ruby-runtime
```

### TLS

Na het opstarten met vagrant moet ik nu voorlopig nog manueel de gebruiker sensu rechten geven aan de folder /vagrant/tls/* waar alle certificaten in staan. Dit moet op elke backend gebeuren. Dit is met volgend commando:

```bash
chown sensu /vagrant/tls/*
```

Om de certificaten zelf te genereren die we gebruiken voor de api en dashboard te beveiligen alsook de etcd peer communication:

CA:

```bash
echo '{"CN":"CA","key":{"algo":"rsa","size":2048}}' | cfssl gencert -initca - | cfssljson -bare ca -
echo '{"signing":{"default":{"expiry":"43800h","usages":["signing","key encipherment","server auth","client auth"]}}}' > ca-config.json
```

backends:

```bash
export ADDRESS=192.168.52.10,backend
export NAME=backend
echo '{"CN":"'$NAME'","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -config=ca-config.json -ca=ca.pem -ca-key=ca-key.pem -hostname="$ADDRESS" -profile=peer - | cfssljson -bare $NAME

export ADDRESS=192.168.52.11,backend2
export NAME=backend2
echo '{"CN":"'$NAME'","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -config=ca-config.json -ca=ca.pem -ca-key=ca-key.pem -hostname="$ADDRESS" -profile=peer - | cfssljson -bare $NAME

export ADDRESS=192.168.52.12,backend3
export NAME=backend3
echo '{"CN":"'$NAME'","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -config=ca-config.json -ca=ca.pem -ca-key=ca-key.pem -hostname="$ADDRESS" -profile=peer - | cfssljson -bare $NAME
```

agent:

```bash
export NAME=client
echo '{"CN":"'$NAME'","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -config=ca-config.json -ca=ca.pem -ca-key=ca-key.pem -hostname="" -profile=client - | cfssljson -bare $NAME
```

Dan om deze certificaten te gebruiken moeten we op elke backend volgende parameters configureren:

```YAML
sensu::ssl_ca_source: '/vagrant/tls/ca.pem'
sensu::backend::ssl_cert_source: "/vagrant/tls/%{facts.fqdn}.pem"
sensu::backend::ssl_key_source: "/vagrant/tls/%{facts.fqdn}-key.pem"

sensu::backend::config_hash:
  etcd-advertise-client-urls: "https://%{facts.networking.interfaces.eth1.ip}:2379"
  etcd-listen-client-urls: "https://%{facts.networking.interfaces.eth1.ip}:2379"
  etcd-listen-peer-urls: 'https://0.0.0.0:2380'
  etcd-initial-cluster: 'backend=https://192.168.52.10:2380,backend2=https://192.168.52.11:2380,backend3=https://192.168.52.12:2380'
  etcd-initial-advertise-peer-urls: "https://%{facts.networking.interfaces.eth1.ip}:2380"
  etcd-initial-cluster-state: 'new'
  etcd-initial-cluster-token: ''
  etcd-name: 'backend'

  etcd-cert-file: "/vagrant/tls/%{facts.fqdn}.pem"
  etcd-key-file: "/vagrant/tls/%{facts.fqdn}-key.pem"
  etcd-trusted-ca-file: "/vagrant/tls/ca.pem"
  etcd-peer-cert-file: "/vagrant/tls/%{facts.fqdn}.pem"
  etcd-peer-key-file: "/vagrant/tls/%{facts.fqdn}-key.pem"
  etcd-peer-client-cert-auth: "true"
  etcd-peer-trusted-ca-file: "/vagrant/tls/ca.pem"

```

In de urls is http vervangen door https. Om de agents nu met de backends te laten communiceren moeten we de backend-url's gerbuik laten maken van een websocket over SSL/TLS. Dit doen we door ws:// te vervangen door wss://

```YAML
sensu::use_ssl: true
sensu::ssl_ca_source: "/vagrant/tls/ca.pem"

sensu::agent::backends:
  - 'wss://192.168.52.10:8081'
  - 'wss://192.168.52.11:8081'
  - 'wss://192.168.52.12:8081'
```

Ook geven we de ca file mee.

## Sensu in testomgeving

Er zijn een aantal wijzigingen voor sensu in puppet t.o.v. mijn lokale setup.
Zo heb ik door compatibiliteitsproblemen met de stdlib module een oudere sensu-module moeten gebruiken. Daardoor zijn er een aantal wijzigingen

### Firewall

Voor de firewall heb ik 3 services gedefinieerd:

- sensu_go : poort 8081
- sensu_go_etcd: poort 2379 en 2380
- sensu_go_dashboard: poort 3000

Deze services zitten in de publieke zone. Dit betekent ook dat de ip-addressen gebruikt voor etcd vervangen moeten worden. De volledige yaml voor `hieradata/role/sensu/tier/test.yaml` ziet er dan als volgt uit wanneer er nog geen etcd cluster draait:

`hieradata/role/sensu/common/tier/test.yaml`:

```yaml
---
classes:
  - profile::sensu::backend
  - sensu::agent
  - sensu::backend

sensu::version: 5.20.0-12118
sensu::backend::url_host: "%{facts.fqdn}"
sensu::backend::url_port: 8080
sensu::username: admin
sensu::backend::password: "%{lookup('vault_sensu::backend::password')}"
sensu::backend::old_password: 

sensu::manage_repo: false
sensu::plugins::manage_repo: false
sensu::use_ssl: true
sensu::ssl_ca_source: '/etc/puppetlabs/puppet/ssl/certs/ca.pem'
sensu::backend::ssl_cert_source: "/etc/puppetlabs/puppet/ssl/certs/%{facts.fqdn}.pem"
sensu::backend::ssl_key_source: "/etc/puppetlabs/puppet/ssl/private_keys/%{facts.fqdn}.pem"

sensu::backend::config_hash:
  etcd-advertise-client-urls: "http://[%{facts.network_public_ipv6}]:2379"
  etcd-listen-client-urls: "http://[%{facts.network_public_ipv6}]:2379"
  etcd-listen-peer-urls: 'http://[::]:2380'
  etcd-initial-cluster: 'icts-t-sensu-1.***=http://***:2380,icts-t-sensu-2.***=http://***:2380,icts-t-sensu-3.***=http://***:2380'
  etcd-initial-advertise-peer-urls: "http://[%{facts.network_public_ipv6}]:2380"
  etcd-initial-cluster-state: 'new'
  etcd-initial-cluster-token: ''
  etcd-name: "%{facts.fqdn}"
  log-level: debug

```

De `sensu::version: 5.20.0-12118` hebben we specifiek meegegeven zodat alle drie de backends dezelfde versie hebben en zo geen errors geven bij de balancer door een verschil in de dashboards.

De firewalld services staan in
`hieradata/role/sensu/common/common.yaml`:

```yaml
firewalld::custom_services:
  sensu_go:
    description: Ports for Sensu Go
    port:
      - port: '8081'
        protocol: tcp
  sensu_go_dashboard:
    description: Ports for Sensu Go dashboard
    port:
      - port: '3000'
        protocol: tcp
  sensu_go_etcd:
    description: Ports for Sensu etcd
    port:
      - port: '2379'
        protocol: tcp
      - port: '2380'
        protocol: tcp

profile::firewalld::services::public:
  - sensu_go
  - sensu_go_etcd
  - sensu_go_dashboard
```

### Initaliseren backends

Op 1 van de backends moet de sensu-backend nog geinitialiseerd worden. Volgende 3 commando's moeten dan als root eenmalig worden gedraaid. Nadat de etcd cluster een keer draait moet in de yaml `etcd-initial-cluster-state: 'new'` new vervangen worden door `existing`!
Als er al een etcd cluster gedraaid heeft maar gestopt is moet je de etcd data verwijderen door eerst de `sensu-backend` te stoppen en dan de inhoud van `var/lib/sensu/sensu-backend/etcd` te verwijderen.

```bash
export SENSU_BACKEND_CLUSTER_ADMIN_USERNAME=YOUR_USERNAME
export SENSU_BACKEND_CLUSTER_ADMIN_PASSWORD=YOUR_PASSWORD
sensu-backend init
```

>If the default password is not set, by default, only one member of the cluster (the first) will >be configured correctly, as the configure step assumes the password is still unchanged. To fix >the other backends, a command is added to `profile::sensu::backend`.

`profile::sensu::backend`:

```yaml
class profile::sensu::backend {
    Yumrepo["sensu"] -> Class["sensu"]
    include ::sensu::backend
    # Fix configure step in puppet module for clusters
    $password = lookup('sensu::backend::password')
    exec { 'fix sensuctl configure step in clusters':
      command => "/usr/bin/sensuctl configure --trusted-ca-file /etc/puppetlabs/puppet/ssl/certs/ca.pem --non-interactive --url https://${::fqdn}:8080 --username admin --password ${password} || /bin/true",
      unless  => '/usr/bin/sensuctl cluster member-list',
      before  => Service[$::sensu::backend::service_name],
      require => Package[$::sensu::backend::package_name],
    }
    Class['::profile::firewalld'] -> Class['::sensu::backend']
}
```

### Mirror repository

Om gebruik te maken van de mirror repositorie van de KU Leuven moeten we deze 2 parameters op false zetten in:  
`hieradata/common/sensu.yaml`

```yaml
sensu::manage_repo: false
sensu::plugins::manage_repo: false
```

Om van de mirror gebruik te maken staan volgende lijnen in
`hieradata/common/sensu.yaml`

```yaml
profile::repositories:
  sensu:
    enabled: 0

profile::repositories::config:
  sensu:
    descr: Sensu repository
    baseurl: https://***/%{facts.os.release.major}/$basearch
    tag: kickstartpost
    extra_options:
      metadata_expire: 3600
```

Waneer je sensu op een vm wilt zetten moet je de `enabled: 0` op 1 zetten.
Bv in `hieradata/role/cloud/common/tier/test.yaml`:

```yaml
---
classes:
  - sensu::agent

profile::repositories:
  sensu:
    enabled: 1
```

### Checks, assets, filters

De checks, assets, handlers, filters... definieren we in `hieradata/role/sensu/common/tier/test.yaml`
Om de assets te gebruiken moeten we de assets repository mirroren zodat de virtuele machine niet elke keer naar het internet moet.
Ook heeft KU Leuven voor in de HPC een aantal checks zelf geschreven. Deze maken geen gebruik van de assets op [bonsai](https://bonsai.sensu.io) en staan daarom niet in <https://repo.icts.kuleuven.be/artifactory/sensu-assets/>.

`hieradata/role/sensu/common/tier/test.yaml`:

```yaml
sensu::backend::assets:
  # Note: Deze asset maakt geen gebruik van de sensu-assets repo, aantal checks zijn geschreven door KU Leuven zelf.
  # Deze checks maken daarom ook geen gebruik van de sensu bonsai assets.
  kuleuven-checks:
    ensure: present
    url: https://***/go-sensu-checks-29.tar.gz
    sha512: 49ee97c06170e2e9f1ec70fabc22643948c2c2960d715eec31a2fb34647d411c2248139c68f47cec7e486570eba52d0be68d137f481afdeeddb48e90ad196fb7
    filters:
      - "entity.system.os == 'linux'"
      - "entity.system.arch == 'amd64'"

  # Note: Onderstaande assets zijn gemirrord van de sensu bonsai-assets repo.
  # zodat de backends niet naar het internet moeten om deze assets op te halen.
  sensu-plugins-cpu-checks:
    ensure: present
    url: https://***/sensu-assets/68546e739d96fd695655b77b35b5aabfbabeb056/sensu-plugins-cpu-checks_4.0.0_centos_linux_amd64.tar.gz
    sha512: 518e7c17cf670393045bff4af318e1d35955bfde166e9ceec2b469109252f79043ed133241c4dc96501b6636a1ec5e008ea9ce055d1609865635d4f004d7187b
    filters:
      - "entity.system.os == 'linux'"
```

### Agents

Momenteel monitoren we de `icts-t-cloud` en `icts-t-mysql` virtuele machines. Dit doen we door in `hieradata/role/cloud/common/tier/test.yaml` de sensu-agent te definiëren.
We moeten aan de agent de 3 backend url's meegeven alsook de 2 `manage_repo` parameters op `false`te zetten.
De file ziet er dan als volgt uit:

in `/hieradata/tier/test.yaml` staan de backend url's:

```yaml
sensu::agent::backends:
  - icts-t-sensu-1.***:8081
  - icts-t-sensu-2.***:8081
  - icts-t-sensu-3.***:8081
```

En in `hieradata/role/cloud/common/tiet/test.yaml` en in `hieradata/role/mysql/backend/tier/test.yaml`:

```yaml
---
classes:
  - sensu::agent

profile::repositories:
  sensu:
    enabled: 1

```

Om de subscriptions te definiëren hebben we in `hieradata/common/sensu.yaml` volgende subscriptions opgelijst:

```yaml
sensu::agent::config_hash:
  subscriptions: "%{alias('sensu_subscriptions')}"
```

en in `hieradata/common.yaml`:

```yaml
sensu_subscriptions:
  - linux
  - "datacenter-%{::datacenter}"
  - "tier-%{::tier}"
  - "role-%{::role}"
  - "role-%{::role}/%{::tier}"
  - "role-%{::role}/%{::subrole}"
  - "role-%{::role}/%{::subrole}/%{::tier}"
  - "role-%{::role}/%{::subrole}/%{::tier}/%{::datacenter}"
  - "role-%{::role}/%{::subrole}/%{::datacenter}"
```

Om dit te gebruiken moeten we in `hieradata/common/lookup.yaml`:

```yaml
 sensu_subscriptions:
    merge: unique
```

### Reverse proxy

Op de reverse proxies is de site `sensu.t.***` gedefinieerd. In de configuratie van de reverse proxy staat ook nog op welke backends servers de site draait. In ons geval zijn dat de 3 sensu backends servers waarvan het dashboard op poort 3000 runt.

In puppet krijg je dan het volgende:

```yaml
reverseproxy::sites:
  "sensu.%{lookup('revprox_domain')}-https":
    servername: sensu.t.***
    protocol: https
    mappings:
      root:
        location: /
        parameters: "%{alias('authentication-templates.must-have-unr')}"
        balancer: icts-t-sensu-https
    extra_options:
      headers:
        - always set Referrer-Policy "strict-origin"
        - always set X-Frame-Options SAMEORIGIN
        - always set X-Content-Type-Options "nosniff"
        - set X-XSS-Protection "1; mode=block"
  "sensu.%{lookup('revprox_domain')}-http":
    servername: "sensu.%{lookup('revprox_domain')}"
    protocol: http
    rewrites: "%{alias('rewrites.rewrite-http-to-https')}"

reverseproxy::balancers:
  icts-t-sensu-https:
    backends:
      - sensu-1-https
      - sensu-2-https
      - sensu-3-https

reverseproxy::backends:
  sensu-1-https:
    protocol: https
    server: icts-t-sensu-1.***
    port: 3000
  sensu-2-https:
    protocol: https
    server: icts-t-sensu-2.***
    port: 3000
  sensu-3-https:
    protocol: https
    server: icts-t-sensu-3.***
    port: 3000
```

### Shibboleth

Voor shibboleth te gebruiken staan volgende lijnen in `hieradata/role/revprox/common/tier/test.yaml`

```yaml
shibboleth::applications:
  "sensu.%{lookup('revprox_domain')}":
    entityID: "https://sensu.%{lookup('revprox_domain')}"
    metadata: kuleuven
    credentialResolver: cr_shib
    session: idp-p
    hosts:
      "sensu.%{lookup('revprox_domain')}": {}
```

![screenshot sensu setup](pictures/sensu.png)
