Руководство по установке и настройке OpenStack
========================================================================
.. _contents:

.. toctree::
   :maxdepth: 2

`OpenStack <http://www.openstack.org/>`_ — это комплекс проектов свободного (`Apache License, Version 2.0 <http://www.apache.org/licenses/LICENSE-2.0>`_) программного обеспечения, предназначенного для создания вычислительных облаков и облачных хранилищ.

На данный момент OpenStack содержит семь ключевых компонентов:

	* :ref:`Сервис авторизации и аутентификации «Keystone» <keystone>`
	* :ref:`Хранилище образов «Glance» <glance>`
	* :ref:`Вычислительный сервис «Nova» <nova>`
	* :ref:`Сетевой сервис Quantum <quantum>`
	* :ref:`Сервис хранения блоков данных «Cinder» <cinder>`
	* :ref:`Хранилище объектов «Swift» <swift>`
	* :ref:`Веб-интерфейс «Horizon» <horizon>`

В данное руководство включена так же установка
	
	* :ref:`Распределенной файловой системы CEPH <link>`
	* Cистемы мониторинга Munin
	* Сервера доменных имен MyDNS

Подготовка к установке
------------------------------------------------------------------------

Структура облака
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. figure:: first.png
   :align: center


Подготовка узлов
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   	
Установка сервисов OpenStack будет производиться из пакетов, на основе операционной системы Ubuntu 12.04 «Precise», c использованием системы автоматизации процесса установки и настройки программного обеспечения Chef.

В данном руководстве рассмаривается структура облака, когда все облачные сервисы, кроме вычислительного, расположены на одном узле, называемом "контроллером облака", к которому по локальной сети подключаются вычислительные узлы. 

Вычислительный сервис nova-compute использует гипервизор `KVM <http://www.linux-kvm.org/page/Main_Page>`_

Публичным и приватным адресам вируальных машин с использованием `MyDns <http://mydns.bboy.net/>`_ автоматически выдаются доменные имена. 

Хранилища образов, дисков виртуальных машин и объектов располагаются на LVM томах.

Наиболее продвинутый вариант реализации сетевой инфраструктуры, в котором каждый(!) тенант получает приватный роутер, с возможностью создания дополнительных роутеров для каждого тенанта через Quantum API. Тенант может создавать свои сети, с возможностью подключения к роутеру. Теперь самое главное: данная схема позволяет каждому тенанту использовать любые сети, т.к. доступ вовне обеспечивается или через SNAT или Floating IPs. Иными словами, в облаке может быть несколько ВМ с одинаковыми(!) внутренними IP-адресами. Это может пригодиться, например, при переходе с одного облака на другое – запаковал машины, слил образ, настроил требуемую инфраструктуру на другом облаке, назначил IP-адреса, которые у тебя были ранее, развернул образы и все полетело без дополнительных изменений. Тот, кто часто вынужден был переносить сервера из одной подсети в другую, наверняка оценят эту возможность. С другой стороны, как часто вам может потребоваться таскать свою инфраструктуру между разными облаками?

Подготовка сетевых интерфейсов: ::
	

	# Public interface
	auto eth0
	iface eth0 inet static
	address 195.208.117.140
	netmask 255.255.255.224

	# Private interface
	auto br-int
	iface br-int inet static
		bridge_ports eth1
		address 10.10.10.0
		netmask 255.0.0.0


В настоящем руководстве для установки компонентов облачной инфраструктуры OpenStack, мы будем использовать продукт `OpstChef <http://www.opst.com/chef>`_. Это Open-Source инструмент управления инфраструктурой серверов. Мы будем рассматривать упрощенный вариант использования этого инструмента, без настройки Chef-сервера, а ограничимся использованием knife-solo. 

Первым делом на управляющей машине необходимо установить менеджер пакетов для языка программирования Руби - RubyGems. В операционной системе Ubuntu сделать это можно c помощью стандартного пакетного менеджера командой:

``sudo apt-get install -y rubygems``

``sudo gem-install knife-solo``

``knife configure -r .defaults``


