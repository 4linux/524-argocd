# Plataforma

Configuração das ferramentas de plataforma do laboratório do curso 524, que rodam no cluster Kubernetes.

Cada ferramenta possui um diretório com os arquivos utilizados na sua instalação.

|Ferramenta|Diretório|Instalação|
|---|---|---|
|SonarQube|[sonarqube/](sonarqube/)|Helm chart oficial `sonarqube/sonarqube`|
|Argo CD|[argocd/](argocd/)|Kustomize sobre o manifesto oficial `install.yaml`|

## SonarQube

```sh
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
helm install sonarqube sonarqube/sonarqube \
  --namespace sonarqube --create-namespace \
  --version 2026.5.1002 \
  -f sonarqube/values.yaml
```

Acesso: http://sonarqube.192.168.88.30.nip.io

Para aplicar uma alteração no arquivo de valores:

```sh
helm upgrade sonarqube sonarqube/sonarqube \
  --namespace sonarqube \
  --version 2026.5.1002 \
  -f sonarqube/values.yaml
```

## Argo CD

O diretório `argocd/` é um Kustomize que utiliza o manifesto oficial de instalação e acrescenta dois ajustes do laboratório: o Argo CD atendendo em HTTP (`server.insecure`) e o Ingress no Traefik. A versão do Argo CD é definida no `kustomization.yaml`.

```sh
kubectl apply --server-side -k argocd
```

Acesso: http://argocd.192.168.88.30.nip.io

Senha inicial do usuário `admin`:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Login pela CLI:

```sh
argocd login argocd.192.168.88.30.nip.io \
  --username admin --plaintext --grpc-web --skip-test-tls
```
