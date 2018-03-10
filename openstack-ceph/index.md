这里使用Pike版本的Openstack和Luminous版本的ceph，所有节点都是ubuntu16.04系统。

主要参考: http://docs.ceph.com/docs/master/rbd/rbd-openstack/

# 1. openstack的安装部署

略

# 2. ceph的安装部署

这里使用了单节点部署，ceph部署在ceph01节点。当然，生产环境需要三副本。

### 软件包安装

    echo deb https://download.ceph.com/debian-luminous/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list

或者，使用国内镜像源会比较快：

    echo deb http://mirrors.163.com/ceph/debian-luminous/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list

或

    echo deb http://mirrors.ustc.edu.cn/ceph/debian-luminous/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list

安装

    wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
    apt update
    apt install ceph-deploy ceph

### 使用ceph-deploy部署

    mkdir ceph-cluster
    cd ceph-cluster
    
    ceph-deploy new ceph01
    ceph-deploy mon create-initial
    ceph-deploy admin ceph01
    ceph-deploy mgr create ceph01
    ceph-deploy osd create --data /dev/vdc ceph01

我测试中发现直接使用vdc有问题，分区后使用vdc1通过。应该是和我的虚拟机环境有关。

### 创建存储池

PG的个数以及副本的数量需要根据实际情况来。

    ceph osd pool create images 32
    ceph osd pool create vms 32
    ceph osd pool create volumes 32

    ceph osd pool set images size 1
    ceph osd pool set vms size 1
    ceph osd pool set volumes size 1

    rbd pool init images
    rbd pool init vms
    rbd pool init volumes

### 权限创建

创建glance和cinder账号。

    ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images'
    ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd pool=images'

    ceph auth get-or-create client.glance > /etc/ceph/ceph.client.glance.keyring
    ceph auth get-or-create client.cinder > /etc/ceph/ceph.client.cinder.keyring

# 3. openstack节点ceph环境准备

软件包安装，注意至少需要和ceph节点同一个大版本。

    apt-get install python-rbd ceph-common

复制ceph节点的配置文件和key文件

    ssh ceph01 cat /etc/ceph/ceph.conf > /etc/ceph/ceph.conf
    ssh ceph01 cat /etc/ceph/ceph.client.glance.keyring > /etc/ceph/ceph.client.glance.keyring
    ssh ceph01 cat /etc/ceph/ceph.client.cinder.keyring > /etc/ceph/ceph.client.cinder.keyring

如果是生产环境，key文件应该分别设置owener为glance和cinder，并设置对其他帐号不可读。

# 4. libvirt权限加入

在每个计算节点，libvirt需要拥有访问ceph的权限。

其中3bb42560-1cf7-11e8-864b-f328bf7f7cb2通过`uuid`生成，不需要一模一样。

创建临时secret.xml文件：

    <secret ephemeral='no' private='no'>
      <uuid>3bb42560-1cf7-11e8-864b-f328bf7f7cb2</uuid>
      <usage type='ceph'>
        <name>client.cinder secret</name>
      </usage>
    </secret>

执行如下命令，其中KEY需要在ceph节点执行`ceph auth get-key client.cinder`获得

    virsh secret-define --file secret.xml
    virsh secret-set-value --secret 3bb42560-1cf7-11e8-864b-f328bf7f7cb2 --base64 KEY

删除临时文件

    rm -rf secret.xml

# 5. cinder的配置

cinder-volume服务需要配置  /etc/cinder/cinder.conf

    [DEFAULT]
    enabled_backends = ceph
    
    [ceph]
    volume_driver = cinder.volume.drivers.rbd.RBDDriver
    volume_backend_name = CEPH_CLL
    rbd_pool = volumes
    rbd_user = cinder
    rbd_ceph_conf = /etc/ceph/ceph.conf
    rbd_keyring_conf = /etc/ceph/ceph.client.cinder.keyring
    rbd_secret_uuid = 3bb42560-1cf7-11e8-864b-f328bf7f7cb2

重启cinder-volume服务

    service cinder-volume restart

nova-compute服务， 设置/etc/nova/nova.conf

    [libvirt]
    ...
    rbd_user = cinder
    rbd_secret_uuid = 3bb42560-1cf7-11e8-864b-f328bf7f7cb2

重启nova-compute服务

    service nova-compute restart

配置cinder的volume-type

    cinder type-create ceph_cll
    cinder type-key ceph_cll set volume_backend_name=CEPH_CLL

# 6. glance和nova配置

glance-api服务,设置/etc/glance/glance-api.conf

    [glance_store]
    stores = rbd
    default_store = rbd
    rbd_store_pool = images
    rbd_store_user = glance
    rbd_store_ceph_conf = /etc/ceph/ceph.conf
    rbd_store_chunk_size = 8
    
    [DEFAULT]
    show_image_direct_url = True 

重启glance-api服务

    service glance-api restart

nova-compute服务， 设置/etc/nova/nova.conf

    [libvirt]
    images_type = rbd
    images_rbd_pool = vms
    images_rbd_ceph_conf = /etc/ceph/ceph.conf
    rbd_user = cinder
    rbd_secret_uuid = 3bb42560-1cf7-11e8-864b-f328bf7f7cb2
    disk_cachemodes="network=writeback"

重启nova服务

    service nova-compute restart

ceph只支持RAW格式的镜像，转换方式：

    qemu-img convert -f qcow2 -O raw cirros-0.3.5-x86_64-disk.img cirros-0.3.5-x86_64-disk.raw

镜像上传

    openstack image create "cirros" \
      --file cirros-0.3.5-x86_64-disk.raw \
      --disk-format raw --container-format bare \
      --public