Используемые скрипты для chef расчитаны на использование на будующих узлах облачной инфраструктуры опреационной системы Ubuntu версии 12.04, и налагает следующие требования: :: 

	set[:mysql][:password]="mySuperSecret"

	set[:controller][:private_ip]="10.10.10.1"
	set[:controller][:public_ip]="192.168.0.101"

	set[:keystone][:token]="mySuperSecret"
	set[:keystone][:password]="mySuperSecret"
	set[:keystone][:email]="admin@post.domain.ru"

	set[:dns][:zone]="cloud.domain.ru"


Работа с облачной системой OpenStack, помимо Web-интерфейса, может осуществляться с помощью `nova-client <http://pypi.python.org/pypi/python-novaclient>`_ или инструмента `euca2ools <http://open.eucalyptus.com/wiki/Euca2oolsGuide>`_. В данном руководстве мы постараемся приводить команды для обоих этих инструментов. 

Для использования команд nova, на клиентской машине необходимо установить пакет python-novaclient:  

``sudo apt-get install python-novaclient``

и аутентифицироваться с помощью openrc файла для нужного проекта, который можно загрузить через Web-интерфейс, на вкладке settings 

.. note::

	 Добавьте в файл openrc строчку **export OS\_NO\_CACHE=1** это избавит Вас от необходимости каждый раз вводить пароль связки ключей, при использовании последней версии клиента

После загрузки файла выполните команду:

``source openrc.sh``

Для использования команд euca, на клиентской машине необходимо установить пакет euca2ools:

``sudo apt-get install euca2ools``

и аутентифицироваться с помощью сертификатов EC2. Скачайте zip-архив на странице настроек проекта, распакуйте файлы и выполните

``source ec2rc.sh``

Для запуска виртуальных в облачной инфраструктуре OpenStack используются образы дисков операционных систем. В последней версии Glance добавлена также возможность создания виртуальной машины и установки операционной системы из iso-образа. Мы рассмотрим два основных формата образов дисков:

    * AMI (англ. Amazon Machine Image). Образ операционной системы в этом формате состоит из трех частей: AKI (англ. Amazon Kernel Image) , ARI (англ. Amazon Ramdisk Image) AMI
    * QCOW2 - это формат дискового образа программы QEMU. Название является аббревиатурой названия формата Copy-On-Write (копирование при записи).
  
Первым шагом будет создание пустого файла образа диска. Современная операционная система семейства Windows требует для работы большое количество дискового пространства, не менее чем 20 GB.

``kvm-img create -f raw windowsserver.img 20G``

Openstack использует интерфейс Virtio для дисков и сетевых адаптеров при запуске виртуальных машин. Это означает, что операционная система виртуальной машины должна иметь драйверы для Virtio. По умолчанию операционные машины семейства Windows не содержат таких драйверов, их необходимо предоставить операционной системе в процессе установки. Образ дискеты с последней версией необходимых драйверов доступен для скачивания c сайта `проекта fedora <http://alt.fedoraproject.org/pub/alt/virtio-win/latest/images/bin/>`_
Процесс установки Windows можно запустить с помощью команды.

``kvm -m 1024 -cdrom windows.iso -drive file=windowsserver.img, if=virtio,boot=on -fda virtio-win-1.1.16.vfd -boot d -nographic -vnc :0``

Здесь параметром cdrom указывается путь к образу установочного диска операционной системы, например windows.iso. Параметр drive - предварительно созданный образ жесткого диска. Параметром fda задается путь к загруженному образу дискеты с драйверами windows.  
После запуска команды начнется стандартный процесс установки Windows. К Консоли управления можно подключиться с помощью любого обозревателя VNC, например vncviewer:

``sudo apt-get install vncviewer -y``

на порт 5900. 

``vncviewer localhost:5900`` 

Если создание образа производится на удаленной машине, подключиться к VNC-консоли можно с использованием ssh-туннеля, например так:

``vncviewer -via "laboshinl@192.168.0.100 -p 22" localhost:0``

