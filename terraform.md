Infrastructure as Code.


Para este paso se utilizarán los siguientes [archivos](https://github.com/dteslya/win-iac-lab/tree/master/terraform).
En este directorio se encuentran los archivos:
```
.
├── 01-PDC.tf
├── 02-ReplicaDC.tf
├── 03-FileServer.tf
├── base.tf
├── terraform.tfvars.example
└── variables.tf
```
Y estos se dividen en 3 partes:
- `base.tf`: En este archivo se define la conexión con vCenter y parámetros básicos como el nombre del datacenter, compute y storage clusters, y el template para la VM.  
- `01-PDC.tf`, `02-ReplicaDC.tf` y `03-FileServer.tf`: aquí están definidas diferentes templates y parámetros importantes como: numero de CPUs, RAM, tamaño de discos, etc.
- `variables.tf` y `terraform.tfvars.example`: estos archivos se utilizan para declarar y definir las variables respectivamente. (Es recomendable añadir el archivo `terraform.tfvars` al `.gitignore` para evitar difundir datos sensibles)

Se instalará el plugin de `vSphere`, especificado en el archivo `base.tf` y leerán los archivos `tf` que estén en el directorio.
```sh
terraform init
```

Se obtiene plan de lo que hará **terraform** en caso se continúe con el proceso imprimiendo en consola las configuraciones. (Este paso puede ser saltado)
```sh
terraform plan
```

Se aplicará lo que se describió anteriormente:
```sh
terraform apply
```

Esto creará 3 VMs. En caso de necesitar cambiar algún parámetro de alguna de las VMs se puede correr de nuevo el comando `terraform apply` con los cambios en el archivo correspondiente.

Para eliminar las VMs:
```sh
terraform destroy
```
 
 Para eliminar solamente una VM:
```sh
terraform destroy -target=vsphere_virtual_machine.01-PDC
```

### Nota:
Dentro de los archivos `tf` hay una sección de `customize` en la cual se hace una [guest customization](https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.vm_admin.doc/GUID-F3E382AB-72F6-498A-BD26-7EC0BFE320A0.html).  Esto se hace para evitar problemas que puedan surgir cuando se crean VMs iguales, como lo podría ser nombres de computadoras duplicados. 

Con esto completado se puede pasar al contenido de [[ansible]].

Referencias de la referencia:
- [Running Terraform in Automation](https://learn.hashicorp.com/terraform/development/running-terraform-in-automation)
- [An Introduction to Terraform](https://blog.gruntwork.io/an-introduction-to-terraform-f17df9c6d180)
- [Datanauts 137: Automating Infrastructure As Code With Terraform](https://packetpushers.net/podcast/datanauts-137-automating-infrastructure-code-terraform/)
- [Sysprep and VMware Guest Customization with Terraform](https://www.virtualizationhowto.com/2018/06/sysprep-and-vmware-guest-customization-with-terraform/)
