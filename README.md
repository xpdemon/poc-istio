# POC Istio Service-Mesh

## Descriptions

[Istio](https://istio.io/) est une plateforme Open Source de [Service Mesh](https://www.redhat.com/fr/topics/microservices/what-is-a-service-mesh) qui permet de contrôler la manière dont les données sont partagées entre les [microservices](https://www.redhat.com/fr/topics/microservices/what-are-microservices). Elle comprend des [API](https://www.redhat.com/fr/topics/api/what-are-application-programming-interfaces) qui permettent d'[intégrer](https://www.redhat.com/fr/topics/integration/what-is-integration) Istio à tout type de plateforme de journalisation, de télémétrie ou de système de politiques. La plateforme Istio est conçue pour fonctionner dans de nombreux environnements : sur site, hébergé dans le cloud, dans des [conteneurs](https://www.redhat.com/fr/topics/containers) [Kubernetes](https://www.redhat.com/fr/topics/containers/what-is-kubernetes), dans des services exécutés sur des [machines virtuelles](https://www.redhat.com/fr/topics/virtualization/what-is-a-virtual-machine), etc.

## Ressources

* [Documentation Istio](https://istio.io/latest/docs/)
* [Conférence DEVOXX](https://www.youtube.com/watch?v=03m9KsI7EtI)

## Fonctionnement global

Istio comporte deux composants : le data plane et le control plane.

Le data plane est la communication entre les services. Sans maillage de services, le réseau ne comprend pas le trafic envoyé et ne peut prendre aucune décision en fonction du type de trafic, de qui il provient ou de qui il est destinataire.

Le service mesh utilise un proxy pour intercepter tout votre trafic réseau, permettant un large éventail de fonctionnalités compatibles avec les applications en fonction de la configuration que vous définissez.

Un Envoy proxy est déployé avec chaque service que vous démarrez dans votre cluster, ou s'exécute avec des services exécutés sur des machines virtuelles.

Le control plane prend la configuration souhaitée et sa vue des services, et programme dynamiquement les serveurs proxy, en les mettant à jour au fur et à mesure que les règles ou l'environnement changent.

![After utilizing Istio](https://istio.io/latest/img/service-mesh.svg)

Sources:  [https://istio.io/latest/about/service-mesh]()

## Cas de test du POC

### Traffic Management

1. **Request Routing** ( routage des requêtes )
   1. [documentation](https://istio.io/latest/docs/tasks/traffic-management/request-routing/)
   2. [usages](./TrafficManagement/RequestRouting/requestRouting.md)
2. **Traffic Shifting** ( redirection du traffic )
   1. [documentation](https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/)
   2. [usages](./TrafficManagement/TrafficShifting/trafficShifting.md)
3. **Circuit Breaking** ( coupure de circuit )
   1. [documentation](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/https:/)
   2. [usage](./TrafficManagement/CircuitBreaking/circuitBreaking.md)
4. **Fault injection** ( injection d'erreurs )
   1. [documentation](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)
   2. [usage](TrafficManagement/FaultInjection/faultInjection.md)
5. **Request Timeouts** (délais d'expiration de requêtes)
   1. [documentation](https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/)
   2. [usage](./TrafficManagement/RequestTimeouts/requestsTimeOuts.md)
6. **Mirroring** (duplication du traffic)
   1. [documentation](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)
   2. [usage](./TrafficManagement/Mirroring/mirroring.md)
7. **TCP Traffic Shifting** ( redirection TCP )
   1. [documentation](https://istio.io/latest/docs/tasks/traffic-management/tcp-traffic-shifting/)
   2. [WIP](./TrafficManagement/TCPTrafficShifting/tcpTrafficShifting.md)
8. **Request Retry** (nouvelle tentative des requêtes)
   1. [documentation](https://istio.io/latest/docs/concepts/traffic-management/#retries)
   2. [usages](./TrafficManagement/RequestRetry/retry.md)

### Sécurité / chiffrement

1. **Certificate Management** (gestion des certificats)
   1. [documentation](https://istio.io/latest/docs/tasks/security/cert-management/)
   2. usage
      1. [mode simple](./Securite/CertificateManagement/Standalone/standaloneTls.md)
      2. [Cert-Manager](./Securite/CertificateManagement/WithCert-manager/tlsWithCertManager.md)
2. **Authorization** (gestion des autorisations d'accès)
   1. [documentation](https://istio.io/latest/docs/tasks/security/authorization/)
   2. [WIP](./Securite/Authorization/authorization.md)

## Vocabulaire

#### Virtual Service:

Configuration affectant le routage du trafic.

Un **virtual service** définit un ensemble de règles de routage du trafic à appliquer lorsqu'un hôte est adressé. Chaque règle de routage définit des critères de correspondance pour le trafic d'un protocole spécifique. Si le trafic correspond, il est envoyé à un service de destination nommé (ou à un **subset** /version de celui-ci) défini dans le registre.

[documentation](https://istio.io/latest/docs/reference/config/networking/virtual-service/)

#### Subset:

Aussi nommé **Service versions**, c'est un regroupement de toutes les instances d'un service donné par version. Cela permet de personnaliser les politiques de trafic.

#### DestinationRule:

Définit les politiques qui s'appliquent au trafic destiné à un service après le routage. Ces règles spécifient la configuration de l'équilibrage de charge, la taille du pool de connexions du side-car et les paramètres de détection des valeurs aberrantes pour détecter et expulser les hôtes non sains du pool d'équilibrage de charge.

Les règles de destination sont utilisées pour configurer ce qu'il advient du trafic au point de terminaison, qui est acheminé par le service virtuel.

> Les règles de destination sont appliquées après l'évaluation des règles de
>
> routage des services virtuels, elles s'appliquent donc à la destination « réelle » du trafic.

La destination **réelle** est configurée à l'aide de`subsets`  .

Sans version de service par défaut explicite vers laquelle router, Istio achemine les demandes vers toutes les versions disponibles de manière circulaire. Il prend également en charge les modèles suivants.

* **Random** (Aléatoire) : les requêtes sont transmises au hasard aux instances du pool.
* **Weighted** (Pondéré) : les requêtes sont transmises aux instances du pool selon un pourcentage spécifique.
* **Least requests** (Equilibré) : les requêtes sont transmises aux instances avec le moins de demandes.

[documentation](https://istio.io/latest/docs/reference/config/networking/destination-rule/)

#### Gateway:

Décrit un load balancer fonctionnant à la périphérie du maillage recevant des connexions HTTP/TCP entrantes ou sortantes. La spécification décrit un ensemble de ports qui doivent être exposés, le type de protocole à utiliser, la configuration SNI pour le load balancer, etc.

[documentation](https://istio.io/latest/docs/reference/config/networking/gateway/)

#### Sidecars:

À l'origine, Istio a ses propres proxys Envoy side-car injectés pour les services. Ces side-cars sont configurés pour accepter le trafic sur tous les ports associés à l'application, pour atteindre chaque workload dans le maillage de services lors du transfert du trafic. Ces side-cars peuvent également être configurés pour **affiner l'ensemble des ports et des protocoles qu'un Envoy proxy accepte** et **limiter l'ensemble des services que l' Envoy proxy peut atteindre** .

#### Circuit Breaking:

Cette fonctionnalité permet de définir des limites d'appels vers des hôtes individuels au sein d'un service. Tels que le nombre de connexions simultanées ou le nombre d'échecs d'appels vers cet hôte, etc. Une fois cette limite atteinte, le Circuit Breaker "se déclenche" et arrête d'autres connexions vers cet hôte. Le modèle de Circuit Breaking permet une défaillance rapide plutôt que des clients essayant de se connecter à un hôte surchargé ou défaillant.

Les règles de Circuit Breaker sont appliquées sur les `destinationRule`, les paramètres s'appliquant à chaque hôte individuel du service, car ces configurations sont appliquées dans un pool d'équilibrage de charge.

##### sources :

* [faun.pub](https://faun.pub/istio-step-by-step-part-13-istio-traffic-management-istio-core-features-e513bfd66fb4)
* [istio.io](https://istio.io/latest/docs/)