Необходимо следовать инструкциям, появляющимся на экране. В процессе установки в окне выбора жесткий диск, не будет выведено каких-либо устройств. Необходимо нажать на кнопку "Загрузить драйверы", в левом нижнем углу и указать путь к подключенным драйверам после чего продолжить установку 

.. note::
	
	 Для удобства дальнейшего использования образа рекомендуется после завершения установки разрешить удаленное администрирование системы через RDP(RemoteDesktopProtocol)

`Тестовые образы от Cirros <https://launchpad.net/cirros/trunk/0.3.0/>`_

`Официальные образы релизов Ubuntu <http://uec-images.ubuntu.com/>`_

Готовые образы некоторых операционных систем доступны так же на сайте `нашего проекта <http://xenlet.stu.neva.ru>`_

``nova cloudpipe-create $project_ID``

Шаблон конфирурационного файла для openvpn ::

	# Edit the following lines to point to your cert files:
	cert cert.pem
	key pk.pem

	ca cacert.pem

	client
	dev tap
	proto udp

	remote $controller_public_ip $port
	resolv-retry infinite
	nobind

	# Downgrade privileges after initialization (non-Windows only)
	user nobody
	group nogroup
	comp-lzo

	# Set log file verbosity.
	verb 2

	keepalive 10 120
	ping-timer-rem
	persist-tun
	persist-key

.. _link:

CEPH
------------------------------------------------------------------------


`Ceph <http://ceph.com>`_ — свободная распределённая файловая система. Ceph может использоваться на системах, состоящих как из нескольких машин, так и из тысяч узлов. Общий объем хранилища данных может измеряться петабайтами, встроенные механизмы продублированной репликации данных (не зависит от отказа отдельных узлов) обеспечивают чрезвычайно высокую живучесть системы, при добавлении или удалении новых узлов, массив данных автоматически перебалансируется с учетом новшеств.

Установка

``gpg --keyserver keyserver.ubuntu.com --recv 17ED316D``

``gpg --export --armor 17ED316D | apt-key add -``

Необходимо добавить репозиторий "Grizzly" 

``echo "deb http://ppa.launchpad.net/openstack-ubuntu-testing/grizzly-trunk-testing/ubuntu/ precise main" >> /etc/apt/sources.list``

Получение ключа

``gpg --keyserver keyserver.ubuntu.com --recv 3B6F61A6 && gpg --export --armor 3B6F61A6 | apt-key add -``

Обновление списка пакетов

``apt-get update``

``apt-get install mysql-server python-mysqldb -y``

``sed -i 's/127.0.0.1/10.10.10.0/g' /etc/mysql/my.cnf``

.. hint::

	Здесь **10.10.10.0** ip-адрес сетевого интерфейса во внутренней сети	

``service mysql restart``

``apt-get install rabbitmq-server``

.. _keystone: 

Keystone
------------------------------------------------------------------------

Keystone provides a single point of integration for OpenStack policy, catalog, token and authentication.

	* keystone handles API requests as well as providing configurable catalog, policy, token and identity services.
	* Each Keystone function has a pluggable backend which allows different ways to use the particular service. Most support standard backends like LDAP or SQL, as well as Key Value Stores (KVS).

Most people will use this as a point of customization for their current authentication services.

Установка
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``apt-get install keystone``

``mysql -uroot -pMysqlPass -e "CREATE DATABASE keystone;"``

``mysql -uroot -pMysqlPass -e "GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'MysqlPass';"``

.. hint:: 
	
	Здесь и далее **MysqlPass** - пароль, введенный при установке пакета mysql-server

В конфигурационном файле /etc/keystone/keystone.conf необходимо: 

	* раскоментировать строчку и изменить токен в секции [DEFAULT]: ::
	
		admin_token = AdminToken

	* в секции [sql] указать путь к созданной базе данных: ::
	
		connection = mysql://root:MysqlPass@10.10.10.0/keystone

	* секцию [catalog] привести к следующему виду: ::

		# dynamic, sql-based backend (supports API/CLI-based management commands)
		# driver = keystone.catalog.backends.sql.Catalog

		# static, file-based backend (does *NOT* support any management commands)
		driver = keystone.catalog.backends.templated.TemplatedCatalog
		template_file = default_catalog.templates

	* в секции [signing]: ::

		token_format = UUID


