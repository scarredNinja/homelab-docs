#HomeLabRebuild/NFS 

sudo mkdir -p /mnt/tvshows /mnt/animation /mnt/movies


sudo mount -t nfs 10.0.60.80:/volume1/tvshows /mnt/tvshows
sudo mount -t nfs 10.0.60.80:/volume1/animation /mnt/animation
sudo mount -t nfs 10.0.60.80:/volume1/movies /mnt/movies


echo "ssh-ed25519 AAAA...rest_of_your_key" | sudo tee -a /root/.ssh/authorized_keys



10.0.60.80:/volume1/movies    /mnt/media/movies    nfs    defaults,nofail,x-systemd.automount  0  0
10.0.60.80:/volume1/tvshows   /mnt/media/tvshows   nfs    defaults,nofail,x-systemd.automount  0  0
10.0.60.80:/volume1/animation /mnt/media/animation nfs    defaults,nofail,x-systemd.automount  0  0
