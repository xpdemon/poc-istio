# Traffic Mirroring

## Objectifs

Créer une regle de Shadow Traffic vers un subset préçis .

La mise en miroir envoie une copie du trafic en direct à un service mis en miroir. Le trafic mis en miroir se produit en dehors circuit critique pour le service principal, a des fins de tests de charge à volumetrie Réel ou d'analyse sans impacter l'utilisateur final .

## Application de la règle

### Raz de la configuration de l'application test (optionnel)

Appliquer le fichier [default-init-book-info.yml](./default/default-init-book-info.yml)

```bash
kubectl apply -f ./default/default-init-book-info.yml
```

### Mise en place de la règle

Appliquer le fichier de configuration [mirroring-virtual-service.yml](./mirroring-virtual-service.yml)

```bash
kubectl apply -f mirroring-virtual-service.yml
```

## En détail

```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  namespace: poc-bbo
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: bbo
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v1
      mirror: #1#
         host: reviews #2#
         subset: v2 #3#
      mirrorPercentage: #4#
        value: 100
```

1. La section `mirror` ouvre la configuration pour le mirroring
2. `host` de destination du shadow traffic
3. `subset` de destination du shadow traffic
4. % du traffic à mirroré

[HTTPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute)

### Tester

Rendons-nous sur la page web de l'application `http://$<istio-ingerssgateway.ExternalIP>/productpage`.

Nous remarquons que tout est normal le cycle nominal est respecté.

Mais nous remarquons aussi sur les graph que lorsque nous ne sommes pas identifié avec l'utilisateur `bbo` du traffic est mirroré vers le service `reviews:V2`

![mirroringOn](/assets/mirroringOn.png)

Si nous nous identifions avec l'utilisateur `bbo` le shadowing stop comme définit dans les configurations

![mirroringOff](/assets/mirroringOff.png)

### Point de fonctionnement

Le proxy Envoy côté client  `productpage`  enverra des requêtes à la fois à `reviews:V1`(circuit nominal) et à `reviews:V2`(circuit en miroir). `v1`et  `v2` sont deux sous-ensembles du service Kubernetes `reviews`, spécifiés dans une Istio DestinationRule.

`productpage` n'attendra pas de réponses de `reviews:V2` le mirroring effectue des requêtes en mode fire and forget (tir et oublie) .


![mirroring](/assets/mirroring.png)
