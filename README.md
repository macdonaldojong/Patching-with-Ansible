# Patching-with-Ansible

'''
[root@ansible-master ansible]# cat hosts
[dev-redhat]
testserver1
testserver2
[root@ansible-master ansible]#

[root@ansible-master ansible]# ansible dev-redhat -m shell -a "uname -a;uptime"
[root@ansible-master ansible]# ansible dev-redhat -m shell -a "ps -eaf | grep apache"
[root@ansible-master ansible]# ansible dev-redhat -m shell -a "systemctl stop httpd.service"

[root@ansible-master ansible]# cat linuxpatching.yml
---
## Demo Ansible Playbook to perform patching on RHEl/CentOS Server
## For Demo Purpose only.. Use it at your own responsibility.
## Can update it as per your requirements - Yogesh 27/12/2016

- hosts: dev-redhat
  become_user: root
  serial: 2


  tasks:

    # purpose of this task to check if application is running or stopped
    - name:  verify application/database processes are not running
      shell: if ps -eaf | egrep 'apache|http'|grep -v grep > /dev/null ;then echo 'process_running';else echo 'process_not_running';fi
      ignore_errors: true
      register: app_process_check

    # this task is decision,play will fail/quit,if application is running
    - name:  decision point to start patching
      fail: msg="{{ inventory_hostname }} have running Application.Please stop the application first, then attempt patching."
      when: app_process_check.stdout == "process_running"

    # this task will upgrade/install the rpm's if application is stopped
    - name:  upgrade all packages on the server
      yum:
       name="kernel"
       state=latest
      when: app_process_check.stdout == "process_not_running" and ansible_distribution == 'CentOS' or ansible_distribution == 'Red Hat Enterprise Linux'
      register: yum_update

    # this task is to check if kernel update happend and system needs reboot ot not
    - name: check if reboot required after kernel update.
      shell: KERNEL_NEW=$(rpm -q --last kernel |head -1 | awk '{print $1}' | sed 's/kernel-//'); KERNEL_NOW=$(uname -r); if [[ $KERNEL_NEW != $KERNEL_NOW ]]; then echo "reboot_needed"; else echo "reboot_not_needed"; fi
      ignore_errors: true
      register: reboot_required

    # this task is to restart the system
    - name: restart system
      command: shutdown -r +1  "Rebooting System After Patching"
      async: 0
      poll: 0
      when: reboot_required.stdout == "reboot_needed"
      register: reboot_started
      ignore_errors: true

    # this task is to wait for 3 minutues for system to come up after the reboot
    - name: pause for 180 secs
      pause:
        minutes: 3

    # this task is to confirm,system is up and responding to ssh
    - name: check if system responding to ssh
      local_action:
        module: wait_for
          host={{ inventory_hostname }}
          port=22
          delay=15
          timeout=300
          state=started
      when: reboot_started|changed
'''

[root@ansible-master ansible]#
