# Criando VMs Ubuntu 20.4 no vSphere 6.7 com Terraform

A idea desse post nasceu do desafio de usar IaC dentro de um DataCenter vmware. O objetivo era automatizar a criação de Cluster Vanilas de Kubernetes usando o Terraform.
O sistema operacional escolhido para os Nodes kubernetes foi o Ubuntu 20.4.4 LTS, que é uma versão estável do Ubuntu com suporte até 2025. 
Usar o Terraform para criar os Nodes deveria ser uma tarefa simples, mas se mostrou muito trabalhosa pelo fato da versão instalada do vCenter não suportar o Ubuntu 20.4.4 LTS. 

Os compontentes dessa bagunça são: 
* Vmware vCenter 6.7; (Aparentemente versões >= 6.7U3 não tem esse problema)
* Ubuntu 20.4.4; 
* Terraform;

Nesse blog vou mostrar a solução que eu usei para criar as VM's, usando o Terraform, a partir de um template customizado do Ubuntu. 
Esse processo pode ser divido em duas partes: 
1. Criação do Template no vCenter
2. Criação das VMs com Terraform

## 1. Criação do Template no vCenter

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
Um outro problema que eu encontrei era que a interface de rede (NIC) não conectava a VM (Com isso o TerraForm falhava 🤦‍♂️). O problema era causado porque o dbus.service era executado depois do open-vm-tools.service. O dbus.service deveria identificar a placa de redes para então o open-vm-tools podesse customiza-la. Precisamos que o serviço open-vm-tools seja executado antes do dbus, para isso precisamos editar o arquivo /li/systemd/system/open-vm-tools.service. Adicione a linha abaixo, ela deve ficar dentro do bloco [units]. 

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
* O CD/DVD não pode ter nehuma ISO carregada escolha a opção Client Device ou remova.

Clique com o botão direito na VM _template > convert to Template_ e então clique em Ok. 

## 2. Criação das VMs com Terraform

Existe um provider para o vSphere oferecido pela Vmware no harshiCorp. Ele atualmente está na versão 2.1.1, para usa-lo você vai precisar das credênciais do vCenter que você deseja usar. Abaixo as variavéis do terraform.tfvars. 

```
# terraform.tfvars
# vsphere Credentials
vcenter_username = ""
vcenter_password = ""
vcenter_server  = ""

# DC info
vsphere_cluster = ""
dc_name  = ""
template_name = ""

# VM info
vm_name = [""] # You can create multiple VM, just by adding to the list
vm_datastore = "" # In our case there is only one datastore
vm_network = "" 
host_name = [""] # Hostname list legth should be the same of vm_name 
ipv4_address = [""] # Ip address list legth should be the same of vm_name 
ipv4_gateway = ""

```
Você pode fazer um clone do repositório do módulo ou simplesmente importa-lo. Agora é só rodar o Terraform. 😍
```
terraform init
terraform plan
terraform apply
```
Referências: 
* https://kb.vmware.com/s/article/59687
* https://fabianlee.org/2021/08/16/terraform-creating-an-ubuntu-focal-template-and-then-guest-vm-in-vcenter/
* https://github.com/vmware/open-vm-tools/issues/421