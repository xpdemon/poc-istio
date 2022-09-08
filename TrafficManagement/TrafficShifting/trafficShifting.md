# Traffic Shifting

## Objectifs

1. Migrer progressivement le trafic d'une ancienne version d'un microservice vers une nouvelle .

## Application de la règle

### Raz de la configuration de l'application test (optionnel)

Router tous le traffic vers les `v1` de tous les microservices en appliquant le fichier [default-virtual-service.yml](./default/default-virtual-service.yml)

```bash
kubectl apply -f ./default/default-virtual-service.yml
```

### Mise en place du Traffic Shifting

Appliquer le fichier [virtual-service-review-50-v3.yml](./virtual-service-review-50-v3.yml)

```bash
kubectl apply -f virtual-service-review-50-v3.yml
```

## En détail

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: poc-bbo
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 50 #1#
        - destination:
            host: reviews
            subset: v3
          weight: 50 #2#
```

1. 50% du traffic total est dirigé vers le subset `v1`
2. 50% du traffic tatal est dirigé vers le subset `v2`

![trafficShifting](/assets/traffiShifting50.png)

Nous remarquons que le traffic est bien balancé a 50% vers `v1` et 50% vers `v2`

> Avec l'implémentation actuelle du side-car Envoy, vous devrez peut-être actualiser `/productpage`plusieurs fois (peut-être 15 ou plus)
> pour voir la bonne distribution. Vous pouvez modifier les règles pour acheminer 90 % du trafic vers `v3`pour voir les étoiles rouges plus souvent.

## Informations complementaires

La **weighted routing feature** (fonctionnalité de routage pondéré d'Istio) est très différente de la migration de version à l'aide des fonctionnalités de déploiement des plates-formes d'orchestration de conteneurs, qui utilisent la mise à l'échelle des instances pour gérer le trafic.

Avec Istio, vous pouvez autoriser les deux versions du `reviews`service à évoluer indépendamment, sans affecter la répartition du trafic entre elles.