Перезапуск сервиса 

``service keystone restart``

Синхронизация с базой данных

``keystone-manage db_sync``

Аутентификация 

``export SERVICE_TOKEN=AdminToken``

``export SERVICE_ENDPOINT="http://10.10.10.0:35357/v2.0"``

Для дальнейшей работы необходимо создать два проекта. Проект, роль, и пользователь **admin**, необходим для функционирования сервисов и администрирования облака.

``keystone tenant-create --name=admin`` ::

	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |                                  |
	|   enabled   |               True               |
	|      id     | 1f155208db0a4c959365a0002b8b507e |
	|     name    |              admin               |
	+-------------+----------------------------------+

``keystone user-create --name=admin --pass=cl0udAdmin --email=cloud@admin.com`` ::

	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|  email   |         cloud@admin.com          |
	| enabled  |               True               |
	|    id    | 1d2a73ea87f249769f6669ee2f812932 |
	|   name   |              admin               |
	| tenantId |                                  |
	+----------+----------------------------------+

``keystone role-create --name=admin`` ::

	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|    id    | 424f7b79893c4266bf5753894a4668d2 |
	|   name   |              admin               |
	+----------+----------------------------------+

``keystone user-role-add --user-id 1d2a73ea87f249769f6669ee2f812932 --role-id 424f7b79893c4266bf5753894a4668d2 --tenant-id 1f155208db0a4c959365a0002b8b507e`` 

Роль **Member** - роль по умолчанию для добавления пользователей облака. Пользователь **tester** и проект **test** необходимы для проверки работы сервисов облачной инфраструктуры после установки.

``keystone tenant-create --name=test`` ::

	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |                                  |
	|   enabled   |               True               |
	|      id     | 37cfbd624d0242b995fa695d8b134bb6 |
	|     name    |               test               |
	+-------------+----------------------------------+

``keystone user-create --name=tester --pass=cl0udAdmin --email=cloud@admin.com`` ::

	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|  email   |         cloud@admin.com          |
	| enabled  |               True               |
	|    id    | cf0828666bfd4a24b12dcd83848ef360 |
	|   name   |              tester              |
	| tenantId |                                  |
	+----------+----------------------------------+

``keystone role-create --name=Member`` ::

	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|    id    | 01242eec84c14106a10759e210c98dee |
	|   name   |              Member              |
	+----------+----------------------------------+

``keystone user-role-add --user-id cf0828666bfd4a24b12dcd83848ef360 --role-id 01242eec84c14106a10759e210c98dee --tenant-id 37cfbd624d0242b995fa695d8b134bb6``

