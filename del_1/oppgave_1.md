# Oppgave 1

## Sett opp kubectl

Først skal vi installere det vi trenger for å komme igang.

```bash
brew install kubectl
aws eks update-kubeconfig --name Oppdrift --region eu-west-1
```

## Pods, Deployments, Services og manifester

Først av alt skal vi se på de viktigste tingene for å definere en tjeneste i Kubernetes.
Vi skal se på pods, deployments og services. I tillegg tar vi en titt på manifest-filer som vi bruker for å holde styr på det hele.

### Pods

Den minste "enheten" vi har er pods. La oss se på en enkel definisjon av en pod og samtidig bli introdusert til manifester.

- Bruk mappen `arbeidsmappa`, som ligger i rot på dette prosjektet.
- Lag en fil `pod.yaml` i denne mappen. Lim inn innholder under.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <DITT-NAVN>-mypod
  labels:
    name: <DITT-NAVN>-mypod
spec:
  containers:
    - name: <DITT-NAVN>-mypod
      image: nginx
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "250m"
```

- kjør `kubectl apply -f pod.yaml` for å lage pod-en.
- Sjekk status på pod-en din med `kubectl get pods`.
- For å se litt ekstra detaljer om denne kan du kjøre `kubectl describe pod mypod`.
- For å slette pod-en kan du kjøre `kubectl delete pod mypod`.
- Prøv å fjerne `resources`-delen av pod-en, re-apply den og se hva som skjer. Du kan bruke `describe`-kommandoen for å se litt ekstra detaljer.

## Deployments

Neste er Deployments, dette er en måte å definere en ønsket tilstand av pods. La oss se på en enkel definisjon av en deployment og bygge videre på det.

_Notat: Ser du noen skrive om ReplicaSets, så er faset ut av Kubernetes, og erstattet med Deployments._

- Lag en fil `deployment.yaml` i arbeidsmappen din. Lim inn innholdet under.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: <DITT-NAVN>-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: <DITT-NAVN>-nginx
          image: nginx
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "150m"
            limits:
              memory: "128Mi"
              cpu: "150m"
```

- Kjør `kubectl apply -f deployment.yaml` for å lage deploymenten.

- Sjekk status på deploymenten din med `kubectl get deployments`.
- Kjør en `kubectl get pods` for å se at det er tre pods som er opprettet.
- Prøv å slette en av de med `kubectl delete pod <pod-navn>`. Hva skjer med pod-en og deploymenten?
- Prøv å slette deploymenten med `kubectl delete deployment nginx-deployment`. Hva skjer med pod-ene?

## Sjekke outputen fra pod-ene

La oss ta en titt på hva vi egentlig har kjørt opp med dette statiske nginx-imaget. Først tar vi en titt på loggene. Velg en av pod-ene og kjør `kubectl logs <pod-navn>`.

_Fancy triks: Du kan også bruke `kubectl logs -f <pod-navn>` for å følge med på loggene live (ikke så relevant akkurat nå da)._

La oss se om denne nettsiden gir oss noe. Vi kan bruke `kubectl port-forward <pod-navn> 8080:80` for å videresende port `80` på pod-en til port `8080` på maskinen vår. Da kan vi åpne http://localhost:8080 i nettleseren og se om vi får noe.

## Services

Kubernetes services er en måte å definere pods som en tjeneste på. På den måten kan en pod koble til en annen pod uten at den trenger å vite noe annet enn tjenestenavnet dens! Det er mange teknikker for hvordan vi jobber med services og gjør networking innad i clusteret, her bruker vi ClusterIPs. Det er en måte å definere en statisk IP-adresse som kan nås fra andre pods eller fra utsiden av Kubernetes-clusteret. La oss se på en enkel definisjon av en service og bygge videre på det etterpå.

- Lag en fil `service.yaml` i arbeidsmappen din. Fyll inn port og selector-feltet, hent inspirasjon fra deploymenten og dokumentasjonen:
- API reference for `service`: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#service-v1-core.
  API reference for selector:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ...
spec:
  selector:
    app: ...
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply services, gjør en `kubectl get` mot den og finn IPen den har fått tildelt. Dette er IP-en gjelder bare for nettverket inni clusteret! Vi kan altså ikke nå den fra vår maskin med mindre vi eksponerer den, vi kan bare koble til den fra inni clusteret. La oss se hvordan den oppfører seg fra innsiden av clusteret.

Først av alt, la oss se hva vi får hvis vi kjører en `curl` mot nginx hvis vi starter det helt blankt som vi har gjort her. Vi testet det med nettleser i stad.

Enten sett opp en port-forward og gjør en `curl http://localhost:<min-port>` og se hva som skjer. Eller kjør `docker run -p 8080:80 nginx` i en terminal og gjør deretter en `curl http://localhost:8080` i en annen terminal for å kjøre et image lokalt og teste via det.

Sånn! Nå vet du forventet output!

For å kjøre dette fra inne i clusteret kan vi starte en kortlevd pod med det vi trenger, og stoppe den etterpå.

Hva gjør vi under her? Prøv å tenk tilbake på tidligere workshops og eventuell google for å forstå hva kommandoen er før du kjører den. `kubectl run -i --tty --rm debug --image=curlimages/curl --restart=Never -- http://10.104.128.227` (bytt ut IP-en med den fra pod-en din).
