# API versionering i Kubernetes

En demo og intro til hvordan Kubernetes håndterer API'er og deres udvikling.

Skrevet og demo'et til et møde om tværgående kodedeling d. 22. februar 2022.

## Kubernetes API'et

Kubernetes består i sin essens af en central API-server som en stribe af
"controllers" benytter til at koordinere deres arbejde. API-serveren indeholde
Kubernetes state i form af en række ressource-instanser.

Alle resources i Kubernetes beskrives / manipuleres via API'et.

Kubernetes har fra dag ét haft et stort fokus på drifts-stabilitet og oppetid.
Det betyder at det skal kunne lade sig gøre at opgradere et live Kubernetes cluster
uden nogen nedetid. Da en opgradering kan betyde substantielle ændringer til
API'et har Kubernetes en lang række regler og features til at understøtte at
API-klienterne kan virker før, efter og under en opgradering. Heriblandt

* Understøttelse af flere samtidige versioner af ressourcer
* Regler for hvornår man må stoppe med at understøtte en version
* Regler for hvilket format en resource må gemmes i.

## API Organisering

[Reference](https://kubernetes.io/docs/reference/using-api/#api-groups)

For at gøre det lettere at administrere og udvikle de enkelte dele af Kubernetes
API'et er det opdelt i grupper der hver især kan versioneres og slås til/fra

En resources gruppe og version er en del af stien til resource der følger formen

```none
<API Group>/<version>/namespaces/<namespace>/<resource>/<name>
```

Eksempelvis vil stien til api-version `v1` af deploymenten (`deployments`) `nginx`
i namespacet `demo` have stien.

```none
apps/v1/namespaces/demo/deployments/nginx
```

## Demo: api'et

```shell
# Initial setup
$ kind create cluster

# (In another shell)
$ kubectl proxy

$ kubectl create namespace demo

```

```shell
# Create a Deployment
$ kubectl -n demo create deployment nginx --image=nginx --replicas=1

# Get it via kubectl
$ kubectl kubectl -n demo get deployment

# Or curl - requires "kubectl proxy" running in another shell
$ curl localhost:8001/apis/apps/v1/namespaces/demo/deployments/nginx

```

## API Versionering

[Reference](https://kubernetes.io/docs/reference/using-api/#api-versioning)

API-versionen af en resource type (et Kind) angives i `apiVersion` feltet

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo
...
```

Der findes 3 typer af API versioner der hver især har sin egen kontrakt (udpluk
fra dokumentationen herunder)

Alpha:

> * The support for a feature may be dropped at any time without notice.

Beta:

> * The support for a feature will not be dropped, though the details may change.
> * The schema and/or semantics of objects may change in incompatible ways in a subsequent beta or stable release

Stable:

> * The version name is vX where X is an integer.
> * The stable versions of features appear in released software for many subsequent versions.

API objekter gemmes i én version, men skal kunne serves i flere versioner.

## Demo: Multiple versioner

Vi opretter et gammelt Ingress, og ser hvad der sker. Ifølge dokumentationen er
version v1beta1 deprecated til fordel for v1 der blandt andet har [følgende ændring](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#ingress-v122):

> * The backend serviceName field is renamed to service.name

```shell

$ kubectl apply -n demo -f resources/ingress.yaml
Warning: networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
ingress.networking.k8s.io/my-ingress created

$ curl localhost:8001/apis/networking.k8s.io/v1beta1/ingresses

# Notice
...
                  "backend": {
                    "serviceName": "example-service",
                    "servicePort": 80
                  }
...

# Then

$ curl localhost:8001/apis/networking.k8s.io/v1/ingresses
...
                  "backend": {
                    "service": {
                      "name": "example-service",
                      "port": {
                        "number": 80
                      }
                    }
                  }
...

# Also, the default is v1:

$ kubectl get -n demo -o json ingress my-ingress

...
                  "backend": {
                      "service": {
                          "name": "example-service",
                          "port": {
                              "number": 80
                          }
                      }
                  }
...

```

## Deprecation

[Reference](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)

Der er en række regler for deprecation

> Rule #1: API elements may only be removed by incrementing the version of the API group.

Så, ændrer vi i API'et skal vi enten øge versionen, eg `v1alpha1` -> `v1alpha2`
eller stabiliteten `v1alpha2` -> `v1beta1`

> Rule #2: API objects must be able to round-trip between API versions in a
> given release without information loss, with the exception of whole REST
> resources that do not exist in some versions.

Kubernetes gemmer objektet i én version, ud fra denne skal vi kunne skabe
alle andre versioner. Fra "[The Kubernetes API
](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning)":

> API resources are distinguished by their API group, resource type, namespace (for namespaced resources), and name. The API server handles the conversion between API versions transparently: all the different versions are actually representations of the same persisted data. The API server may serve the same underlying data through multiple API versions.

Lad os sige vi indførte en ny resource type.

```yaml
apiVersion: reload.dk/v1alpha1
kind: Project
metadata:
  name: my-project
spec:
  size: large

```

Og derefter lavede en ny version

```yaml
apiVersion: reload.dk/v1alpha2
kind: Project
metadata:
  name: my-project
spec:
  size: large
  complexity: small

```

Så skal det kunne lade sig gøre at konvertere fra `v1alpha2` til `v1alpha1` og
omvendt, hvilket kan betyde at `v1alpha1` objekter nu begynder at se sådan her ud:

```yaml
apiVersion: reload.dk/v1alpha1
kind: Project
metadata:
  name: my-project
  annotations:
    complexity.project.reload.dk: small
spec:
  size: large

```

eller sågar

```yaml
apiVersion: reload.dk/v1alpha1
kind: Project
metadata:
  name: my-project
spec:
  size: large
  complexity: small

```

> Rule #3: An API version in a given track may not be deprecated in favor of a less stable API version.

Så vi må ikke deprecate `v1` før `v2` findes, også selvom vi har en  `v2beta1`

> Rule #4a: minimum API lifetime is determined by the API stability level
>
> * GA API versions may be marked as deprecated, but must not be removed within a major version of Kubernetes
> * Beta API versions must be supported for 9 months or 3 releases (whichever is longer) after deprecation
> * Alpha API versions may be removed in any release without prior deprecation notice

Dette sikrer at man kan stole på stabilitets-niveauerne.

> Rule #4b: The "preferred" API version and the "storage version" for a given group may not advance until after a release has been made that supports both the new version and the previous version

Eller med andre ord, du må ikke gemme objekter i en ny version, før du har haft
en release der version der gemte i det gamle format _og_ kunne udlæse i det nye.

Dette sikrer at du altid kan rulle tilbage til en gammel version

Se evt [denne video](https://youtu.be/_65Md2qar14?t=1198) for en gennemgang af
Regel 4 a+b

## Videre læsning og referencer

SIG-Architecture har en langt mere detaljeret vejledning til at lave [kompatible ændringer](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#on-compatibility)
blandt andet er der en længere række af regler

> An API change is considered compatible if it:
>
> * adds new functionality that is not required for correct behavior (e.g.,
> does not add a new required field)
> * does not change existing semantics, including:
>   * the semantic meaning of default values _and behavior_
>   * interpretation of existing API types, fields, and values
>   * which fields are required and which are not
>   * mutable fields do not become immutable
>   * valid values do not become invalid
>   * explicitly invalid values do not become valid
>
> Put another way:
>
> 1. Any API call (e.g. a structure POSTed to a REST endpoint) that succeeded
> before your change must succeed after your change.
> 2. Any API call that does not use your change must behave the same as it did
> before your change.
> 3. Any API call that uses your change must not cause problems (e.g. crash or
> degrade behavior) when issued against an API servers that do not include your
> change.
> 4. It must be possible to round-trip your change (convert to different API
> versions and back) with no loss of information.
> 5. Existing clients need not be aware of your change in order for them to
> continue to function as they did previously, even when your change is in use.
> 6. It must be possible to rollback to a previous version of API server that
> does not include your change and have no impact on API objects which do not use
> your change.  API objects that use your change will be impacted in case of a
> rollback.

Efterfulgt af en række eksempler.

Mumshad Mannambeth (instruktør på et af de mere populære CKA/CKAD kurser) har
har lavet en [rigtig fin video](https://www.youtube.com/watch?v=_65Md2qar14)
der gennemgår størstedelen af det jeg har dækket herover.

Jeg har desuden brugt følgende artikler

* [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-changes),
  den officielle "indgangs-artikel".
* [Deprecated API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/),
  den officielle liste af hvad der er deprecated og hvordan man skal migrere manuelt.
* [Kubernetes API Basics - Resources, Kinds, and Objects](https://iximiuz.com/en/posts/kubernetes-api-structure-and-terminology/)
  en stribe gode eksempler og tegninger der forklarer de basale dele af API'et.

Hvis man vil gå endnu dybere

* [Versions in CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)
 relevant når man selv skal til at lave objekt-typer.
