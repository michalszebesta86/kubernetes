# Postup nasazení aplikace do lokálního Kubernetes (kind)

Návod krok za krokem: .NET API, Docker image, lokální cluster kind a nasazení přes `kubectl`. Na konci je přehled dalšího kroku (Argo CD).

## Jednorázová příprava

Potřebné nástroje: .NET SDK, Docker Desktop, `kubectl`, `kind`.

```powershell
winget install Kubernetes.kind
winget install Kubernetes.kubectl
```

`Kubernetes.kind` a `Kubernetes.kubectl` jsou identifikátory balíčků v nástroji `winget` (správce balíčků ve Windows). Každý z nich nainstaluje jiný program:

| Balíček | Program | K čemu slouží |
|---|---|---|
| `Kubernetes.kind` | `kind` (Kubernetes IN Docker) | **Vytváří a spravuje lokální cluster.** Spustí Docker kontejner, který se tváří jako celý Kubernetes node. Používá se hlavně k vývoji a testování na vlastním počítači. Typické příkazy: `kind create cluster`, `kind delete cluster`, `kind load docker-image`, `kind get clusters`. |
| `Kubernetes.kubectl` | `kubectl` | **Ovládací nástroj (CLI) pro práci s clusterem.** Komunikuje s API serverem clusteru: nasazuje aplikace, vypisuje pody, čte logy, škáluje, maže. Funguje s jakýmkoli clusterem (kind, Docker Desktop, Azure AKS apod.). Typické příkazy: `kubectl apply`, `kubectl get pods`, `kubectl logs`, `kubectl port-forward`. |

Jednoduše: **`kind` postaví cluster, `kubectl` ho ovládá.** `kind` bez `kubectl` by cluster vytvořil, ale nic bys v něm nenasadil, a `kubectl` bez clusteru nemá s čím mluvit. Po vytvoření clusteru kind sám nastaví `kubectl` kontext `kind-demo`, takže `kubectl` ví, na který cluster se připojit.

Po instalaci otevři nový terminál a ověř:

```powershell
dotnet --list-sdks
docker --version
kind --version
kubectl version --client
```

### Vytvoření clusteru (jednou)

Aby se ti po restartu počítače nezměnil port API serveru (Windows ho může zablokovat), použij konfiguraci s pevným portem. Soubor `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
```

**Kam soubor uložit:** na umístění nezáleží, doporučené je kořen projektu `C:\Temp\k8s-demo`, vedle složek `DemoApi` a `k8s`:

```
C:\Temp\k8s-demo\
├── kind-config.yaml
├── DemoApi\
└── k8s\
```

Příkaz `kind create cluster` pak spouštěj ze stejné složky (`cd C:\Temp\k8s-demo`), protože cesta v `--config` je relativní k aktuální složce terminálu. Pokud soubor uložíš jinde, použij plnou cestu (`--config C:\cesta\kind-config.yaml`).

Konfigurace se použije **jen při vytvoření clusteru**. Už běžící cluster ji nezíská, je nutné ho smazat (`kind delete cluster --name demo`) a vytvořit znovu. Potom je potřeba znovu provést `kind load docker-image` a `kubectl apply`.

Před volbou portu ověř, že není v rezervovaném rozsahu:

```powershell
netsh interface ipv4 show excludedportrange protocol=tcp
```

Vytvoření a kontrola clusteru (Docker Desktop musí běžet):

```powershell
kind create cluster --name demo --config kind-config.yaml
kubectl get nodes
```

Uzel `demo-control-plane` musí být ve stavu `Ready`. Kind cluster je ve skutečnosti jeden Docker kontejner s tímto jménem.

---

## Postup pro každou aplikaci

### 1. Aplikace

```powershell
dotnet new webapi -n DemoApi --use-minimal-apis
cd DemoApi
```

Do `Program.cs` před `app.Run();` přidej health endpoint (hodí se pro probe i pro test):

```csharp
app.MapGet("/health", () => Results.Ok(new { status = "ok", host = Environment.MachineName }));
```

Hodnota `host` ukáže název podu, který požadavek zpracoval.

### 2. Dockerfile

Ulož vedle `.csproj`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://+:5000
EXPOSE 5000
ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

Verzi `10.0` uprav podle použitého SDK (`8.0`, `9.0` apod.).

