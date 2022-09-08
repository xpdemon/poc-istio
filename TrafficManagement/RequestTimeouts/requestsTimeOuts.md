# Request Timeout

## Contexte

Un timeout HTTP peut être spécifié à l'aide du champ `timeout` de la règle de routage . Par défaut, le délai du timeout est désactivé.

## Objectifs

Pour configurer les Timeout des requêtes dans Envoy nous allons remplacer le timeout du service `reviews` à 1 seconde. 

> Pour voir son effet, cependant, nous introduirons également un délai artificiel de 2 secondes dans les appels au service `ratings`.

## Application de la règle

### Raz de la configuration de l'application test (optionnel)

Appliquer le fichier [default-virtual-service.yml](./default/default-virtual-service.yml)

```bash
kubectl apply -f ./default/default-virtual-service.yml
```

### Mise en place du timeout

Appliquer le fichier [reviewTimeout](./review-timeOut-virtuals-services.yml)

```bash
kubectl apply -f review-timeOut-virtuals-services.yml
```

Appliquer le fichier [ratingDelay](./rating-delay-virtual-service.yml)

```bash
kubectl apply -f rating-delay-virtual-service.yml
```

> Ce dernier fichier de configuration vient ajouter un délai de réponse artificiel au service Review afin de pouvoir constater
> que le timeout fonctionne bien, pour plus d'informations voir la section [FaultInjection](/TrafficManagement/FaultInjection/faultInjection.md)

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
    - route:
        - destination:
            host: reviews
            subset: v2
      timeout: 0.5s #1#
```

1. Dans la section `route` nous rajoutons un `timeout` de 0.5 seconde.

![timout](/assets/timeout.png)

Nous voyons que la page prend environ 1 seconde a s'afficher et que les avis ne sont pas disponibles.

> La raison pour laquelle la réponse prend 1 seconde, même si le délai d'attente est configuré à une demi-seconde,
>
> est qu'il y a une nouvelle tentative codée en dur dans le `productpage`service, il appelle donc le service de temporisation `reviews`deux fois avant de revenir.

Une autre chose à noter à propos des timeout dans Istio est qu'en plus de les remplacer dans les règles de routage, comme nous l'avons fait. Ils peuvent également être remplacés à la demande si l'application ajoute un header `x-envoy-upstream-rq-timeout-ms` sur les requêtes sortantes. Dans l'header, le délai d'attente est spécifié en millisecondes au lieu de secondes.

sources :

[istio.io](https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/)
