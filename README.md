# Instalare și Configurare Istio - modalitatea simpla

## 1. Instalare Istio

Pentru documentația completă despre instalarea Istio, vizitați: [Getting Started with Istio](https://istio.io/latest/docs/setup/getting-started/#download)

## 2. Comenzi pentru Instalare

1. Descărcați și instalați Istio:

    ```bash
    curl -L https://istio.io/downloadIstio | sh -
    cd istio-1.23.3
    export PATH=$PWD/bin:$PATH
    ```

2. Verificați instalarea Istio folosind comanda `istioctl`:

    ```bash
    istioctl version
    ```

3. Pentru a instala Istio în clusterul Kubernetes:

    ```bash
    istioctl install
    ```

   - Verificare instalare namespace Istio:

     ```bash
     kubectl get ns
     ```

## 3. Configurare pentru Namespace

Pentru a activa injectarea automată a sidecar-ului Istio (Envoy proxy) într-un namespace specific, folosiți următoarea comandă:

        kubectl label namespace <nume-namespace> istio-injection=enabled

Si vom primi raspunsul `namespace/istio-practice-app labeled`

După aplicarea label-ului, este necesar să ștergeți și să re-aplicați manifestul aplicației pentru a permite injectarea sidecar-urilor. Comenzile sunt:

```bash
kubectl delete -f mainfest-ms-app-istio.yaml
kubectl apply -f mainfest-ms-app-istio.yaml
```

În urma aplicării configurației, vom observa că un pod are două containere, confirmând că sidecar-ul Envoy proxy a fost adăugat. Exemplu de rezultat:

        pod/cartservice-77549b5fd5-4pfmb             2/2     Running            0             39s

Pentru a vedea detalii despre pod și containerul `istio-proxy` adăugat, folosiți comanda:

        kubectl describe pod/cartservice-77549b5fd5-4pfmb -n istio-practice-app

Exemplu de output:

```
Normal   Created    6m6s   kubelet            Created container istio-proxy
Normal   Started    6m6s   kubelet            Started container istio-proxy
```

## 4. Vizualizare și Monitorizare

Istio oferă mai multe opțiuni pentru vizualizarea și monitorizarea traficului și a performanței aplicației (Kiali, Grafana, Prometheus, Jaeger), prin intermediul unor servicii.

 - Aplicați manifestul pentru addon-urile Istio (modificați calea în funcție de locația instalării):

        kubectl apply -f /home/ggheorghe/Desktop/projects/istio-1.23.3/samples/addons

 - Verificați statusul pod-urilor și serviciilor din namespace-ul istio-system:

        kubectl get pod -n istio-system
        kubectl get svc -n istio-system

 - Exemplu de rezultat pentru serviciul Kiali:

        kiali                  ClusterIP      10.101.89.89     <none>        20001/TCP,9090/TCP

 - Pentru a accesa Kiali (interfața web de monitorizare), folosiți:

        kubectl port-forward svc/kiali -n istio-system 20001

Accesați apoi: `127.0.0.1:20001` pentru a deschide interfața Kiali în browser:

---

# Ce este VirtualService și DestinationRule?

## Pe scurt

- **VirtualService**:
  - **Rol**: Controlează rutarea traficului către diferitele versiuni (sau subseturi) ale unui serviciu, pe baza unor criterii specifice (de exemplu, anteturi, parametri de query sau alte condiții de potrivire).
  - **Nivel de vizibilitate**: VirtualService operează la nivel de serviciu și definește reguli pentru traficul direcționat către un anumit serviciu din cluster.
  - **Exemplu de utilizare**: Rutarea către o versiune specifică (ex: `v1`, `v2`) a unui serviciu pe baza unui antet HTTP, sau rutare canary pentru testarea unei versiuni noi cu un procent redus din trafic.

- **DestinationRule** (decide cum se comportă conexiunea odată ajunsă la destinație):
  - **Rol**: Este folosit pentru a crea 'subset-uri' (versiuni, certificate mTLS, etc.)
  - **Nivel de vizibilitate**: DestinationRule operează la nivel de destinație și se aplică instanțelor unui serviciu (pod-uri), permițând configurarea de politici avansate.
  - **Exemplu de utilizare**: Definirea subseturilor `v1`, `v2` ale unui serviciu și configurarea conexiunilor TLS pentru a asigura securitatea între microservicii.

---

## Exemple

### 1. VirtualService - Pot exista exemple mai simpliste, acesta continand ConsistenHashing - o filtrare mai detaliata a Traficului 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: example-virtualservice
spec:
  hosts:
    - "example.com" # Numele serviciului care conține pod-urile/subseturile la care vrem să trimitem traficul primit în acest VirtualService
                    # => poate necesita numele complet de DNS: "<nume-svc>.<nume-ns>.svc.cluster.local"
  http:
    - match:
        - headers:
            x-request-id:
              exact: "some-value"
      route:
        - destination:
            host: app-backend-service
            subset: v1 # Către ce subset se rutează traficul (acesta fiind caracterizat în `DestinationRule`)
            port:
              number: 8080
          weight: 100
    - route:
        - destination:
            host: app-backend-service
            subset: v2
            port:
              number: 8080
          weight: 100
```

In acest exemplu:

 - Traficul care are un antet `x-request-id` cu valoarea "some-value" este direcționat către instanțele din subsetul v1 al serviciului app-backend-service.
 - Restul cererilor sunt procesate în mod normal, dar consistent hashing se va aplica astfel încât aceleași valori ale antetului vor duce la aceleași instanțe de serviciu.

**! Important:** Unde avem `route.subset: <nume-subset>` -> acolo ne spune catre ce subset se ruteaza traficul (acesta fiind caracterizat in `DestinationRules` )

### 2. DestinationRules

 - Definește subseturile (versiunile v1 și v2) pentru app-service, aplică un algoritm de load balancing (`least_conn`) și configurează conexiuni sigure folosind TLS

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: app-service-destinationrule
spec:
  host: app-service.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # Balansare pe baza conexiunilor active (least connections)
    tls:
      mode: ISTIO_MUTUAL  # Folosește TLS mutual pentru securizarea conexiunii
  subsets: # Mai jos sunt declarate 2 subset-uri, unde sunt caracterizate cum vrem noi
    - name: v1 # Numele pe care il va avea pod-ul selectat in baza label-ului
      labels:
        version: v1 # Va selecta pod-ul in baza acestui label
    - name: v2
      labels:
        version: v2

```

**Explicatie:**
 - host: Specifică serviciul app-service.
 - trafficPolicy.loadBalancer: Configurat pentru least_conn, adică trafic distribuit uniform pe baza numărului de conexiuni active.
 - trafficPolicy.tls: Activează TLS mutual (ISTIO_MUTUAL), securizând conexiunea între microservicii.
 - subsets: Definește subseturile v1 și v2 folosind etichete de versiune (version: v1 și version: v2).

 # Ce este un Gateway ?

 *From Istio Doc*:
 - Gateway describes a load balancer operating at the edge of the mesh receiving incoming or outgoing HTTP/TCP connections.

The specification describes:
1. a set of ports that should be exposed
2. the type of protocol to use 
3. SNI configuration for the load balancer

*Doc Finishs Here*

 - Un Istio Gateway este folosit pentru a controla modul în care traficul extern intră într-un mesh Istio. 
 - Gateway-urile permit definirea de reguli  (cum ar fi host-urile și porturile pe care traficul este acceptat, și folosesc TLS pentru securizare). 
 - Ele funcționează similar cu Ingress din Kubernetes, dar oferă un control mai detaliat.

 ### Exemplu simplu de configurare Gateway:

 ```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: example-gateway
spec:
  selector:
    istio: ingressgateway  # Specifică folosirea unui pod Ingress Gateway Istio
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "example.com"
      tls:
        mode: SIMPLE
        credentialName: "example-credential"  # Secret Kubernetes cu certificatul TLS

 ```

 Explicația configurării

- selector: Specifică că acest Gateway va fi aplicat la podurile cu eticheta istio: ingressgateway. Acest selector folosește de obicei un Ingress Gateway definit de Istio pentru a gestiona traficul de intrare.

    - servers: Definește o listă de servere (porturi și protocoale) prin care traficul poate intra în mesh. În acest caz:
        - port:
            - number: 443 - portul pe care va fi acceptat traficul.
            - protocol: HTTPS - indică faptul că traficul este securizat.
        - hosts: Specifică domeniul care va accepta conexiuni (în acest caz, `example.com`).
        - tls:
            - mode: SIMPLE - modul TLS simplu, folosit pentru a securiza conexiunea fără a solicita autentificare din partea clientului.
            - credentialName: Numele secretului Kubernetes care conține certificatul TLS și cheia privată (example-credential).

### Cum se leagă Gateway de un VirtualService ?

In declararea `VirtualService` avem:

```yaml
spec:
  hosts:
    - "example.com"
  gateways:
    - example-gateway  # Leagă acest VirtualService de Gateway-ul nostru
```