| Řádek | Význam |
|---|---|
| `FROM ...sdk AS build` | první fáze, image s plným SDK, slouží jen k sestavení |
| `WORKDIR /src` | pracovní složka uvnitř image |
| `COPY *.csproj .` + `RUN dotnet restore` | stažení NuGet balíčků zvlášť kvůli cache (rychlejší další buildy) |
| `COPY . .` | zkopíruje zdrojový kód |
| `RUN dotnet publish -c Release -o /app` | zkompiluje aplikaci do složky `/app` |
| `FROM ...aspnet` | druhá fáze, malý runtime image bez SDK |
| `COPY --from=build /app .` | přenese hotovou aplikaci z první fáze |
| `ENV ASPNETCORE_URLS=http://+:5000` | port, na kterém aplikace poslouchá (`+` = všechna rozhraní) |
| `EXPOSE 5000` | jen dokumentace portu |
| `ENTRYPOINT [...]` | příkaz spuštěný při startu kontejneru |

### 3. Sestavení image a lokální test

```powershell
docker build -t demoapi:1.0 .
docker run --rm --name demoapi -p 5000:5000 demoapi:1.0
```

Otevři `http://localhost:5000/health`. Pokud funguje, zastav kontejner přes `Ctrl+C`.

**Parametry:**

| Parametr | Význam |
|---|---|
| `-t demoapi:1.0` | jméno a verze (tag) image |
| `.` | build context, aktuální složka (zde se hledá `Dockerfile`) |
| `--rm` | po zastavení kontejner automaticky smaže |
| `--name demoapi` | vlastní jméno kontejneru (jinak náhodné, např. `cranky_diffie`) |
| `-p 5000:5000` | `port_na_počítači:port_v_kontejneru` |
| `-d` | běh na pozadí (nepovinné) |
| `-e PROMENNA=hodnota` | nastavení proměnné prostředí (nepovinné) |

Pokud log hlásí jiný port, než čekáš (např. 8080), po úpravě Dockerfilu jsi zapomněl znovu spustit `docker build`. Port na levé straně `-p` můžeš změnit bez zásahu do Dockerfilu (`-p 5100:5000`), např. když ho obsazuje XAMPP.

### 4. Nahrání image do kindu

Kind má vlastní runtime uvnitř uzlu a image z tvého Dockeru nevidí. Je nutné ho tam nahrát:

```powershell
kind load docker-image demoapi:1.0 --name demo
```

V Docker Desktopu se kvůli tomu nic nového neobjeví. Ověření:

```powershell
docker exec demo-control-plane crictl images | Select-String demoapi
```

### 5. Kubernetes manifesty

Soubor `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapi
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demoapi
  template:
    metadata:
      labels:
        app: demoapi
    spec:
      containers:
        - name: demoapi
          image: demoapi:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: demoapi
spec:
  selector:
    app: demoapi
  ports:
    - port: 80
      targetPort: 5000
```

**Význam:**

- **Deployment**: chci 2 repliky podů z image `demoapi:1.0`. Když pod spadne, Kubernetes ho nahradí.
- **`imagePullPolicy: IfNotPresent`**: použije se image nahraný do kindu, nehledá se na internetu.
- **`labels` + `selector`**: propojení Deploymentu s jeho pody.
- **`readinessProbe`**: pod dostane provoz až ve chvíli, kdy odpoví na `/health`.
- **Service**: stabilní adresa pro skupinu podů, rozděluje provoz mezi repliky.

Čísla na sebe musí navazovat: `ASPNETCORE_URLS` (5000) → `containerPort` a probe (5000) → `targetPort` Service (5000). Port 80 je jen port Service uvnitř clusteru.

### 6. Nasazení a kontrola

```powershell
kubectl apply -f k8s/deployment.yaml
kubectl get pods
```

Očekávaný stav: `Running` a `READY 1/1` u obou podů.

### 7. Přístup z počítače

```powershell
kubectl port-forward svc/demoapi 5100:80
```

Otevři `http://localhost:5100/health`. Příkaz drží terminál, další příkazy zadávej v druhém okně. `port-forward` se spojí s jedním konkrétním podem, takže střídání názvů podů v poli `host` nemusí být vidět. Rozdělování provozu mezi repliky dělá Service uvnitř clusteru.

---

## Nová verze aplikace

Při každé změně kódu použij **nový tag**. Při stejném tagu a `IfNotPresent` by Kubernetes dál používal starý image.

