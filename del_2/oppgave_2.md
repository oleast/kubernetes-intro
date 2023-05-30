# Oppgave 2

## Eksponere tjenester vår, service-typer og ingress
Det finnes forskjellige typer services i Kubernetes. De vanligste er ClusterIP, NodePort og LoadBalancer. Vi har prøvd oss på den første og skal prøve oss på den andre. 

Prøv å endre `spec.type` og sett den til `NodePort`. Reapply manifestet. ta en titt på services med `kubectl get service <SERVICEN MIN> -o wide` og finn hvilken port du har fått. 

Denne porten treffer nå servicen din uavhengig av hvilken node du treffer. Gjør en `kubectl get nodes -owide`. Velg intern-IPen til en av nodene og kjør http://<IP>:<PORT> fra innsiden av clusteret for å sjekke at det funker.

### Ingress
Neste er ingress. Ingress er et objekt i clusteret for å rute extern http(s) trafikk til en service. Det er en måte å eksponere tjenester på utenfor clusteret. 
La oss sette opp en ingress mot servicen vår. Her skal vi i tillegg utnytte oss av at for managed kubernetes kan vi ofte få platformen til å lage en LoadBalancer for oss. Fyll ut det under. 
```yaml
---
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <NAVN>-nginx-ingress
  annotations:     
    alb.ingress.kubernetes.io/load-balancer-name: ingress
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  ingressClassName: alb
  rules:
    - host: <navn>-nginx.bekk.cloud
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <NAVN>-nginx-service
                port:
                  number: <PORTEN FRA SERVICEN>
```

- Apply og describe. Gå hit og finn LBen din: https://eu-west-1.console.aws.amazon.com/ec2/home?region=eu-west-1#LoadBalancers:. 
- Gå til route 53 og sett opp en A alias record mot LBen din. Kall den den samme (altså <navn>-nginx.bekk.cloud).
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