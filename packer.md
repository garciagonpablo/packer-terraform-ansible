Images as code. Herramienta Open Source desarrollada por Hashicorp.


Para el siguiente setup se estarán utilizando los siguiente [archivos](https://github.com/dteslya/win-iac-lab/tree/master/packer).

## Setup

```sh
packer build -var-file=vars.json windows-server-2016.json
```

Con esto packer hará lo siguiente:
1. Se conectará a vpshere para crear la máquina virtual.
2. Montar las imágenes ISO especificadas en el archivo json.
3. Creará un floppy disk y pondrá los archivos del directorio `setup` en él.
4. Monta el floppy disk.
5. Encenderá la VM y esperará a tener una dirección IP.
6. Conectará a Windows via WinRM (Windows Remote Managment).
7. Corre los scripts provisionales.
8. Apaga la VM y la convierte en un template.

Los archivos necesarios para este proceso son los siguientes:
### `vars.json`: 
Archivo con variables sobre el entorno que se está intentando realizar.

### `windows-server-2016.json`: 
El archivo de configuración **principal** para packer.
- En la primera parte se declaran todas las variables y se definen las variables sensibles para que no sus valores no sean impresos durante el build run.
- En la segunda parte llamada `builders` se le indica a `vpshere-iso` como conectarse y dónde colocar la VM template.
```json
"communicator": "winrm",
"winrm_username": "Administrator",
"winrm_password": "{{user `winardmin_password`}}",
```
Esta sección es crucial ya que le indica a packer como conectarse con el guest OS. Estas credenciales son configuradas durante la configuración de windows.

- Floppy Files: Aquí se indica el Windows ISO files y VMWare Tools ISO que se le pasarán como CDROM files al guest OS. El guest OS montará el los ISO en el orden en el que sean presentados.
```json
"iso_paths": [
	"{{user `os_iso_path`}}",
	"{{user `vmtools_iso_path`}}"
]
```
En este caso el ISO de windows será **D:** y el ISO de VMTools será **E:**. 


### `autounattend.xml`
El segundo archivo más importante. Llamado archivo respuesta, el el que permite automatizar completamente la instalación de Windows. El setup de windows lee este archivo. Este archivo puede ser creado por ti u obtenido por medio de un repositorio.

Para crear el archivo de cero dirigirse al siguiente [recurso](https://www.derekseaman.com/2012/07/windows-server-2012-unattended.html), el cuál habla de como configurar una WS 2012 por lo que habrá que buscar la diferencia entre un Windows Server 2012 y el que se busca configurar, por ejemplo entre el 2012 y 2016 la diferencia es que se debe de cambiar el nombre de la imagen a: *Windows Server 2016 SERVERSTANDARD* y hay que establecer el tamaño de la primera partición a *500*.

- La primera sección importante de este archivo es `specialize` pass:
```xml
<settings pass="specialize">
        ...
            <RunSynchronous>
                <RunSynchronousCommand wcm:action="add">
                    <Path>a:\vmtools.cmd</Path>
                    <Order>1</Order>
                    <WillReboot>Always</WillReboot>
                </RunSynchronousCommand>
            </RunSynchronous>
        ...
    </settings>
```
El cual comanda a Windows Setup a correr `vmtools.cmd` batch script desde el floppy drive.


- La segunda sección importante es el `oobeSystem` pass:
```xml
<settings pass="oobeSystem">
        ...
            <AutoLogon>
                <Password>
                    <Value>S3cret!</Value>
                    <PlainText>true</PlainText>
                </Password>
                <LogonCount>2</LogonCount>
                <Username>Administrator</Username>
                <Enabled>true</Enabled>
            </AutoLogon>
            <FirstLogonCommands>
                <SynchronousCommand wcm:action="add">
                    <Order>1</Order>
                    <!-- Enable WinRM service -->
                    <CommandLine>powershell -ExecutionPolicy Bypass -File a:\setup.ps1</CommandLine>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
            </FirstLogonCommands>
            <UserAccounts>
                <AdministratorPassword>
                    <Value>S3cret!</Value>
                    <PlainText>true</PlainText>
                </AdministratorPassword>
            </UserAccounts>
        ...
    </settings>
```
Esta sección le comanda a Windows Setup a realizar *autologon* usando la cuenta de *Adminstrator*, la cual también es configurada en esta sección. También manda a correr `setup.ps1` powershell script.

### `setup.ps1`:
Este script tiene tres funciones.
1. Cambia el perfil de Windows Firewall a privado para que Windows pueda aceptar incoming network connections.
```ps1
$profile = Get-NetConnectionProfile
Set-NetConnectionProfile -Name $profile.Name -NetworkCategory Private
```

2. Luego permite a *packer* para que pueda conectarse a windows.
```ps1
winrm quickconfig -quiet
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
winrm set winrm/config/service/auth '@{Basic="true"}'
```

3. Por ultimo resetea el contador de *autologon* a 0 por razones de seguridad (para que una vez creada el sistema ya no pueda ingresar como *Administrator* automáticamente).
```ps1
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -Name AutoLogonCount -Value 0
```

### `vmtools.cmd`
Este script instala VMWare Tools.
```cmd
e:\setup64 /s /v "/qb REBOOT=R"
```
Importante mencionar que se esta haciendo en el drive letter es la **E:** ya que en la **D:** está el ISO de Windows.


## Build:
Para llevar acabo la build se corre el siguiente comando:
```sh
packer build -var-file=vars.json windows-server-2016.json
```



Con esto ya se tiene una configuración básica que permite automatizar la build de Windows Server 2016. Pasos extra podrían ser automatizar la instalación de windows updates, pre-instalar software usando `Chocolatey`, permitir remote desktop, etc.

Con esto ya podemos pasar a la sección de [[terraform]].

Referencias de la referencia:
- [Official Packer documentation](https://packer.io/docs/index.html)
- [Packer Builder for VMware vSphere](https://github.com/jetbrains-infra/packer-builder-vsphere)
- [Using Packer to create Windows images](https://www.bloggingforlogging.com/2017/11/23/using-packer-to-create-windows-images/) by Jordan Borean
- [Big collection of Windows templates](https://github.com/StefanScherer/packer-windows) by Stefan Scherer on Github