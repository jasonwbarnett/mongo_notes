    gcloud compute disks create NAME [NAME ...] [--description DESCRIPTION]
        [--image IMAGE | --source-snapshot SOURCE_SNAPSHOT]
        [--image-project IMAGE_PROJECT] [--size SIZE] [--type TYPE]
        [--zone ZONE] [GLOBAL-FLAG ...]

    gcloud compute instances create NAME [NAME ...]
        [--boot-disk-device-name BOOT_DISK_DEVICE_NAME]
        [--boot-disk-size BOOT_DISK_SIZE] [--boot-disk-type BOOT_DISK_TYPE]
        [--can-ip-forward] [--description DESCRIPTION]
        [--disk [PROPERTY=VALUE,...]]
        [--image IMAGE | centos-6 | centos-7 | container-vm | coreos | debian-7
          | debian-7-backports | opensuse-13 | rhel-6 | rhel-7 | sles-11
          | sles-12 | ubuntu-12-04 | ubuntu-14-04 | ubuntu-14-10 | ubuntu-15-04
          | windows-2008-r2 | windows-2012-r2] [--image-project IMAGE_PROJECT]
        [--local-ssd [PROPERTY=VALUE,...]]
        [--machine-type MACHINE_TYPE; default="n1-standard-1"]
        [--maintenance-policy MAINTENANCE_POLICY]
        [--metadata KEY=VALUE,[KEY=VALUE,...]]
        [--metadata-from-file KEY=LOCAL_FILE_PATH,[KEY=LOCAL_FILE_PATH,...]]
        [--network NETWORK; default="default"]
        [--address ADDRESS | --no-address] [--no-boot-disk-auto-delete]
        [--no-restart-on-failure] [--preemptible]
        [--no-scopes | --scopes [ACCOUNT=]SCOPE,[[ACCOUNT=]SCOPE,...]]
        [--tags TAG,[TAG,...]] [--zone ZONE] [GLOBAL-FLAG ...]

    disk types:
      pd-ssd
      pd-standard

    gcloud compute disks create mongodb01-root mongodb02-root mongodb03-root mongodb04-root mongodb05-root \
        --image centos-7 \
        --size 10GB \
        --type pd-standard \
        --zone us-central1-f

    gcloud compute disks create mongodb01-storage mongodb02-storage mongodb03-storage mongodb04-storage \
        --size 2200GB \
        --type pd-standard \
        --zone us-central1-f

    for vm in mongodb01 mongodb02 mongodb03 mongodb04; do
      gcloud compute instances create ${vm} \
          --disk name=${vm}-root,mode=rw,boot=yes,device-name=${vm}-root,auto-delete=no \
          --disk name=${vm}-storage,mode=rw,boot=no,device-name=${vm}-storage,auto-delete=no \
          --machine-type n1-standard-1 \
          --network default \
          --zone us-central1-f
    done

    gcloud compute ssh mongodb01 --zone us-central1-f

    cat <<'EOF' | sudo tee -a /etc/yum.repos.d/mongodb-org-2.6.repo
    [mongodb-org-2.6]
    name=MongoDB 2.6 Repository
    baseurl=http://downloads-distro.mongodb.org/repo/redhat/os/x86_64/
    gpgcheck=0
    enabled=1
    EOF

    sudo yum install --exclude="mongodb-org mongodb-org-server" mongo-10gen-server-2.2.4-mongodb_1.x86_64 mongo-10gen-2.2.4-mongodb_1.x86_64

    sudo mkdir /data
    sudo /usr/share/google/safe_format_and_mount \
      -o defaults,auto,noatime,noexec /dev/sdb /data
    echo '/dev/sdb /data xfs defaults,auto,noatime,noexec 0 0' | sudo tee -a /etc/fstab

    sudo chown mongod:mongod /data

    cat <<'EOF' | sudo tee -a /etc/security/limits.conf
    mongod soft nofile 64000
    mongod hard nofile 64000
    mongod soft nproc 32000
    mongod hard nproc 32000
    EOF

    cat <<'EOF' | sudo tee -a /etc/security/limits.d/90-nproc.conf
    mongod soft nproc 32000
    mongod hard nproc 32000
    EOF

    sudo blockdev --setra 32 /dev/sdb

    echo 'ACTION=="add", KERNEL=="sdb", ATTR{bdi/read_ahead_kb}="16"' \
      | sudo tee -a /etc/udev/rules.d/85-mongod.rules

    echo 300 | sudo tee /proc/sys/net/ipv4/tcp_keepalive_time
    echo "net.ipv4.tcp_keepalive_time = 300" | sudo tee -a /etc/sysctl.conf

    sudo service mongod start
    sudo chkconfig mongod on
