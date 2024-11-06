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