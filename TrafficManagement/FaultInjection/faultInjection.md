# Fault Injection

## Objectifs

Effectuer des tests de chaos au niveau de la couche d'application, en injectant des délais d'attente ou des erreurs HTTP dans vos services,

## Application des règles

### Raz de la configuration de l'application test (optionnel)

Appliquer le fichier [default-init-book-info.yml](./delay/default/default-init-book-info.yml)

```bash
kubectl apply -f ./delay/default/default-init-book-info.yml
```

### Erreur de délai HTTP

Appliquer le fichier de configuration [delay-fault-injection.yml](./delay/delay-fault-injection.yml)

```bash
kubectl apply -f ./delay/delay-fault-injection.yml
```

#### Tester la configuration

Rendons nous sur la page web de l'application `http://$<istio-ingerssgateway.ExternalIP>/productpage` et logons nous avec l'utilisateur "bbo" (dans notre cas de test).

Nous remarquons:

1. Que le temps d'affichage de la page est relativement long (environ 6 secondes).
2. Que les reviews ne s'affichent pas nous avons à la place un message d'erreur

#### En détail

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  namespace: poc-bbo
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - match:
        - headers:
            end-user:
              exact: bbo
      fault: #1#
        delay: #2#
          percentage: #3#
            value: 100.0
          fixedDelay: 7s #4#
      route:
        - destination:
            host: ratings
            subset: v1
    - route:
        - destination:
            host: ratings
            subset: v1
```

1. Section d'injection de fautes
2. Type de faute à injecter
3. % du traffic touché par l'injection
4. délai avant de transmete la requête DOIT ETRE  **>=1ms**

[HTTPFaultInjection-Delay](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Delay)

Il y a plusieur choses à savoir sur l'application d'exemple :

1. Un Timeout codé en dur de 10 secondes sur le service `reviews:v2` vers le service `rating`
2. Un Timeout codé en dur de 3 seconde sur le service `productpage`
3. Un Retry codé en dur d'une tentative sur le service `productpage`
4. Un Timeout codé en dur de 2.5 secondes sur le  service `reviews:v3`

Par concéquant en introduisant un temps de reponse de 7 secondes sur le service `rating` le service `reviews:v2` se comporte normalement  ( 7 < 10 ).

Par contre le service `productpage` lui n'acceptera au maximum que 6 secondes de délai (3 secondes de timeout  * 2 tentatives ) ce qui provoque ici notre erreur (7 > 10) .

![delayFaultInjection](/assets/delayFaultInjection.png)

Pour résoudre ce problème nous pouvons [tansférer le traffic](/TrafficManagement/TrafficShifting/trafficShifting.md) du service `reviews:v2` (10sc de timeout) vers le service `reviews:v3` (2.5 secondes de timeout) et reduire notre injection de faute à  un délai inferieur a 2.5 secondes ,

en appliquant le fichier de configuration [resolve-fault.yml](./delay/resolve-fault.yml)

```bash
kubectl apply -f ./delay/resolve-fault.yml
```

### Erreur Abandon des requêtes HTTP

Appliquer le fichier de configuration [abort-rating-virtual-service](./httpAbortFault/abort-rating-virtual-service.yml)

```bash
kubectl apply -f ./httpAbortFault/abort-rating-virtual-service.yml
```

#### Tester la configuration

Rendons nous sur la page web de l'application `http://$<istio-ingerssgateway.ExternalIP>/productpage` et logons nous avec l'utilisateur "bbo" (dans notre cas de test).

Nous remarquons que :

1. La page web nous indique que le service `rating` n'est pas disponible.
2. si nous ne nous authentifions pas, aucun problème.

![faultInjectionAbort](/assets/faultInjectionAbort.png)

#### En détail

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  namespace: poc-bbo
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - match:
        - headers:
            end-user:
              exact: bbo
      fault:
        abort:
          percentage:
            value: 100.0
          httpStatus: 500 #1#
      route:
        - destination:
            host: ratings
            subset: v1
    - route:
        - destination:
            host: ratings
            subset: v1
```

1. `httpStatus` représente le code que l'Envoy proxy doit retourner

[HTTPFaultInjection-Abort](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPFaultInjection-Abort)
