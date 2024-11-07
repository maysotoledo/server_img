**Passo a Passo para Configurar um Servidor de Imagens com Ubuntu para Clonar e Restaurar Máquinas pela Rede**

**1. Instalação e Configuração do Servidor TFTP e DHCP**

1.1. **Instalar os pacotes necessários**:
   - Abra o terminal e instale os pacotes `tftpd-hpa`, `dnsmasq`, e `nfs-kernel-server`.
     ```bash
     sudo apt update
     sudo apt install tftpd-hpa dnsmasq nfs-kernel-server
     ```

1.2. **Configurar o TFTP**:
   - Edite o arquivo de configuração do TFTP em `/etc/default/tftpd-hpa`:
     ```bash
     sudo nano /etc/default/tftpd-hpa
     ```
   - Altere as configurações para:
     ```
     TFTP_USERNAME="tftp"
     TFTP_DIRECTORY="/srv/tftp"
     TFTP_ADDRESS=":69"
     TFTP_OPTIONS="--secure"
     ```
   - Crie o diretório para o TFTP e defina as permissões:
     ```bash
     sudo mkdir -p /srv/tftp
     sudo chown -R tftp:tftp /srv/tftp
     ```

1.3. **Configurar o DNSMASQ como servidor DHCP**:
   - Edite o arquivo de configuração em `/etc/dnsmasq.conf`:
     ```bash
     sudo nano /etc/dnsmasq.conf
     ```
   - Adicione as linhas abaixo:
     ```
     interface=eth0  # Altere para a interface correta
     dhcp-range=192.168.1.100,192.168.1.200,12h
     dhcp-boot=pxelinux.0
     enable-tftp
     tftp-root=/srv/tftp
     ```
   - Reinicie o `dnsmasq`:
     ```bash
     sudo systemctl restart dnsmasq
     ```

**2. Configurar o PXE e Clonezilla**

2.1. **Baixar e configurar o Clonezilla**:
   - Baixe a imagem do Clonezilla Live:
     ```bash
     wget https://clonezilla.org/downloads/stable/iso/clonezilla-live.iso -P /srv/tftp
     ```
   - Extraia os arquivos necessários:
     ```bash
     sudo mkdir -p /srv/tftp/clonezilla
     sudo mount -o loop /srv/tftp/clonezilla-live.iso /mnt
     sudo cp -r /mnt/* /srv/tftp/clonezilla/
     sudo umount /mnt
     ```

2.2. **Configurar o SYSLINUX para boot PXE**:
   - Instale o `syslinux`:
     ```bash
     sudo apt install syslinux
     ```
   - Copie o arquivo `pxelinux.0` para o diretório TFTP:
     ```bash
     sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp
     ```
   - Crie o diretório de configuração do PXE:
     ```bash
     sudo mkdir -p /srv/tftp/pxelinux.cfg
     ```
   - Crie o arquivo de configuração em `/srv/tftp/pxelinux.cfg/default`:
     ```bash
     sudo nano /srv/tftp/pxelinux.cfg/default
     ```
   - Adicione as seguintes linhas:
     ```
     DEFAULT clonezilla-menu
     PROMPT 0
     TIMEOUT 50

     LABEL clonezilla-menu
         MENU LABEL Clonezilla Menu
         KERNEL /clonezilla/live/vmlinuz
         APPEND initrd=/clonezilla/live/initrd.img boot=live union=overlay username=user config components noswap edd=on nomodeset ocs_live_run="ocs-live-general" ocs_live_extra_param="" keyboard-layouts="" ocs_live_batch="no" locales="en_US.UTF-8" ocs_prerun="mount /srv/nfs_share" fetch=tftp://192.168.1.1/clonezilla/live/filesystem.squashfs
     ```

**3. Configurar o Servidor NFS**

3.1. **Criar o diretório de imagens**:
   - Crie o diretório para armazenar as imagens:
     ```bash
     sudo mkdir -p /srv/nfs_share
     sudo chmod 777 /srv/nfs_share
     ```

3.2. **Configurar o NFS**:
   - Edite o arquivo `/etc/exports`:
     ```bash
     sudo nano /etc/exports
     ```
   - Adicione a linha:
     ```
     /srv/nfs_share 192.168.1.0/24(rw,sync,no_subtree_check)
     ```
   - Exporte a configuração:
     ```bash
     sudo exportfs -a
     sudo systemctl restart nfs-kernel-server
     ```

**4. Testar o Boot pela Rede**

4.1. **Configurar as estações de trabalho**:
   - Acesse a BIOS das máquinas e configure o boot pela rede (PXE).

4.2. **Iniciar o processo de clonagem/restauração**:
   - Quando as máquinas fizerem boot, o menu do Clonezilla será exibido com opções de clonar para o servidor ou restaurar a partir da imagem.

