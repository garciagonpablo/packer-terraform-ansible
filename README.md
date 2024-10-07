Utilización de [[terraform]], [[ansible]] y [[packer]] para configuración de escenarios de certificación.

Este documento usa como base la serie de 3 artículos escritos por Dmitry Teslya:
- [Parte 1](https://dteslya.engineer/blog/2018/12/20/automate-windows-vm-creation-and-configuration-in-vsphere-using-packer-terraform-and-ansible-part-1-of-3/)
- [Parte 2](https://dteslya.engineer/blog/2019/01/21/automate-windows-vm-creation-and-configuration-in-vsphere-using-packer-terraform-and-ansible-part-2-of-3/)
- [Parte 3](https://dteslya.engineer/blog/2019/02/19/automate-windows-vm-creation-and-configuration-in-vsphere-using-packer-terraform-and-ansible-part-3-of-3/)

En estos artículos se empieza por la creación de un OS template utilizando [[packer]], luego pasarán a [[terraform]] para inicializar las VMs y por ultimo se utiliza [[ansible]] para configurar el OS dentro de las VMs.
 Por lo que se recomienda leyendo el archivo de [[packer]].