```powershell
docker build -t demoapi:1.1 .
kind load docker-image demoapi:1.1 --name demo
```

V `deployment.yaml` změň `image: demoapi:1.1` a spusť:

```powershell
kubectl apply -f k8s/deployment.yaml
```

Kubernetes provede postupnou výměnu podů bez výpadku.

---

## Pojmy

### Přehled

| Pojem | Stručný význam |
|---|---|
| **Cluster** | celá sestava strojů spravovaná Kubernetes |
| **Node** | jeden stroj v clusteru (ve kindu jeden Docker kontejner) |
| **Control plane** | "mozek" clusteru, rozhoduje o umístění podů a vystavuje API |
| **Worker** | node, na kterém běží aplikace |
| **Pod** | nejmenší nasaditelná jednotka, obal kolem kontejneru |
| **Deployment** | popis požadovaného stavu (např. 2 repliky), Kubernetes ho udržuje |
| **Service** | stabilní adresa a rozdělování provozu mezi pody |

### Vztah pojmů

```
Cluster
└── Node (stroj)
    └── Pod (obal kolem kontejneru)
        └── Kontejner (tvoje API z Docker image)
```

### Cluster

Celá sestava, kterou Kubernetes spravuje. Skládá se z jednoho nebo více nodů a z řídicí vrstvy, která rozhoduje, co kde poběží. Cluster si můžeš představit jako "jeden velký počítač" složený z více strojů: ty říkáš, **co** chceš (např. "2 kopie mého API"), a Kubernetes se postará o **kde** a **jak**.

- V produkci: obvykle několik až desítky nodů v cloudu (např. Azure AKS).
- U tebe: cluster `demo` s jedním nodem, vytvořený přes `kind`.
- Příkaz: `kubectl cluster-info`.

### Node

Jeden počítač (fyzický nebo virtuální), na kterém v clusteru běží aplikace. Je to "pracovní stroj" Kubernetes. Dva druhy:

- **Control plane** (řídicí node): rozhoduje, na který node se který pod umístí, hlídá, jestli skutečný stav odpovídá požadovanému, a vystavuje API, na které mluví `kubectl`.
- **Worker** (pracovní node): stroj, na kterém reálně běží pody s aplikacemi.

U tebe je `demo-control-plane` **jeden Docker kontejner**, který se tváří jako celý počítač. Je současně control plane i worker, takže na něm běží i tvoje API. Proto `kind load docker-image` posílá image právě do něj.

- Příkaz: `kubectl get nodes` (stav `Ready` znamená, že je node připraven).

### Pod

Nejmenší jednotka, kterou Kubernetes nasazuje. Obvykle je to obal kolem **jednoho kontejneru** (tvoje API). Důležité vlastnosti:

- Pod vždy běží na nějakém nodu.
- Má vlastní IP adresu uvnitř clusteru.
- Je **pomíjivý**: když zanikne (spadne, smaže se, nahradí se novou verzí), vznikne nový pod s **jiným názvem a jinou IP**. Proto se na pody neodkazuje napřímo, ale přes Service.
- Pody obvykle nevytváříš ručně, vytváří je Deployment.

Název podu `demoapi-7bfbccc4bb-2zws5` se skládá z: název Deploymentu (`demoapi`) + hash ReplicaSetu (`7bfbccc4bb`, mění se s každou novou verzí) + náhodná koncovka (`2zws5`). Hodnota `host` v odpovědi `/health` je právě název podu.

- Příkazy: `kubectl get pods`, `kubectl describe pod <pod>`, `kubectl logs <pod>`.

### Deployment

Popis požadovaného stavu aplikace: jaký image, kolik replik, jaké porty a probe. Kubernetes neustále porovnává **skutečný stav** s **požadovaným** a rozdíl opravuje. Proto když smažeš pod, vznikne okamžitě nový, aby počet replik seděl.

Deployment zároveň řeší postupnou aktualizaci (rolling update): při změně image tagu vymění pody po jednom bez výpadku.

- Příkazy: `kubectl get deployments`, `kubectl scale deployment demoapi --replicas=4`.

### Replika

Jedna kopie podu. `replicas: 2` znamená, že běží dva identické pody. Více replik = vyšší dostupnost (když jedna spadne, druhá obsluhuje provoz) a větší výkon.

### Service