Файл /etc/keystone/default_catalog.templates необходимо привести к следующему виду ::

	# config for TemplatedCatalog, using camelCase because I don't want to do
	# translations for keystone compat
	catalog.RegionOne.identity.publicURL = http://195.208.117.140:$(public_port)s/v2.0
	catalog.RegionOne.identity.adminURL = http://195.208.117.140:$(admin_port)s/v2.0
	catalog.RegionOne.identity.internalURL = http://195.208.117.140:$(public_port)s/v2.0
	catalog.RegionOne.identity.name = Identity Service

	# fake compute service for now to help novaclient tests work
	catalog.RegionOne.compute.publicURL = http://195.208.117.140:$(compute_port)s/v1.1/$(tenant_id)s
	catalog.RegionOne.compute.adminURL = http://195.208.117.140:$(compute_port)s/v1.1/$(tenant_id)s
	catalog.RegionOne.compute.internalURL = http://195.208.117.140:$(compute_port)s/v1.1/$(tenant_id)s
	catalog.RegionOne.compute.name = Compute Service

	catalog.RegionOne.volume.publicURL = http://195.208.117.140:8776/v1/$(tenant_id)s
	catalog.RegionOne.volume.adminURL = http://195.208.117.140:8776/v1/$(tenant_id)s
	catalog.RegionOne.volume.internalURL = http://195.208.117.140:8776/v1/$(tenant_id)s
	catalog.RegionOne.volume.name = Volume Service

	catalog.RegionOne.ec2.publicURL = http://195.208.117.140:8773/services/Cloud
	catalog.RegionOne.ec2.adminURL = http://195.208.117.140:8773/services/Admin
	catalog.RegionOne.ec2.internalURL = http://195.208.117.140:8773/services/Cloud
	catalog.RegionOne.ec2.name = EC2 Service

	catalog.RegionOne.image.publicURL = http://195.208.117.140:9292/v1
	catalog.RegionOne.image.adminURL = http://195.208.117.140:9292/v1
	catalog.RegionOne.image.internalURL = http://195.208.117.140:9292/v1
	catalog.RegionOne.image.name = Image Service

	catalog.RegionOne.network.publicURL = http://195.208.117.140:9696
	catalog.RegionOne.network.adminURL = http://195.208.117.140:9696
	catalog.RegionOne.network.internalURL = http://195.208.117.140:9696
	catalog.RegionOne.network.name = Network Service

	catalog.RegionOne.object_store.publicURL = http://195.208.117.140:8080/v1/AUTH_$(tenant_id)s
	catalog.RegionOne.object_store.adminURL = http://195.208.117.140:8080/
	catalog.RegionOne.object_store.internalURL = http://195.208.117.140:8080/v1/AUTH_$(tenant_id)s
	catalog.RegionOne.object_store.name = S3 Service

.. note::
	
	Здесь и далее **195.208.117.140** ip-адрес сетевого интерфейса контроллера облака во внешней сети

Использование
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``keystone user-list`` ::

	+----------------------------------+--------+---------+-----------------+
	|                id                |  name  | enabled |      email      |
	+----------------------------------+--------+---------+-----------------+
	| 1d2a73ea87f249769f6669ee2f812932 | admin  |   True  | cloud@admin.com |
	| cf0828666bfd4a24b12dcd83848ef360 | tester |   True  | cloud@admin.com |
	+----------------------------------+--------+---------+-----------------+

.. _glance: 

Glance
------------------------------------------------------------------------

The Glance architecture has stayed relatively stable since the Cactus release. The biggest architectural change has been the addition of authentication, which was added in the Diablo release. Just as a quick reminder, Glance has four main parts to it:

	* glance-api accepts Image API calls for image discovery, image retrieval and image storage.
	* glance-registry stores, processes and retrieves metadata about images (size, type, etc.).
	* A database to store the image metadata. Like Nova, you can choose your database depending on your preference (but most people use MySQL or SQlite).
	* A storage repository for the actual image files. In the diagram above, Swift is shown as the image repository, but this is configurable. In addition to Swift, Glance supports normal filesystems, RADOS block devices, Amazon S3 and HTTP. Be aware that some of these choices are limited to read-only usage.

There are also a number of periodic process which run on Glance to support caching. The most important of these is the replication services, which ensures consistency and availability through the cluster. Other periodic processes include auditors, updaters and reapers.

As you can see from the diagram in the Conceptual Architecture section, Glance serves a central role to the overall IaaS picture. It accepts API requests for images (or image metadata) from end users or Nova components and can store its disk files in the object storage service, Swift.

Установка
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``apt-get install glance``

``mysql -uroot -pMysqlPass -e "CREATE DATABASE glance;"``

В конфигурационных файлах /etc/glance glance-api.conf и /etc/glance/glance-registry.conf необходимо изменить: ::

	[DEFAULT]
	sql_connection = mysql://root@MysqlPass@10.10.10.0/glance

	[keystone_authtoken]
	auth_host = 127.0.0.1
	auth_port = 35357
	auth_protocol = http
	admin_tenant_name = admin
	admin_user = admin
	admin_password = cl0udAdmin

	[paste_deploy]
	flavor = keystone

``service glance-api restart``

``service glance-registry restart``

``glance-manage db_sync``

.. warning::

	Glance требует версию пакета warlock>=0.7.0,<2 а в репозитории Ubuntu 'Precise' версия 0.1.0, необходимо установить свежую версию с помощью pip install

