# Circuit Breaking

## Objectifs

1. Configurer des règles de circuit breaking
2. Tester le circuit breaker

## Installation des outils d'exemples

### Fortio

> Fortio nous servira de client pour faire des appels au service `productpage`
> 
>[documentation](https://github.com/fortio/fortio)

Déployer l'application à l'aide du fichier [deploy-fortio.yml](./Fortio/InstallFortio/deploy-fortio.yml)

```bash
kubectl apply -f ./Fortio/InstallFortio/deploy-fortio.yml
```

## Application des règles

Appliquer le fichier de configuration [productpage-destination-rules.yml](./productpage-destination-rules.yml)

```bash
kubectl apply -f ./productpage-destination-rules.yml
```

## En détail

```yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  namespace: poc-bbo
  name: productpage
spec:
  host: productpage
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1 #1#
        maxRequestsPerConnection: 1 #2#
    outlierDetection: #3#
      consecutive5xxErrors: 1 #4#
      interval: 1s #5#
      baseEjectionTime: 3m #6#
      maxEjectionPercent: 100 #7#
  subsets:
    - name: v1
      labels:
        version: v1
```

1. Nombre maximal de requêtes HTTP en attente vers une destination.
2. Nombre maximal de requêtes par connexion à un backend.
3. `outlierDetection` définit la section de detection des valeurs aberrantes.
4. Nombre d'erreurs 5xx avant qu'un hôte ne soit éjecté du pool de connexion.
5. L'intervalle de temps pour l'analyse d'éjection.
6. Durée minimale d'éjection. Un hôte restera éjecté pendant une période égale au produit de la durée d'éjection minimale et du nombre de fois où l'hôte a été éjecté. Cette technique permet au système d'augmenter automatiquement la période d'éjection des serveurs défectueux en amont .
7. % maximal d'hôtes dans le pool qui peuvent être éjectés

[TrafficPolicy](https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy)

[ConnectionPoolSettings.HTTPSettings](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-HTTPSettings)

[Détection des valeurs aberrantes](https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection)

### Tester

Vous pouvez vous connecter à l'ihm de Fortio via htpp://$<istio-ingressgateway.ExternalIP>/fortio

![fortio](/assets/fortio.png)

1. Url du service a appeler (ici l'url intra cluster nous pourrions aussi bien passer par la gateway)
2. Query per second (requêtes par seconde) ici 0 nous ne mettons aucun délai entre les requêtes
3. Nombre de requêtes par connexion
4. Nombre de connexions à ouvrir

```bash
Starting at 8 qps with 5 thread(s) [gomax 4] : exactly 50, 10 calls each (total 50 + 0)
Ended after 6.34113039s : 50 calls. qps=7.885
Sleep times : count 45 avg 0.67771117 +/- 0.0226 min 0.625108044 max 0.69210216 sum 30.4970028
Aggregated Function Time : count 50 avg 0.01774835 +/- 0.02514 min 0.001795816 max 0.09072715 sum 0.887417512
# range, mid point, percentile, count
>= 0.00179582 <= 0.0018 , 0.00179791 , 2.00, 1
> 0.0018 <= 0.002 , 0.0019 , 10.00, 4
> 0.002 <= 0.0025 , 0.00225 , 22.00, 6
> 0.0025 <= 0.003 , 0.00275 , 30.00, 4
> 0.003 <= 0.0035 , 0.00325 , 38.00, 4
> 0.0035 <= 0.004 , 0.00375 , 52.00, 7
> 0.004 <= 0.0045 , 0.00425 , 64.00, 6
> 0.0045 <= 0.005 , 0.00475 , 66.00, 1
> 0.005 <= 0.006 , 0.0055 , 74.00, 4
> 0.045 <= 0.05 , 0.0475 , 82.00, 4
> 0.05 <= 0.06 , 0.055 , 90.00, 4
> 0.06 <= 0.07 , 0.065 , 98.00, 4
> 0.09 <= 0.0907272 , 0.0903636 , 100.00, 1
# target 50% 0.00392857
# target 75% 0.045625
# target 90% 0.06
# target 99% 0.0903636
# target 99.9% 0.0906908
Sockets used: 39 (for perfect keepalive, would be 5)
Jitter: false
Code 200 : 12 (24.0 %)
Code 503 : 38 (76.0 %)
Response Header Sizes : count 50 avg 40.32 +/- 71.75 min 0 max 168 sum 2016
Response Body/Total Sizes : count 50 avg 1465.64 +/- 2183 min 153 max 5351 sum 73282
Saved result to data/2022-03-31-133533_Fortio.json (graph link)
All done 50 calls 17.748 ms avg, 7.9 qps
```

Nous remarquons avec cette configuration de Fortio que 24% des requêtes aboutissent et que les autre sont interceptées par le circuit Breaker

```bash
Code 200 : 12 (24.0 %)
Code 503 : 38 (76.0 %)
```

Statistiques confirmées par Kiali.


![circuitBreaking](/assets/circuitBreaking.png)

## Aller plus loin sur la Détection des valeurs aberrantes

La détection des valeurs aberrantes est une stratégie de résilience d'Istio pour détecter le comportement inhabituel de l'hôte et expulser les hôtes non sains de l'ensemble des hôtes sains à charge équilibrée à l'intérieur d'un cluster. Il suit automatiquement l'état de chaque hôte individuel et vérifie les métriques telles que les erreurs consécutives et la latence associées aux appels de service. S'il trouve des valeurs aberrantes, il les expulsera automatiquement.

L'éjection/expulsion implique que l'hôte est identifié comme non sain et ne sera pas utilisé pour répondre aux demandes des utilisateurs dans le cadre du processus d'équilibrage de charge. Si une demande est envoyée à une instance de service et qu'elle échoue (renvoie un code d'erreur 50X), Istio éjecte l'instance du pool pendant une durée spécifiée.

L'ensemble de ce processus augmente la disponibilité globale des services en s'assurant que seuls les pods sains participent aux demandes des utilisateurs

![Outlier Detection](https://dotnetvibes.files.wordpress.com/2019/06/istio-outlier-detection.png?w=700)

Avec les paramètres ci-dessous, le service dépendant est analysé toutes les 10 secondes et s'il y a des hôtes qui échouent plus de 3 fois avec un code d'erreur 5XX, il sera éjecté du pool pendant 20 secondes.

![conf](https://dotnetvibes.files.wordpress.com/2019/06/outlier-detection-configurations.png?w=700)

sources:

[istio.io](https://istio.io/)

[dzone.com](https://dzone.com/articles/istio-circuit-breaker-with-outlier-detection)
