# Oppgave 2

## Eksponere tjenesten vår, service-typer og gateways

Det finnes forskjellige typer services i Kubernetes. De vanligste er ClusterIP, NodePort og LoadBalancer. Vi har prøvd oss på den første, og skal nå prøve oss på den andre.

Prøv å endre `spec.type` ved å sette den til `NodePort`. Re-apply manifestet. Ta en titt på services med `kubectl get service <SERVICEN MIN> -o wide`, og finn hvilken port du har fått.

Denne porten treffer nå servicen din uavhengig av hvilken node du treffer. Gjør en `kubectl get nodes -owide`. Velg intern-IP-en til en av nodene og kjør http://<IP>:<PORT> fra innsiden av clusteret for å sjekke at det funker.

### Gateway

Neste er gateway. Gateway er et objekt i clusteret for å rute extern http(s) trafikk til en service. Det er en måte å eksponere tjenester på utenfor clusteret.

La oss sette opp en gateway mot servicen vår. Her skal vi i tillegg utnytte det vi får fra _Managed Kubernetes_. Her kan vi ofte få platformen til å lage en LoadBalancer for oss, en funksjonalitet vi måtte laget selv om vi hostet Kubernetes på egne maskiner.

Fyll ut det under.

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: <NAVN>-nginx-gateway
spec:
  gatewayClassName: gke-l7-gxlb
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: "<NAVN>-nginx.google.oppdrift.cloud"
      allowedRoutes:
        namespaces:
          from: Same

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <NAVN>-nginx-httproute
spec:
  parentRefs:
    - name: <NAVN>-nginx-gateway
      sectionName: http
  hostnames:
    - "<NAVN>-nginx.google.oppdrift.cloud"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: "/"
      backendRefs:
        - name: <NAVN>-nginx-service
          port: <PORTEN FRA SERVICEN>
```

- Apply og describe. Gå hit og finn LoadBalanceren (LB-en) din: https://console.cloud.google.com/net-services/loadbalancing/list/loadBalancers?project=bekk-oppdrift:
(Huske å bytte bruker oppe i høyre hjørne om den viser at du ikke har tilgang)
- Gå til Cloud DNS og sett opp en A alias record mot LB-en din. Kall den det samme (altså <navn>-nginx.google.oppdrift.cloud). Gå til CloudDNS -> oppdrift -> "Add standard". Legg inn domenet i DNS-name, og IP-adressen.
- Sjekk om det funker! Gi den et lite minutt på å progagere DNS.

## Bygge ut deploymenten

La oss spe på komponentene våre litt. Både for å gjøre de litt mer realistiske men også for å utforske hva vi kan gjøre.

### Strategi

Først, la oss se hva som skjer når vi ruller ut en endring ting deploymenten vår. Ha et terminalvindu åpent og følg med ved å kjøre `kubectl get pods -w` i det. Endre deretter imaget til `nginx:1.24.0-alpine` og kjør en kubectl apply -f. Du kan også gjøre en `kubectl describe` på servicen. Hva skjer? Hvorfor?

- Prøv å endre strategien til `Recreate`. https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#deploymentspec-v1-apps. Når du har endret strategien, rull ut en ny endring og se hva som skjer. Hva er forskjellen på `RollingUpdate` og `Recreate`? Når ønsker vi den ene fremfor den andre?

## Sidecar containers

Next up, sidecars. En sidecar er en container som kjører i samme pod som en annen container. Det er et ganske kult konsept, da du kan dele opp og gjenbruke små komponenter på tvers. Typiske ting man kjører som sidecars er containere for å håndtere logging, monitorering, e.l.

La oss lage og sette opp et eksempel. Før du gjør en apply på denne, les gjennom og prøv å forstå hva den gjør først. Sjekk hvordan den funker ved å bruke `kubectl logs <POD> -c <CONTAINER>`. Se som vanlig etter felter du må fylle ut.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <FYLL-MEG-UT>-sidecar-example
spec:
  volumes:
    - name: log
      emptyDir: {}
  containers:
    - image: busybox
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "100m"
      name: application
      args:
        - /bin/sh
        - -c
        - >
          while true; do
            echo "$(date) INFO hello" >> /var/log/myapp.log ;
            sleep 1;
          done
      volumeMounts:
        - name: log
          mountPath: /var/log
    - name: sidecar
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "100m"
      image: busybox
      args:
        - /bin/sh
        - -c
        - tail -f /var/log/myapp.log
      volumeMounts:
        - name: log
          mountPath: /var/log
```
