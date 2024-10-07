Configuration as code. Esta herramienta nos servirá para provisionar las VMs. 

Para utilizar `ansible` con una VM de windows es necesario instalar la librería `pywinrm`.
```sh
pip install pywinrm
```

Utilizando el siguiente [directorio](https://github.com/dteslya/win-iac-lab/tree/master/ansible). Editar los archivos `inventory.yml` y `group_vars/all.yml` según los parámetros del entorno que se utilizará. 

`Ansible` ofrece una herramienta llamada `Ansible Vault` para almacenar credenciales encriptadas en el archivo `group_vars/all.yml` usando el comando:
```sh
ansible-vault encrypt_string <sting_to_encrypt>
```
Repetir para cada una de las contraseñas. Almacena la contraseña en el archivo `.vault_pass` (colocar en el `.gitignore` para no difundir la data).

Siguiendo esta estructura del directorio:
```
.
├── group_vars
│   └── all.yml
├── roles
│   ├── common
│   │   └── tasks
│   │       ├── enable_rdp.yml
│   │       └── main.yml
│   ├── fileserver
│   │   └── tasks
│   │       └── main.yml
│   ├── primary_domain_controller
│   │   └── tasks
│   │       └── main.yml
│   └── replica_domain_controller
│       └── tasks
│           └── main.yml
├── .vault_pass
├── ansible.cfg
├── inventory.yml
└── winlab.yml
```

Ansible leerá el archivo `ansible.cfg` que apunta al archivo `inventory.yml` y al archivo vault password `.vault_pass`. 
```sh
ansible-playbook winlab.yml
```
Luego comenzará a aplicar los roles descritos en `winlab.yml` a los hosts descritos en `inventory.yml`.

Los roles están configurados en el archivo: `roles/primary_domain_controller/tasks/main.yml`.
```yml
---
- name: install ad
  win_domain:
    dns_domain_name: "{{ domain }}"
    domain_netbios_name: "{{ netbios_domain }}"
    safe_mode_password: "{{ domain_safemode_password }}"
  register: ad

- name: reboot server
  win_reboot:
    msg: "Installing AD. Rebooting..."
    pre_reboot_delay: 15
    reboot_timeout: 600
    post_reboot_delay: 420
  when: ad.changed
```
En el `install ad` se instalará el rol AD DS rol en el servidor el cuál crea un nuevo forest (contenedor top-level de un set de estructuras que comparten un esquema, configuración y catalogación global, contiene uno o mas dominos). Y después promueve el servidor a un controlador de dominio (PDC) (Servidor en un ambiente de Windows Active Directory responsable de manejar la seguridad de red, autenticación de usuarios y permiso de accesos). 

Luego con el `reboot server` se reiniciará el servidor si la tarea anterior regresa un estado de *changed* en `ad.changed`.

Para instalar el módulo de powershell de `xRemoteDesktopAdmin` con `in_psmodule`, se utiliza el rol `roles/common/tasks/main.yml` que llama `roles/common/tasks/enable_rdp.yml`. Esto activa el Remote Desktop Protocol (RDP) hacia el PDC.
```yml
- name: Windows | Check for xRemoteDesktopAdmin Powershell module
  win_psmodule:
    name: xRemoteDesktopAdmin
    state: present

- name: Windows | Enable Remote Desktop
  win_dsc:
    resource_name: xRemoteDesktopAdmin
    Ensure: present
    UserAuthentication: Secure

- name: Windows | Check for xNetworking Powershell module
  win_psmodule:
    name: xNetworking
    state: present

- name: Firewall | Allow RDP through Firewall
  win_dsc:
    resource_name: xFirewall
    Name: "Administrator access for RDP (TCP-In)"
    Ensure: present
    Enabled: True
    Profile: "Domain"
    Direction: "Inbound"
    Localport: "3389"
    Protocol: "TCP"
    Description: "Opens the listener port for RDP"
```
Luego de esto el módulo `xNetworking` es instalado y se abre el puerto del RDP en el Windows Firewall con `win_dsc`. 

El Replica Domain Controller (RDC) es configurado por el script: `roles/replica_domain_controller/tasks/main.yml`.
```yml
---
- name: change DNS server
  when: not ansible_windows_domain_member
  win_dns_client:
    adapter_names: '*'
    ipv4_addresses: "{{ groups['primary_domain_controller'][0] }}"

- name: join domain
  win_domain_membership:
    dns_domain_name: "{{ domain }}"
    domain_admin_user: "{{ domain_admin }}"
    domain_admin_password: "{{ domain_admin_password }}"
    state: domain
  register: domain_joined

- name: reboot after domain join
  win_reboot:
  when: domain_joined.reboot_required

- name: Wait for system to become reachable over WinRM
  wait_for_connection:
    timeout: 900

- name: install ad
  win_domain_controller:
    dns_domain_name: "{{ domain }}"
    domain_admin_user: "{{ domain_admin }}"
    domain_admin_password: "{{ domain_admin_password }}"
    safe_mode_password: "{{ domain_safemode_password }}"
    state: domain_controller
  register: ad

- name: reboot server
  win_reboot:
    msg: "Installing AD. Rebooting..."
    pre_reboot_delay: 15
  when: ad.changed
```
En este script se cambia la dirección a la que apunta el DNS hacia el PDC y luego se hace un reboot. Luego el AD rol es instalado y se reinicia de nuevo. 


El servidor de archivos es instalado por `roles/fileserver/tasks/main.yml`
```yml
---
- name: change DNS server
  win_dns_client:
    adapter_names: '*'
    ipv4_addresses: 
      - "{{ groups['primary_domain_controller'][0] }}"
      - "{{ groups['replica_domain_controller'][0] }}"

- name: join domain
  win_domain_membership:
    dns_domain_name: "{{ domain }}"
    domain_admin_user: "{{ domain_admin }}"
    domain_admin_password: "{{ domain_admin_password }}"
    state: domain
  register: domain_joined

- name: reboot after domain join
  win_reboot:
  when: domain_joined.reboot_required
```
Esto repite lo que hizo el script del Secundary Domain Controller (SDC) sin la tarea de `install ad`.

Referencias de la referencia:
- [Ansible Windows Guide](https://docs.ansible.com/ansible/latest/user_guide/windows.html)
- [Ansible & DSC](https://docs.ansible.com/ansible/latest/user_guide/windows_dsc.html)
- [Manage Windows like Linux with Ansible](https://www.youtube.com/watch?v=FEdXUv02Dbg)
- [Configure An Ansible Testing System On Windows (Part 2)](https://www.frostbyte.us/configure-an-ansible-testing-system-on-windows-part-2/)
- [Configure An Ansible Testing System On Windows (Part 3)](https://www.frostbyte.us/configure-an-ansible-testing-system-on-windows-part-3/)