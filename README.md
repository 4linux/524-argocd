Laboratório - 524 - CI/CD com Gitea Actions, Nexus, SonarQube e Argo CD
=============================

Repositório do laboratório do curso de Integração e Entrega Contínua da [4Linux][1], com o **Argo CD** como responsável pelo deploy no Kubernetes.

Dependências
------------

Para a criação do laboratório é necessário ter pré instalado os seguintes softwares:

* [Git][2]
* [VirtualBox][3]
* [Vagrant][5]

A máquina física precisa de no mínimo **16GB de memória RAM** e processador de 64 bits com virtualização habilitada.

> Para máquinas com Windows aconselhamos, se possível, que as instalações sejam feitas pelo gerenciador de pacotes **[Cygwin][6]**.

> Para máquinas com MAC OS aconselhamos, se possível, que as instalações sejam feitas pelo gerenciador de pacotes **brew**.

### Rede do VirtualBox (Linux e MAC OS)

O laboratório utiliza a rede `192.168.88.0/24`. Em Linux e MAC OS o VirtualBox só permite redes fora da faixa `192.168.56.0/21` quando liberadas no arquivo `/etc/vbox/networks.conf`. Crie o arquivo com o conteúdo abaixo:

```
* 192.168.88.0/24
```

Laboratório
-----------

O Laboratório será criado utilizando o [Vagrant][7], ferramenta para criar e gerenciar ambientes virtualizados com foco em automação.

Nesse laboratório, que está centralizado no arquivo [Vagrantfile][8], serão criadas 3 máquinas com as seguintes características:

Nome     | vCPUs | Memória RAM | IP            | S.O.¹              | Provisionamento²
-------- |:-----:|:-----------:|:-------------:|:------------------:| ---------------------------
ci-tools | 2     | 3072MB      | 192.168.88.10 | bento/ubuntu-24.04 | provision/ci-tools.yaml
nexus    | 1     | 2560MB      | 192.168.88.20 | bento/ubuntu-24.04 | provision/nexus.yaml
k3s      | 2     | 7168MB      | 192.168.88.30 | bento/ubuntu-24.04 | provision/k3s.yaml

> **¹**: Esses Sistemas operacionais estão sendo utilizados no formato de Boxes, a forma como o Vagrant chama as imagens do sistema operacional.

> **²**: O provisionamento é feito com Ansible, executado de dentro de cada máquina.

O que cada máquina entrega ao final do provisionamento:

Máquina  | Função no curso | O que já vem pronto
-------- | --------------- | -------------------
ci-tools | Servidor Git e CI. O runner do Gitea Actions e o Selenium são criados em aula | Docker, Docker Compose e Gitea em http://192.168.88.10:3000
nexus    | Repositório de artefatos e registry das imagens | Sonatype Nexus em http://192.168.88.20:8081
k3s      | Cluster Kubernetes onde rodam o Argo CD, o SonarQube e a aplicação, instalados em aula | k3s com Traefik, kubectl e Helm

O Gitea já é entregue instalado, com o Gitea Actions habilitado e o usuário administrador abaixo. O arquivo compose fica em `/opt/gitea/docker-compose.yaml` na máquina ci-tools.

Usuário | Senha
------- | ---------
root    | qwe123qwe

O Argo CD e o SonarQube são instalados durante o curso, no k3s. Para que o provisionamento já o entregue instalado, altere `argocd_install` para `true` no arquivo [provision/vars.yaml](provision/vars.yaml) antes do `vagrant up`.

As versões das ferramentas também estão centralizadas em [provision/vars.yaml](provision/vars.yaml).

Criação do Laboratório
----------------------

Para criar o laboratório é necessário fazer o `git clone` desse repositório e, dentro da pasta baixada, realizar a execução do `vagrant up`, conforme abaixo:

```bash
git clone https://github.com/4linux/524-argocd
cd 524-argocd/
vagrant up
```

_O Laboratório **pode demorar**, dependendo da conexão de internet e poder computacional, para ficar totalmente preparado._

> Em caso de erro na criação das máquinas sempre valide se sua conexão está boa e os logs de erros na tela. O provisionamento pode ser repetido com `vagrant provision <vm>`.

Validação
---------

Ao final, valide cada máquina:

```bash
vagrant status
vagrant ssh ci-tools -c "docker version --format '{{.Server.Version}}' && docker compose version"
curl -s http://192.168.88.10:3000/api/healthz
vagrant ssh nexus -c "docker ps"
vagrant ssh k3s -c "kubectl get nodes && kubectl get pods -A"
```

O Nexus leva alguns minutos para iniciar. Quando estiver pronto, a interface responde em http://192.168.88.20:8081. A senha inicial do usuário `admin` é obtida com:

```bash
vagrant ssh nexus -c "sudo cat /nexus-data/admin.password"
```

Comandos do Vagrant
-------------------

Comandos                | Descrição
:----------------------:| ---------------------------------------
`vagrant up`            | Cria/Liga as VMs baseado no Vagrantfile
`vagrant up <vm>`       | Cria/Liga somente uma VM
`vagrant provision`     | Provisiona mudanças lógicas nas VMs
`vagrant status`        | Verifica se as VMs estão ativas ou não
`vagrant ssh <vm>`      | Acessa a VM
`vagrant ssh <vm> -c <comando>` | Executa comando via ssh
`vagrant reload <vm>`   | Reinicia a VM
`vagrant halt`          | Desliga as VMs
`vagrant destroy`       | Remove as VMs

> Para maiores informações acesse a [Documentação do Vagrant][13]

[1]: https://4linux.com.br
[2]: https://git-scm.com/downloads
[3]: https://www.virtualbox.org/wiki/Downloads
[5]: https://www.vagrantup.com/downloads
[6]: https://cygwin.com/install.html
[7]: https://www.vagrantup.com/
[8]: ./Vagrantfile
[13]: https://www.vagrantup.com/docs
