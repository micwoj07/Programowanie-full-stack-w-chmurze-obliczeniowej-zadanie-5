# Programowanie-full-stack-w-chmurze-obliczeniowej-zadanie-5

###############################################################
LABORATORIUM 5
Autor: Michał Wójtowicz
Grupa: 2.4
###############################################################

Instrukcja uruchomienia krok po kroku

--Start klastra Minikube--

minikube start --cpus=2 --memory=2048
minikube status

--Włączenie serwera metryk--

minikube addons enable metrics-server
kubectl get pods -n kube-system -l k8s-app=metrics-server

Wynik:
metrics-server-85b7d694d7-qv4jv   1/1   Running   0   5s

--Utworzenie Namespace--

kubectl create namespace ns-dev
kubectl create namespace ns-prod
kubectl get namespaces

Wynik (fragment):
ns-dev   Active
ns-prod  Active

--Wgranie polityk zasobów--

kubectl apply -f resourcequota-dev.yaml
kubectl apply -f resourcequota-prod.yaml
kubectl apply -f limitrange.yaml

--Weryfikacja Quota i LimitRange--

kubectl describe resourcequota xlimits -n ns-dev
kubectl describe resourcequota xlimits-prod -n ns-prod
kubectl describe limitrange dev-limits -n ns-dev

---Quota ns-dev---
Name: xlimits
Resource  Used  Hard
cpu       0     200m
memory    0     256Mi
pods      0     10

---Quota ns-prod---
Name: xlimits-prod
Resource  Used  Hard
cpu       0     400m
memory    0     512Mi
pods      0     20

---LimitRange ns-dev---
Type: Container
Max CPU: 200m
Max RAM: 256Mi
Default Request: CPU=100m, RAM=10Mi
Default Limit:   CPU=200m, RAM=20Mi

--Test poprawnie działającego deploymentu yes-test--

kubectl apply -f yes-test.yaml
kubectl get pods -n ns-dev -l app=yes-test --show-labels
kubectl top pods -n ns-dev -l app=yes-test

Mamy:
yes-test-5cfd5f56bb-5d6nt   1/1 Running  app=yes-test
yes-test-5cfd5f56bb-jwvch   1/1 Running  app=yes-test

Zużycie (fragment):
CPU 2m, 3m
MEM 16Mi, 16Mi

Wniosek:
Deployment yes-test działa poprawnie i mieści się w narzuconych limitach.

--Test deploymentu zero-test--

kubectl apply -f zero-test.yaml
kubectl get pods -n ns-dev -l app=zero-test --show-labels
kubectl describe pod -n ns-dev -l app=zero-test | sed -n '/Limits:/,/Requests:/p'

(fragment):
Limits:
  cpu:    200m
  memory: 20Mi
Requests:
  cpu:    100m
  memory: 10Mi

Wniosek:
Deployment zero-test nie miał deklaracji resources w YAML, ale Pod uruchomił się poprawnie i otrzymał limity zgodne z LimitRange dev-limits.

--Test deploymentu no-test (przekroczenie limitów – nie działa poprawnie)--

kubectl apply -f no-test.yaml
kubectl get pods -n ns-dev -l app=no-test

Wynik:
No resources found in ns-dev namespace.

kubectl describe deployment no-test -n ns-dev

(fragment):
Replicas: 3 desired | 0 available | 3 unavailable
NewReplicaSet: (0/3 replicas created)
ReplicaFailure: True, FailedCreate

Wniosek:
Deployment  no-test nie utworzył żadnego Poda, ponieważ zadeklarowane w YAML żądania i limity przekraczały dozwolone zasoby (LimitRange: 0.2 CPU / 256Mi RAM oraz Quota Namespace).

Podsumowanie

- Utworzono namespace ns-dev i ns-prod.
- Zasoby w ns-prod są 2× większe niż w ns-dev (CPU 400m i RAM 512Mi).
- Deployment yes-test działa poprawnie.
- Deployment zero-test działa poprawnie mimo braku resources w YAML i przejął limity z LimitRange.
- Deployment no-test nie działa, co potwierdza poprawne działanie mechanizmów ograniczania zasobów i spełnienie punktów 2–4 zadania.