``apt-get install python-pip``

``pip install warlock --upgrade``

Команды Glance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Загрузка тестового образа

``glance image-create --name cirros-0.3.0 --is-public true --container-format bare --disk-format qcow2 --copy-from https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img``

.. note ::
	
	Для тестового образа cirros-0.3.0 помимо ssh-ключа для авторизации можно использовать  логин **cirros** и пароль **cubswin:)**

.. _nova: 

Nova
------------------------------------------------------------------------

Nova is the most complicated and distributed component of OpenStack. A large number of processes cooperate to turn end user API requests into running virtual machines. Below is a list of these processes and their functions:

	* nova-api accepts and responds to end user compute API calls. It supports OpenStack Compute API, Amazon's EC2 API and a special Admin API (for privileged users to perform administrative actions). It also initiates most of the orchestration activities (such as running an instance) as well as enforces some policy (mostly quota checks).
	* The nova-compute process is primarily a worker daemon that creates and terminates virtual machine instances via hypervisor's APIs (XenAPI for XenServer/XCP, libvirt for KVM or QEMU, VMwareAPI for VMware, etc.). The process by which it does so is fairly complex but the basics are simple: accept actions from the queue and then perform a series of system commands (like launching a KVM instance) to carry them out while updating state in the database.
	* nova-volume manages the creation, attaching and detaching of persistent volumes to compute instances (similar functionality to Amazon’s Elastic Block Storage). It can use volumes from a variety of providers such as iSCSI or Rados Block Device in Ceph. A new OpenStack projects, Cinder, will eventually replace nova-volume functionality. In the Folsom release, nova-volume and the Block Storage service will have similar functionality.
	* The nova-network worker daemon is very similar to nova-compute and nova-volume. It accepts networking tasks from the queue and then performs tasks to manipulate the network (such as setting up bridging interfaces or changing iptables rules). This functionality is being migrated to Quantum, a separate OpenStack service. In the Folsom release, much of the functionality will be duplicated between nova-network and Quantum.
	* The nova-schedule process is conceptually the simplest piece of code in OpenStack Nova: take a virtual machine instance request from the queue and determines where it should run (specifically, which compute server host it should run on).
	* The queue provides a central hub for passing messages between daemons. This is usually implemented with RabbitMQ today, but could be any AMPQ message queue (such as Apache Qpid). New to the Folsom release is support for Zero MQ (note: I've only included this so that Eric Windisch won't be hounding me mercilessly about it's omission).
	* The SQL database stores most of the build-time and run-time state for a cloud infrastructure. This includes the instance types that are available for use, instances in use, networks available and projects. Theoretically, OpenStack Nova can support any database supported by SQL-Alchemy but the only databases currently being widely used are sqlite3 (only appropriate for test and development work), MySQL and PostgreSQL.
	* Nova also provides console services to allow end users to access their virtual instance's console through a proxy. This involves several daemons (nova-console, nova-vncproxy and nova-consoleauth).

Nova interacts with many other OpenStack services: Keystone for authentication, Glance for images and Horizon for web interface. The Glance interactions are central. The API process can upload and query Glance while nova-compute will download images for use in launching images.

Установка
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler``	

``mysql -uroot -pMysqlPass -e "CREATE DATABASE nova;"``

В файле /etc/nova/api-paste.ini: ::

	[filter:authtoken]
	paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
	auth_host = 127.0.0.1
	auth_port = 35357
	auth_protocol = http
	admin_tenant_name = admin
	admin_user = admin
	admin_password = cl0udAdmin
	signing_dir = /tmp/keystone-signing-nova

В файле /etc/nova/nova.conf: ::

	[DEFAULT]
	logdir=/var/log/nova
	state_path=/var/lib/nova
	lock_path=/run/lock/nova
	verbose=True
	api_paste_config=/etc/nova/api-paste.ini
	compute_scheduler_driver = nova.scheduler.filter_scheduler.FilterScheduler
	s3_host=10.10.10.0
	ec2_host=10.10.10.0
	ec2_dmz_host=10.10.10.0
	rabbit_host=10.10.10.0
	cc_host=10.10.10.0
	dmz_cidr=169.254.169.254/32
	metadata_host=10.10.10.0
	metadata_listen=0.0.0.0
	nova_url=http://10.10.10.0:8774/v1.1/
	sql_connection=mysql://root:MysqlPass@10.10.10.0/nova
	ec2_url=http://10.10.10.0:8773/services/Cloud
	root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

	# Auth
	use_deprecated_auth=false
	auth_strategy=keystone
	keystone_ec2_url=http://10.10.10.0:5000/v2.0/ec2tokens
	# Imaging service
	glance_api_servers=10.10.10.0:9292
	image_service=nova.image.glance.GlanceImageService

	# Vnc configuration
	novnc_enabled=true
	novncproxy_base_url=http://195.208.117.140:6080/vnc_auto.html
	novncproxy_port=6080
	vncserver_proxyclient_address=195.208.117.140
	vncserver_listen=0.0.0.0

	# Network settings
	network_api_class=nova.network.quantumv2.api.API
	quantum_url=http://10.10.10.0:9696
	quantum_auth_strategy=keystone
	quantum_admin_tenant_name=service
	quantum_admin_username=quantum
	quantum_admin_password=service_pass
	quantum_admin_auth_url=http://10.10.10.0:35357/v2.0
	libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
	linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
	firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

	# Compute #
	compute_driver=libvirt.LibvirtDriver

	# Cinder #
	volume_api_class=nova.volume.cinder.API
	osapi_volume_listen_port=5900

Синхронизация с базой

``nova-manage db_sync``



``apt-get install -y kvm libvirt-bin pm-utils nova-conductor``


Перезапуск сервисов

``find /etc/init.d -name nova* -exec {} restart \;``

.. hint :: 

	Посмотреть список работающих сервисов Nova можно командой nova-manage service list

.. _cinder: 

Cinder
------------------------------------------------------------------------

Cinder separates out the persistent block storage functionality that was previously part of Openstack Compute (in the form of nova-volume) into it's own service. The OpenStack Block Storage API allows for manipulation of volumes, volume types (similar to compute flavors) and volume snapshots.

	* cinder-api accepts API requests and routes them to cinder-volume for action.
	* cinder-volume acts upon the requests by reading or writing to the Cinder database to maintain state, interacting with other processes (like cinder-scheduler) through a message queue and directly upon block storage providing hardware or software. It can interact with a variety of storage providers through a driver architecture. Currently, there are drivers for IBM, SolidFire, NetApp, Nexenta, Zadara, linux iSCSI and other storage providers.
	* Much like nova-scheduler, the cinder-scheduler daemon picks the optimal block storage provider node to create the volume on.
	* Cinder deployments will also make use of a messaging queue to route information between the cinder processes as well as a database to store volume state.

Like Quantum, Cinder will mainly interact with Nova, providing volumes for its instances.

Установка
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms``

``sed -i 's/false/true/g' /etc/default/iscsitarget``

``service iscsitarget start``

``service open-iscsi start``

``mysql -uroot -pMysqlPass -e "CREATE DATABASE cinder;"``

В файле /etc/cinder/api-pate.ini: ::

	[filter:authtoken]
	paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
	service_protocol = http
	service_host = 127.0.0.1
	service_port = 5000
	auth_host = 127.0.0.1
	auth_port = 35357
	auth_protocol = http
	admin_tenant_name = admin
	admin_user = admin
	admin_password = cl0udAdmin
	signing_dir = /var/lib/cinder

В файле /etc/cinder/cinder.conf: ::
	
	[DEFAULT]
	rootwrap_config = /etc/cinder/rootwrap.conf
	sql_connection = mysql://root:MysqlPass@10.10.10.0/cinder
	api_paste_confg = /etc/cinder/api-paste.ini
	iscsi_helper = tgtadm
	volume_name_template = volume-%s
	volume_group = tn0
	verbose = True
	auth_strategy = keystone
	state_path = /var/lib/cinder
	volumes_dir = /var/lib/cinder/volumes
	
.. hint ::
	
	Здесь tn0 - название группы логических томов lvm2

``cinder-manage db sync``

``service cinder-volume restart``

``service cinder-api restart``

.. _horizon: 

Dashboard
------------------------------------------------------------------------

Horizon is a modular Django web application that provides an end user and administrator interface to OpenStack services.

.. figure:: horizon.png 

As with most web applications, the architecture is fairly simple:

	* Horizon is usually deployed via mod_wsgi in Apache. The code itself is separated into a reusable python module with most of the logic (interactions with various OpenStack APIs) and presentation (to make it easily customizable for different sites).
	* A database (configurable as to which one). As it relies mostly on the other services for data, it stores very little data of its own.

From a network architecture point of view, this service will need to be customer accessible as well as be able to talk to each service's public APIs. If you wish to use the administrator functionality (i.e. for other services), it will also need connectivity to their Admin API endpoints (which should not be customer accessible).

Установка
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``apt-get install openstack-dashboard memcached node-less``

.. _quantum: 

Quantum
------------------------------------------------------------------------

.. figure:: quantum.png
   :align: center

За работу с сетевой частью OpenStack отвечает библиотека Quantum, которая обеспечивает функцию «сеть как сервис» между сетевыми интерфейсами ВМ (vNIC) под управлением других сервисов OpenStack, фактически предоставляя API, позволяющее управлять всей сетевой частью облака. В зависимости от поставленных задач и спроектированной целевой конфигурации облака, к Quantum можно подключать плагины, такие как Open vSwith, Cisco UCS/Nexus, Linux Brige, NEC OpenFlow, Nicira Network Virtualization Platform (NVP) и некоторые другие. 
Quantum provides "network connectivity as a service" between interface devices managed by other OpenStack services (most likely Nova). The service works by allowing users to create their own networks and then attach interfaces to them. Like many of the OpenStack services, Quantum is highly configurable due to it's plug-in architecture. These plug-ins accommodate different networking equipment and software. As such, the architecture and deployment can vary dramatically. In the above architecture, a simple Linux networking plug-in is shown.

	* quantum-server accepts API requests and then routes them to the appropriate quantum plugin for action.
	* Quantum plugins and agents perform the actual actions such as plugging and unplugging ports, creating networks or subnets and IP addressing. These plugins and agents differ depending on the vendor and technologies used in the particular cloud. Quantum ships with plugins and agents for: Cisco virtual and physical switches, Nicira NVP product, NEC OpenFlow products, Open vSwitch, Linux bridging and the Ryu Network Operating System. Midokua also provides a plug-in for Quantum integration. The common agents are L3 (layer 3), DHCP (dynamic host IP addressing) and the specific plug-in agent.
	* Most Quantum installations will also make use of a messaging queue to route information between the quantum-server and various agents as well as a database to store networking state for particular plugins.

Quantum will interact mainly with Nova, where it will provide networks and connectivity for its instances.

Установка
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 pip install cliff --upgrade

.. _swift:

Swift
------------------------------------------------------------------------

The swift architecture is very distributed to prevent any single point of failure as well as to scale horizontally. It includes the following components:

	* Proxy server (swift-proxy-server) accepts incoming requests via the OpenStack Object API or just raw HTTP. It accepts files to upload, modifications to metadata or container creation. In addition, it will also serve files or container listing to web browsers. The proxy server may utilize an optional cache (usually deployed with memcache) to improve performance.
	* Account servers manage accounts defined with the object storage service.
	* Container servers manage a mapping of containers (i.e folders) within the object store service.
	* Object servers manage actual objects (i.e. files) on the storage nodes.
	* There are also a number of periodic process which run to perform housekeeping tasks on the large data store. The most important of these is the replication services, which ensures consistency and availability through the cluster. Other periodic processes include auditors, updaters and reapers.

The object store can also serve static web pages and objects via HTTP. In fact, the diagrams in this blog post are being served out of Rackspace Cloud's Swift service.

Authentication is handled through configurable WSGI middleware (which will usually be Keystone).


Установка
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
