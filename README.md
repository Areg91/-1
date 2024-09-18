Домашнее задание к занятию «Отказоустойчивость в облаке»
Цель задания
В результате выполнения этого задания вы научитесь:

Конфигурировать отказоустойчивый кластер в облаке с использованием различных функций отказоустойчивости.
Устанавливать сервисы из конфигурации инфраструктуры.
Чеклист готовности к домашнему заданию
Создан аккаунт на YandexCloud.
Создан новый OAuth-токен.
Установлено программное обеспечение  Terraform.
Инструкция по выполнению домашнего задания
Сделайте fork репозитория c Шаблоном решения к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/gitlab-hw или https://github.com/имя-вашего-репозитория/8-03-hw).
Выполните клонирование данного репозитория к себе на ПК с помощью команды git clone.
Выполните домашнее задание и заполните у себя локально этот файл README.md:
впишите вверху название занятия и вашу фамилию и имя
в каждом задании добавьте решение в требуемом виде (текст/код/скриншоты/ссылка)
для корректного добавления скриншотов воспользуйтесь инструкцией "Как вставить скриншот в шаблон с решением"
при оформлении используйте возможности языка разметки md (коротко об этом можно посмотреть в инструкции по MarkDown)
После завершения работы над домашним заданием сделайте коммит (git commit -m "comment") и отправьте его на Github (git push origin);
Для проверки домашнего задания преподавателем в личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
Любые вопросы по выполнению заданий спрашивайте в чате учебной группы и/или в разделе “Вопросы по заданию” в личном кабинете.
Инструменты и дополнительные материалы, которые пригодятся для выполнения задания
Документация сетевого балансировщика нагрузки
Задание 1
Возьмите за основу решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке».

Теперь вместо одной виртуальной машины сделайте terraform playbook, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Рекомендуем изучить документацию сетевого балансировщика нагрузки для того, чтобы было понятно, что вы сделали.

Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

Перейдите в веб-консоль Yandex Cloud и убедитесь, что:

созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.
Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.
В качестве результата пришлите:

1. Terraform Playbook.

2. Скриншот статуса балансировщика и целевой группы.

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.

Решение 1
1 mkdir projects
2 cd projects
3 cd /path/to/project

4 nano main.yml
      
---
- hosts: localhost
  connection: local
  gather_facts: false
  
  tasks:
    - name: Deploy two identical virtual machines with Terraform
      terraform:
        project_path: ./terraform
        state: present
        vars:
          count: 2
          instance_name: myvm{{ item }}
          image_id: centos-8-v20230314
          zone: ru-central1-a
          resources_prefix: vm
          tags:
            Name: VM_{{ item }}
          network_interfaces:
            - subnet_id: {{ subnet_id }}
              security_groups:
                - default
              assign_public_ip: true

      loop: "{{ range(0, count|int - 1)|list }}"
    
    - name: Wait for instances to become ready
      wait_for:
        host: "{{ ansible_ssh_host }}"
        port: 22
        timeout: 300
        search_regex: OpenSSH
        sleep: 5
        delay: 5
      
    - name: Install nginx on the instances
      shell: |
        yum install -y epel-release
        yum install -y nginx
        systemctl enable nginx
        systemctl start nginx
      args:
        executable: /bin/bash

    - name: Create target group
      community.yandex.cloud.ymc_target_group:
        name: nginx-tg
        zone: ru-central1-a
        loadbalancer_id: lb-nginx
        service_account_id: "{{ service_account_id }}"
        state: present

    - name: Add instances to target group
      community.yandex.cloud.ymc_member:
        member_type: INSTANCE
        target_group_id: nginx-tg
        members: "{{ groups['all'] }}"
        service_account_id: "{{ service_account_id }}"
        state: present

    - name: Create a network load balancer
      community.yandex.cloud.ymc_network_load_balancer:
        name: nginx-lb
        target_group_ids:
          - nginx-tg
        listeners:
          - protocol: HTTP
            port: 80
            target_ports: 80
            healthcheck:
              type: HTTP
              path: /
              port: 80
        zones:
          - ru-central1-a
        state: present
        service_account_id: "{{ service_account_id }}"

    - name: Get external IP address of the load balancer
      community.yandex.cloud.ymc_load_balancer:
        id: nginx-lb
        state: info
      register: lb_info

    - name: Check that both virtual machines are healthy in the target group
      community.yandex.cloud.ymc_health_check:
        target_group_id: nginx-tg
        expected_status: HEALTHY
        state: present
      register: check_result
      until: check_result is succeeded
      retries: 10
      delay: 10

    - name: Test connectivity by sending request to the external IP address
      uri:
        url: "http://{{ lb_info.data.external_ips[0] }}/"
        method: GET
      register: response
      failed_when: response.status != 200

    - debug:
        var: response.json

    - debug:
        msg: Successfully deployed infrastructure!
Этот плейбук создает две виртуальные машины, таргет-группу и сетевой балансировщик нагрузки, настраивает Nginx на виртуальных машинах и проверяет работоспособность всей инфраструктуры.
![1](https://github.com/user-attachments/assets/27123798-8aaa-4c32-aafd-1dbdb6d72c16)
![2](https://github.com/user-attachments/assets/84af34da-5730-4d22-bd49-f2379348966c)
