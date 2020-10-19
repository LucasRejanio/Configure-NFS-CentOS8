# Configure-NFS-CentOS8
NFS é um protocolo cliente / servidor de plataforma cruzada que permite que máquinas clientes acessem arquivos compartilhados pelo servidor NFS em uma rede. Os sistemas cliente podem montar localmente os sistemas de arquivos a partir do servidor NFS e acessar arquivos e diretórios como se estivessem montados localmente. Neste guia, vamos orientá-lo na instalação e configuração do NFS Server no CentOS 8 / RHEL 8. Nessa instalação vamos utilizar duas maquinas, chamaremos ela de NodeV01 e NodeV02, a V01 será nossa maquina provedora do serviço NFS, a V02 será a maquina CLient 

# NodeV01

## Primeira etapa: Instalar e configurar o NFS no servidor CentOS 8 / RHEL 8
Para começar, instalaremos o pacote do servidor NFS chamado nfs-utils que atua como o daemon NFS. Para instalar o pacote nfs-utils, inicie o terminal e execute o comando:

```console
$ sudo dnf install nfs-utils -y
```
Quando a instalação estiver concluída, inicie e habilite o serviço nfs-server para que ele seja feito automaticamente durante as reinicializações. Execute os seguintes comandos:

```console
$ sudo systemctl start nfs-server.service
$ sudo systemctl enable nfs-server.service
```

Para confirmar se o serviço NFS está em execução, execute:

```console
$ sudo systemctl status nfs-server.service
```

Você pode verificar a versão do protocolo nfs que está executando, executando o comando:

```console
$ rpcinfo -p | grep nfs
```

## Segunda etapa: Criação e exportação de compartilhamento NFS
Nesta etapa, vamos criar um sistema de arquivos que será compartilhado do servidor para os sistemas do cliente. Neste guia, criaremos um diretório em /mnt/nfs_share como mostrado abaixo:

```console
$ sudo mkdir -p /mnt/nfs_shares
```

Para evitar restrições de arquivo no diretório de compartilhamento NFS, é aconselhável configurar a propriedade do diretório conforme mostrado. Isso permite a criação de arquivos dos sistemas cliente sem encontrar quaisquer problemas de permissão.

```console
$ sudo chown -R nobody: /mnt/nfs_shares
```

Além disso, você pode decidir ajustar as permissões do diretório de acordo com sua preferência. Por exemplo, neste guia, iremos atribuir todas as permissões (ler, escrever e executar) para a pasta de compartilhamento NFS

```console
$ sudo chmod -R 777 /mnt/nfs_shares
```

Para que as alterações entrem em vigor, reinicie o daemon NFS:

```console
$ sudo systemctl restart nfs-utils.service
```

Para exportar o compartilhamento NFS para que os sistemas cliente possam acessá-lo, precisamos editar o arquivo /etc/exports. </br>

Exemplos: 
```
/mnt/nfs_shares CLIENT-IP:(rw, sync, no_all_squash, root_squash)
/mnt/nfs_shares *(rw, sync, no_all_squash, root_squash)
```
Vejamos o significado dos parâmetros usados:

- rw  - Significa leitura / gravação. Ele concede permissões de leitura e gravação ao compartilhamento NFS.
- sync - O parâmetro requer a gravação das alterações no disco antes que qualquer outra operação possa ser realizada.
- no_all_squash - Isso mapeará todos os UIDs e GIDs das solicitações do cliente para UIDS e GIDs idênticos residentes no servidor NFS.
- root_squash - O atributo mapeia solicitações do usuário raiz no lado do cliente para um UID / GID anônimo.

Para exportar a pasta criada acima, use o comando exportfs:

```console
$ sudo exportfs -arv
```

A opção -a implica que todos os diretórios serão exportados, -r significa reexportar todos os diretórios e, por fim, o sinalizador -v exibe a saída detalhada. </br>

Apenas para ter certeza sobre a lista de exportação, você pode exibir a lista de exportação usando o comando:

```console
$ sudo exportfs -s
```

# NodeV02

## Primeira etapa: Instale os pacotes NFS necessários: 

```console
$ sudo dnf install nfs-utils nfs4-acl-tools -y
```

Para exibir os compartilhamentos NFS montados no servidor, use o comando showmount:

```console
$ showmount -e 192.168.2.102
```

## Segunda etapa: Montar o compartilhamento NFS remoto localizado no servidor

Em seguida, precisamos montar o diretório de compartilhamento NFS remoto no sistema cliente local. Mas primeiro, vamos criar um diretório para montar o compartilhamento NFS.

```console
$ sudo mkdir p /mnt/client_share
```
Para montar o compartilhamento NFS, execute o comando abaixo. Lembre-se de que é nescessário passar o endereço IP do servidor NFS.

```console
$ sudo mount -t nfs SERVER-IP:/mnt/nfs_shares /mnt/client_share
```
ou
```console
$ sudo mount -t nfs4 SERVER-IP:/mnt/nfs_shares/ /mnt/client_share
```

Você pode verificar se o compartilhamento NFS remoto foi montado executando:
```console
$ sudo mount | grep -i nfs
```

Para tornar o compartilhamento de montagem persistente após uma reinicialização, você precisa editar o arquivo /etc/fstab e anexar a entrada abaixo.

```
SERVER-IP:/mnt/nfs_shares/ /mnt/client_share nfs defaults 0 0
```
```
#
# /etc/fstab
# Created by anaconda on Thu Apr 23 05:12:32 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk/'.
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
#
# After editing this file, run 'systemctl daemon-reload' to update systemd
# units generated from this file.
#
UUID=3510a17e-27dc-4ae2-9243-d600c16f4106 /                       xfs     defaults        0 0
SERVER-IP:/mnt/nfs_shares/  /mnt/client_share       nfs     defaults        0 0
```

## Testando a configuração do servidor e do cliente NFS

Neste ponto, concluímos todas as configurações. No entanto, precisamos testar nossa configuração e garantir que tudo funcione. Portanto, primeiro, vamos criar um arquivo de teste no diretório de compartilhamento do servidor NFS e verificar se ele está presente no diretório NFS montado do cliente.

NodeV01
```console
$ sudo touch /mnt/nfs_shares/server_nfs_file.txt
```

NodeV02
```console
$ ls -l /mnt/client_share
```
```console
$ sudo touch /mnt/client_share/client_nfs_file.txt
```

NodeV01
```console
$ ls -l /mnt/nfs_shares
```
