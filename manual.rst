Ручная установка 
========================================================================

Установка контроллера облака
------------------------------------------------------------------------

.. _filesystem:

CEPH
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


`Ceph <http://ceph.com>`_ — свободная распределённая файловая система. Ceph может использоваться на системах, состоящих как из нескольких машин, так и из тысяч узлов. Общий объем хранилища данных может измеряться петабайтами, встроенные механизмы продублированной репликации данных (не зависит от отказа отдельных узлов) обеспечивают чрезвычайно высокую живучесть системы, при добавлении или удалении новых узлов, массив данных автоматически перебалансируется с учетом новшеств.

Установка

``# gpg --keyserver keyserver.ubuntu.com --recv 17ED316D``

``# gpg --export --armor 17ED316D | apt-key add -``

``# apt-get update`` 

``# sudo apt-get install ceph``





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


``keystone user-list`` ::

	+----------------------------------+--------+---------+-----------------+
	|                id                |  name  | enabled |      email      |
	+----------------------------------+--------+---------+-----------------+
	| 1d2a73ea87f249769f6669ee2f812932 | admin  |   True  | cloud@admin.com |
	| cf0828666bfd4a24b12dcd83848ef360 | tester |   True  | cloud@admin.com |
	+----------------------------------+--------+---------+-----------------+

.. _glance: 

Glance
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


Загрузка тестового образа

``glance image-create --name cirros-0.3.0 --is-public true --container-format bare --disk-format qcow2 --copy-from https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img``

.. note ::
	
	Для тестового образа cirros-0.3.0 помимо ssh-ключа для авторизации можно использовать  логин **cirros** и пароль **cubswin:)**

.. _nova: 

Nova
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
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``apt-get install openstack-dashboard memcached node-less``

.. _quantum: 

Quantum
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 pip install cliff --upgrade

.. _swift:

Swift
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


.. _munin:

Munin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. _names:

MyDNS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Добавление вычислительных узлов
------------------------------------------------------------------------

Quantum
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Nova
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Swift
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Ceph
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
