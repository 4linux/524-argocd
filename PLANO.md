# Infra do laboratório 524-argocd — mapeamento

Plano da infra do curso com Argo CD. **Implementado**: Vagrantfile e playbooks escritos; as três VMs subiram e passaram nos testes manuais em 05/10/2026 (antes de o Gitea entrar no provisionamento da `ci-tools`). A infra atual em `TJRR/524` continua intocada; esta é uma infra separada, baseada nela.

## VMs

| VM | IP | RAM | CPU | O que roda |
|---|---|---|---|---|
| `ci-tools` | 192.168.88.10 | 3GB | 2 | Gitea, runner do Gitea Actions, Selenium Grid (Docker) |
| `nexus` | 192.168.88.20 | 2,5GB | 1 | Sonatype Nexus (Docker), registry das imagens |
| `k3s` | 192.168.88.30 | 7GB | 2 | k3s com Traefik, Argo CD, SonarQube (Helm), aplicação em homolog e production |

Total de cerca de 12,5GB: o requisito do host passa a ser **16GB de RAM**.

## Decisões

- **Sem Jenkins e sem GitLab**: sai a VM `gitlab-ci`, o usuário e as configurações do Jenkins.
- **CI no Gitea Actions**: runner na `ci-tools`, usando o Docker da VM para os builds (sem Docker-in-Docker).
  - Runner registrado em aula, junto com o aluno (não é provisionado).
  - Rodar o runner no k3s foi descartado como caminho principal: exige pod privilegiado com DinD, o que dá ao CI acesso de root ao nó e contradiz a separação entre CI e cluster. Pode virar tópico extra.
- **Gitea fora do cluster**: o Git não pode depender do cluster que o Argo CD controla.
- **Nexus em VM própria**: é o registry do k3s, então fica fora do cluster. O IP `.20` mantém o endereço `192.168.88.20:8082` já usado nos manifestos da aplicação.
- **SonarQube dentro do k3s**, instalado pelo Argo CD via Helm chart oficial. É o exemplo de "Argo CD gerenciando software de terceiros".
- **Gitea já instalado na `ci-tools`** pelo provisionamento: sem assistente de primeiro acesso, Actions habilitado, admin `root`/`qwe123qwe`, compose em `/opt/gitea`. Runner e Selenium continuam para a aula.
- **No k3s só entram Argo CD e SonarQube, instalados em aula.** A variável `argocd_install` permite já entregar o Argo CD instalado.
- **Ingress com Traefik**, que já vem no k3s. O ingress-nginx foi descontinuado em março de 2026.
  - Impacto: trocar `ingressClassName: nginx` por `traefik` em `simplePythonFlask/deploy/base/web.yaml`.

## O que muda em relação ao `TJRR/524`

| Atual | Novo |
|---|---|
| `ubuntu/focal64` (sem suporte; prende o k3s na v1.30) | `bento/ubuntu-24.04` |
| Plugins `vagrant-vbguest` e `vagrant-disksize` | Nenhum plugin |
| Ansible via `pip install "ansible<7"` | Ansible do apt do Ubuntu (já traz `community.docker`) |
| Repositório Docker fixo em `xenial` com `apt_key` | Repositório oficial com keyring em `/etc/apt/keyrings` |
| Tarefa de swap com bug (cria `/extraswap` sem uso) | Removida; a memória das VMs foi aumentada |
| Calico + ingress-nginx v1.1.2 | Flannel e Traefik padrão do k3s |
| SonarQube 9.6.1 em Docker | SonarQube no k3s via Helm |
| `jenkins-k8s.yaml` (dá `cluster-admin` à SA `default`) | Removido |
| `provision/compose/`, `provision/archive/`, `ansible/roles/sistema` (sem uso) | Removidos |

## Provisionamento planejado

Estrutura:

```
Vagrantfile
provision/
├── vars.yaml          # IPs, versões, argocd_install
├── tasks/
│   ├── common.yaml    # /etc/hosts (*.4labs.example), pacotes básicos
│   └── docker.yaml    # Docker CE + compose + insecure-registries do Nexus
├── templates/         # compose do Gitea
├── ci-tools.yaml      # common + docker + Gitea instalado (runner e Selenium em aula)
├── nexus.yaml         # common + docker + container do Nexus
└── k3s.yaml           # common + sysctl do Sonar + registries.yaml + k3s + kubeconfig + Helm
```

Detalhes:

- **Vagrant**: `ansible_local` com `install = false`; um shell provisioner antes instala `ansible` pelo apt.
- **nexus**:
  - Container `sonatype/nexus3:3.96.4`, portas 8081, 8082 e 8083.
  - Volume em `/nexus-data`, com dono UID 200.
  - Heap reduzido com `INSTALL4J_ADD_VM_PARAMS="-Xms1g -Xmx1g -XX:MaxDirectMemorySize=1g"`.
- **k3s**:
  - `sysctl` `vm.max_map_count=524288` e `fs.file-max=131072`, exigidos pelo SonarQube.
  - `/etc/rancher/k3s/registries.yaml` apontando `192.168.88.20:8082` para `http://`, criado **antes** de instalar o k3s.
  - k3s `v1.36.5+k3s1` com `--node-ip 192.168.88.30 --write-kubeconfig-mode 644`.
  - Kubeconfig em `~vagrant/.kube/config`, Helm `v4.3.0`, alias `k`.
  - Se `argocd_install: true`: namespace `argocd` e `kubectl apply --server-side` do `install.yaml` da `v3.5.3`.

Versões consultadas em 02/10/2026: k3s `v1.36.5+k3s1`, Nexus `3.96.4`, Helm `v4.3.0`, Argo CD `v3.5.3`. Conferir de novo ao implementar.

## Pontos de atenção

- **Rede `192.168.88.0/24`**: o VirtualBox, em hosts Linux e macOS, só aceita redes host-only fora de `192.168.56.0/21` se estiverem liberadas em `/etc/vbox/networks.conf`. Documentar no README, por exemplo com a linha `* 192.168.88.0/24`.
- **Nexus Community Edition**: tem limites de uso (confirmar os números atuais no site da Sonatype antes de citar na apostila).
- **Memória do k3s**: validar se 7GB comportam Argo CD, SonarQube e os dois ambientes da aplicação.
- **Chart do SonarQube**: definir versão e valores mínimos para o lab (sem PostgreSQL externo, ou com o embutido do chart).

## Próximos passos

Pendente desta etapa:

- Rodar `vagrant provision ci-tools` e validar o Gitea (o playbook do Gitea ainda não rodou na VM; a configuração foi testada só em container local).
- Fazer o commit inicial deste repositório.

Roteiro combinado para a volta, nesta ordem:

1. **Configurar o Gitea**: repositórios da aplicação e de deploy, runner do Gitea Actions na `ci-tools`.
2. **Configurar o Nexus**: repositório `docker-hosted` na porta 8082, usuário e permissões. A versão 3.96 tem interface bem diferente da apostila; revisar telas e passos.
3. **Configurar o SonarQube** no k3s (Helm), projeto e token.
4. **Configurar o Argo CD** no k3s.
5. **Criar a pipeline inteira**, com todos os steps, e validar de ponta a ponta.
6. **Quebrar a pipeline em partes** para o curso: o aluno constrói passo a passo, de forma didática.

Ajustes técnicos já conhecidos:

- Trocar o `ingressClassName` da aplicação de `nginx` para `traefik` (`simplePythonFlask/deploy/base/web.yaml`).
- Definir onde ficam os manifestos do SonarQube e do App of Apps da plataforma.
- Validar um job real com `docker build` dentro do container do runner.
