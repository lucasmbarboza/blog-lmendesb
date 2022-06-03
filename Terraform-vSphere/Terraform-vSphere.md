# Deploy VM Ubuntu 20.4 on vSphere 6.7 with Terraform

A idea desse post nasceu do desafio de usar IaC dentro de um DataCenter vmware. O objetivo era automatizar a criação de Cluster Vanilas de Kubernetes usando o Terraform.
O sistema operacional escolhido para os Nodes kubernetes foi o Ubuntu 20.4.4 LTS, que é uma versão estável do Ubuntu com suporte até 2025. 
Usar o Terraform para criar os Nodes deveria ser uma tarefa simples, mas se mostrou muito trabalhosa pelo fato da versão instalada do vCenter não suportar o Ubuntu 20.4.4 LTS. 

Os compontentes dessa bagunça são: 
* Vmware vCenter 6.7; (Aparentemente versões >= 6.7U3 não tem esse problema)
* Ubuntu 20.4.4; 
* Terraform;

Nesse blog vou mostrar a solução que eu usei para criar as VM's, usando o Terraform, a partir de um template customizado do Ubuntu. 
Esse processo pode ser divido em duas partes: 
1- Criação do Template no vCenter
2- Criação das VMs com Terraform

## 1- Criação do Template no vCenter

Com a VM do Ubuntu já instalada vamos fazer algumas modificações para que o template funcione bem com o Terraform. Alias, a maior parte  dos problemas que eu encontrei não foi com o Terraform e sim a dupla Cloud-Init e vCenter. Começamos instalando o open-vm-tools no Ubuntu.
```
sudo apt update
sudo apt install open-vm-tools -y 
```
Um passo muito importante é remover o cloud-init, caso isso não seja feito o vCenter não vai conseguir customizar a VM e o deploy do terraform irá falhar 😥. Meu objetivo inicial era automatizar a criação de um cluster Kubernetes, então as VMs deveriam ter IP fixo, por isso a customização é necessária.
```
sudo dpkg-reconfigure cloud-init
sudo apt-get purge cloud-init
sudo rm -rf /etc/cloud/ && sudo rm -rf /var/lib/cloud/
sudo reboot
```
Um outro problema que eu encontrei era que a interface de rede (NIC) não conectava a VM (Com isso o TerraForm falhava 🤦‍♂️). O problema era causado porque o dbus.service era executado depois do open-vm-tools.service. O dbus.service deveria identificar a placa de redes para então o open-vm-tools podesse customiza-la. Precisamos editar o arquivo do open-vm-tools.service, ele fica localizado em /li/systemd/system/open-vm-tools.service. Adicione a linha abaixo, ela deve ficar dentro bloco [units]. Com isso o open-vm-tools.service só será executado depois do dbus.service.

```
[units]
...
After=dbus.service

```

Devemos remover as configurações de rede da máquina. Apague os arquivos em /etc/netplan e desligue a VM.
```
sudo rm /etc/netplan/*.yaml
sudo shutdown now
```
Antes prosseguir devemos editar as settings da VM: 
* Mude do Network Adpter para a padrão do vCenter 
* O CD/DVD não pode ter nehuma ISO carregada escolha a opção Client Device