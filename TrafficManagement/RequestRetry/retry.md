# Request Retry

## Contexte

Le comportement par défaut du **retry** pour les requêtes HTTP consiste à réessayer deux fois avant de renvoyer l'erreur.

L'intervalle entre les tentatives (plus de 25 ms) est variable et déterminé automatiquement par Istio, évitant ainsi au service appelé d'être submergé de requêtes

## Objectifs

Configurer une surchage du retry par défaut sur le service `ratings`.

## Application de la règle

### Raz de la configuration de l'application test (optionnel)

Appliquer le fichier [default-virtual-service.yml](./default/default-virtual-service.yml)

```bash
kubectl apply -f ./default/default-virtual-service.yml
```

### Mise en place de la règle

Appliquer le fichier [retry-virtual-service.yml](./retry-virtual-service.yml)

```bash
kubectl apply -f retry-virtual-service.yml
```

## En détail

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  namespace: poc-bbo
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3 #1#
      perTryTimeout: 2s #2#

```

1. Nombre de tentatives a effectuer
2. Délai du timeout de chaque nouvelle tentative.

Pour tester le retry nous pouvons appliquer le fichier [testCase](./testCase/test-retry-virtual-service.yml)

```bash
kubectl apply -f ./testCase/test-retry-virtual-service.yml
```

Analysons cette configuration:

```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  namespace: poc-bbo
spec:
  hosts:
    - ratings
  http:
    - fault:
        delay:
          percent: 100
          fixedDelay: 3s
      route:
        - destination:
            host: ratings
            subset: v1
      retries:
        attempts: 3
        perTryTimeout: 2s
```

Dans un premier temps nous fixons un délai arbitraire de réponse de 3 secondes sur le service,

ensuite nous appliquons le retry.

Dans ces circonstances, nous devrions avoir un temps de réponse d'environ 6 secondes du service. 3 tentatives espacées de 2 sc.

![testRetry](/assets/testRetry.png)

## Informations complementaires

Notons que c'est l'Envoy proxy du service appelé que nous configurons, c'est lui qui reçoit les requêtes et fait les appels au service `ratings` et qui gère donc quand faire les retry et dans quelles circonstances puis retourne une erreur ou un succès.
