# Requests Routing

## Objectifs

1. L'objectif initial de cette tâche est d'appliquer des règles qui acheminent tout le trafic vers la  `v1`(version 1) des microservices.
2. Ensuite nous redirigeons le traffic pour un utilisateur particulier vers la `v2` et le reste vers la `v1`.

## Application de la règle de routage globale

Appliquer le fichier [redirect-virtual-services.yml](./TrafficManagement/RequestRouting/redirect-virtuals-services.yml)

```bash
kubectl apply -f redirect-virtual-services.yml
```

### En détail

```yaml
apiVersion: networking.istio.io/v1alpha3 #1#
kind: VirtualService #2#
metadata:
  name: reviews
  namespace: poc-bbo
spec:
  hosts: #3#
    - reviews
  http: #4#
    - route:
        - destination: #5#
            host: reviews
            subset: v1
```

1. Nous utilisons l'API networking d'Istio
2. Ce changement vient modifier un service virtuel
3. Il s'agit de la liste d'hôtes du service virtuel, où l'adresse ou les adresses que le client utilise pour envoyer des requêtes au service. Il peut s'agir d'une adresse IP, d'un nom DNS ou, selon la plate-forme, d'un nom abrégé (tel qu'un nom abrégé de service Kubernetes) qui se résout, implicitement ou explicitement, en un nom de domaine complet .

   Les préfixes génériques ("*") peuvent également être utilisés, ce qui nous permet de créer un ensemble unique de règles de routage pour tous les services correspondants. Les hôtes de services virtuels ne doivent pas nécessairement faire partie du registre de services Istio, ce sont simplement des destinations virtuelles. Cela vous permet de modéliser le trafic pour les hôtes virtuels qui n'ont pas d'entrées routables à l'intérieur du maillage.
4. La partie `http` est appelée règles de routage. Cette section décrit les conditions de correspondance et les actions à entreprendre pour le routage `HTTP/1.1`ou `HTTP2` /et le routage du trafic `gRPC`.
5. Cette section est l'endroit où la destination réelle est définie. Dans ce cas, s'il s'exécute sur Kubernetes et le nom d'hôte est un nom de service Kubernetes.

> *L'utilisation de noms courts comme celui-ci ne fonctionne que si les hôtes de destination et le service virtuel se trouvent réellement dans le même namespace Kubernetes. Étant donné que l'utilisation du nom court Kubernetes peut entraîner des erreurs de configuration, nous vous recommandons de spécifier des noms d'hôte complets dans les environnements de production.*

![requestRouting](/assets/request-routing.png "Le traffic est redirigé uniquement vers les V1")

## Application de la règle de routage par User

Appliquer le fichier [redirec-virtuals-services-by-user.yml](./TrafficManagement/RequestRouting/redirec-virtuals-services-by-user.yml)

```bash
 kubectl apply -f redirec-virtuals-services-by-user.yml`
```

### En detail

Cet exemple est acceptable par le fait que le service `productpage` ajoute un header personnalisé `end-user` à toutes les requêtes HTTP sortantes vers le service `reviews`.

```yaml
# redirection d'un user ver la v2 de reviews
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: poc-bbo
spec:
  hosts:
    - reviews
  http:
    - match: #1#
        - headers:
            end-user:
              exact: bbo
      route:
        - destination:
            host: reviews
            subset: v2 #2#
    - route:
        - destination:
            host: reviews
            subset: v1 #3#
```

1. Dans la partie après la section `http`. Comme dans le `virtualservice` ci-dessus , vous pouvez spécifier un utilisateur particulier. Ainsi, la règle de routage s'appliquera uniquement à l'utilisateur défini dans la configuration.
2. Redirection du traffic pour l'utilisateur vers le subset `v2`
3. Pour tous les autres utilisateurs redirection ver le sunbset `v1`

![userRequestRouting](/assets/userRequestRouting.png)

Après application de la règle quand nous nous authentifions sur l'application avec l'utilisateur le traffic est routé vers le subset`v2` sinon vers le `v1`

sources :

* [faun.pub](https://faun.pub/istio-step-by-step-part-13-istio-traffic-management-istio-core-features-e513bfd66fb4)
* [istio.io](https://istio.io/latest/docs/tasks/traffic-management/request-routing/)