Stabilní adresa pro skupinu podů. Protože pody vznikají a zanikají s měnícími se IP, Service je schová za jedno jméno (`demoapi`) a rozděluje provoz mezi zdravé repliky. Pody vybírá podle štítků (`selector: app: demoapi`).

V manifestu: `port: 80` je port Service uvnitř clusteru, `targetPort: 5000` je port v kontejneru.

- Příkaz: `kubectl get svc`.

### Další pojmy z manifestu

| Pojem | Význam |
|---|---|
| **Label (štítek)** | dvojice klíč-hodnota na objektu (`app: demoapi`), slouží k propojování objektů |
| **Selector** | výběr objektů podle štítků (Deployment a Service si tak najdou své pody) |
| **Manifest** | YAML soubor popisující objekty (Deployment, Service), který aplikuješ přes `kubectl apply` |
| **Namespace** | logické oddělení v rámci clusteru (např. `default`, `argocd`) |
| **Readiness probe** | kontrola, zda je pod připraven přijímat provoz (u nás dotaz na `/health`) |
| **Image** | šablona aplikace (výstup `docker build`), ze které vznikají kontejnery |
| **Kontejner** | běžící instance image, izolovaný proces s vlastním prostředím |
| **Kontext** | které clustery zná `kubectl`; kind nastaví kontext `kind-demo` |

### Shrnutí jednou větou

**Cluster** je sestava **nodů**, na nich běží **pody** s kontejnery, počet a podobu podů hlídá **Deployment** a **Service** k nim dává jednu stabilní adresu.

---

## Užitečné příkazy

### Kubernetes

| Příkaz | K čemu |
|---|---|
| `kubectl get pods` | stav podů |
| `kubectl describe pod <pod>` | události a důvody chyb |
| `kubectl logs <pod>` | výpis z aplikace |
| `kubectl get svc` | seznam služeb |
| `kubectl delete pod <pod>` | test samooprav (vznikne nový pod) |
| `kubectl scale deployment demoapi --replicas=4` | ruční škálování |

### Docker

```powershell
docker images           # seznam image
docker ps               # běžící kontejnery
docker ps -a            # včetně zastavených
docker stop demoapi     # zastavení kontejneru
docker logs demoapi     # výpis z aplikace
```

### Kind

```powershell
kind get clusters
kind delete cluster --name demo
```

---

## Typické chyby

| Chyba | Příčina a řešení |
|---|---|
| `no objects passed to apply` | YAML soubor je prázdný, neuložil se, nebo má příponu `.txt` navíc. Zkontroluj `type k8s\deployment.yaml` a `dir k8s`. |
| `connectex: ... actively refused` | Cluster neběží. Ověř `docker ps -a`, spusť Docker Desktop a `docker start demo-control-plane`. |
| `ports are not available ... forbidden by its access permissions` | Windows (Hyper-V/WSL2) blokuje starý port API serveru. Smaž cluster (`kind delete cluster --name demo`), vytvoř ho znovu (nejlépe s pevným portem v `kind-config.yaml`) a znovu proveď `kind load`. |
| `ErrImagePull` / `ImagePullBackOff` | Image není v uzlu. Zkontroluj `kind load` a shodu názvu a tagu v manifestu. Ověření přes `crictl images`. |
| Aplikace poslouchá na 8080 místo 5000 | Starý image. Po úpravě Dockerfilu znovu `docker build`. V nových verzích .NET lze použít `ASPNETCORE_HTTP_PORTS=5000`. |

**Poznámka:** po smazání a novém vytvoření clusteru je vždy nutné image znovu nahrát přes `kind load`, protože byl uvnitř starého uzlu.

---

## Další krok: Argo CD (GitOps)

Místo ručního `kubectl apply` budeš manifesty ukládat do Gitu a Argo CD je bude samo synchronizovat do clusteru.

1. Složku `k8s/` pushni do Git repozitáře (např. GitHub).
2. Nainstaluj Argo CD do clusteru:
   ```powershell
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
3. Zpřístupni UI: `kubectl port-forward svc/argocd-server -n argocd 8443:443`.
4. Vytvoř `Application`, která ukazuje na repozitář a složku s manifesty.
5. Změnu (tag image, počet replik) udělej v Gitu a Argo CD ji nasadí. Ruční změny v clusteru Argo vrátí zpět (self-heal).
